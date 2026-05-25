-- 1. Baza 11a (Siedziba) - Tworzenie dzienników (LOG) aby FAST było możliwe
CREATE MATERIALIZED VIEW LOG ON kursanci WITH PRIMARY KEY;
CREATE MATERIALIZED VIEW LOG ON kursanci WITH ROWID, PRIMARY KEY INCLUDING NEW VALUES; -- Log dla lokalnego on commit

-- 2. Baza 11b (Filia) - Tworzenie migawki FAST wskazującej na siedzibę
-- Uwaga: W bazie 11b najpierw musisz mieć dblink do 11a!
-- CREATE DATABASE LINK dblinkSiedziba CONNECT TO RBDN2_ST13 IDENTIFIED BY start123 USING 'baza11a';
CREATE MATERIALIZED VIEW mv_kursanci_z_siedziby
REFRESH FAST 
AS SELECT * FROM kursanci@dblinkSiedziba;

-- 3. Baza 11a (Siedziba) - Migawka dla tabeli lokalnej ON COMMIT
CREATE MATERIALIZED VIEW mv_kursanci_lok
REFRESH FAST ON COMMIT 
AS SELECT * FROM kursanci;

-- 4. Migawka z wyliczonym łącznym przychodem i 19% podatkiem
CREATE MATERIALIZED VIEW mv_przychod_podatek
BUILD IMMEDIATE
REFRESH COMPLETE ON DEMAND
AS 
SELECT nazwa_kursu, 
       SUM(cena * ilosc_uczestnikow) AS przychod,
       SUM(cena * ilosc_uczestnikow) * 0.19 AS podatek
FROM kursyAll
GROUP BY nazwa_kursu;

-- 5. Migawka REP_wykladowcy (z filii)
CREATE MATERIALIZED VIEW REP_wykladowcy
REFRESH COMPLETE ON DEMAND
AS SELECT * FROM wykladowcyFilia;

-- Ręczne odświeżenie
-- EXECUTE DBMS_MVIEW.REFRESH('REP_wykladowcy', 'C');

-- 6. Migawka opóźniona - godziny wykładowców z filii
CREATE MATERIALIZED VIEW REP_godz_wykladowcy_godziny
BUILD DEFERRED
REFRESH COMPLETE 
START WITH LAST_DAY(SYSDATE) 
NEXT SYSDATE + 1/24
AS 
SELECT w.imie, w.nazwisko, SUM(r.godz) AS laczna_liczba_godzin
FROM wykladowcyFilia w
JOIN kursyFilia k ON w.wykladowca_id = k.wykladowca_id
JOIN rodzajeFilia r ON k.rodzaj_id = r.rodzaj_id
GROUP BY w.imie, w.nazwisko;

-- 7. Migawka REP_kursy
CREATE MATERIALIZED VIEW REP_kursy
BUILD IMMEDIATE
REFRESH COMPLETE 
START WITH SYSDATE 
NEXT SYSDATE + 7
AS 
SELECT r.nazwa AS nazwa_kursu, w.imie, w.nazwisko, r.godz AS ilosc_godzin, r.cena AS oplata
FROM kursyFilia k
JOIN rodzajeFilia r ON k.rodzaj_id = r.rodzaj_id
JOIN wykladowcyFilia w ON w.wykladowca_id = k.wykladowca_id;

-- 8. Zwykła perspektywa łącząca REP_kursy i bazę lokalną
CREATE VIEW kursy_siedziba_filia_view AS
SELECT nazwa_kursu, imie, nazwisko, ilosc_godzin, oplata FROM REP_kursy
UNION
SELECT r.nazwa, w.imie, w.nazwisko, r.godz, r.cena 
FROM kursySiedziba k
JOIN rodzajeSiedziba r ON k.rodzaj_id = r.rodzaj_id
JOIN wykladowcySiedziba w ON w.wykladowca_id = k.wykladowca_id;

-- 9. Wyświetlenie informacji o migawkach
SELECT mview_name, refresh_mode, refresh_method, build_mode FROM USER_MVIEWS;



