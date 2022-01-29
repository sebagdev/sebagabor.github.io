---
id: 50
title: 'Nie taki znowu zwykły SQL Server cz. 2 – CTE w modelach hierarchicznych cd.'
date: '2017-03-19T18:00:51+01:00'
author: sg
layout: post
guid: 'http://sgdev.pl/?p=50'
permalink: /2017/03/19/nie-taki-znowu-zwykly-sql-server-cz-2-cte-w-rekurencji/
categories:
    - 'Nie taki znów zwykły SQL'
tags:
    - cte
    - sql
    - 'sql server'
    - with
---

Nadszedł czas na kontynuację naszej przygody z CTE. W poprzednim odcinku pokazałem jak bardzo wspólne wyrażenia tablicowe zbliżone są do podzapytań oraz jak można z nich korzystać. Wspomniałem też na samym początku, że pozwalają na obsługę modeli hierarchicznych z poziomu języka SQL, zatem postaram się nieco przybliżyć to zagadnienie.

#### Czym w ogóle jest model hierarchiczny ?

W dużym uproszczeniu mamy z nim do czynienia wtedy kiedy modelujemy jakąś hierarchię (ach to masło maślane). Najprościej jest zwizualizować sobie drzewiastą strukturę:

[![Hierarchical Model](https://upload.wikimedia.org/wikipedia/commons/thumb/e/eb/Hierarchical_Model.svg/512px-Hierarchical_Model.svg.png)](https://commons.wikimedia.org/wiki/File%3AHierarchical_Model.svg "By U.S. Department of Transportation vectorization: Own work [Public domain], via Wikimedia Commons")

W podobny sposób układają się np. katalogi i pliki w systemie operacyjnym, modelować w ten sposób możemy np. pracowników i przełożonych – każdy pracownik ma swojego przełożonego, aż po prezesa. Istnieją i istniały też dedykowane bazy hierarchiczne, jeszcze przed czasami dominacji modelu relacyjnego. Jedną z pozostałości po takich bazach jest np. rejestr Windows. Co ciekawe, jeśli się przyjrzeć współczesnym formatom, takim jak XML także ta hierarchiczność tam występuje, ale… nie miało być o tym 🙂

Jak wspominałem na rynku dominuje model relacyjny i chcąc odzwierciedlić np. zależność między pracownikami musimy odwołać się do tabeli przechowującej pracownika:

```
USE ShowCaseDb;
GO

-- Stwórzmy tabelkę reprezentującą kto jest czyim podwładnym w świecie Śródziemia
CREATE TABLE BEINGS (
    ID INT NOT NULL PRIMARY KEY, 
    NAME NVARCHAR(40) NOT NULL,
    EMPLOYER_ID INT NULL -- to nasz pracodawca
)
GO
```

W takim przypadku naszym kluczem obcym będzie:

```
ALTER TABLE BEINGS ADD FOREIGN KEY (EMPLOYER_ID) REFERENCES BEINGS;
GO
```

Tak modelując takie rozwiązanie najsensowniej jest stworzyć relację w ramach tej samej tabeli.  
Możemy załadować kilka przykładowych wierszy:

```
INSERT INTO BEINGS(ID, NAME, EMPLOYER_ID) VALUES (1, 'Iluvatar', NULL);
INSERT INTO BEINGS(ID, NAME, EMPLOYER_ID) VALUES (2, 'Morgoth', 1);
INSERT INTO BEINGS(ID, NAME, EMPLOYER_ID) VALUES (3, 'Manwe', 1);
INSERT INTO BEINGS(ID, NAME, EMPLOYER_ID) VALUES (7, 'Sauron', 2);
INSERT INTO BEINGS(ID, NAME, EMPLOYER_ID) VALUES (8, 'Witchking of Angmar', 7);
INSERT INTO BEINGS(ID, NAME, EMPLOYER_ID) VALUES (9, 'Gandalf', 3);
INSERT INTO BEINGS(ID, NAME, EMPLOYER_ID) VALUES (10, 'Saruman', 3);
INSERT INTO BEINGS(ID, NAME, EMPLOYER_ID) VALUES (11, 'Radagast', 3);
INSERT INTO BEINGS(ID, NAME, EMPLOYER_ID) VALUES (12, 'Frodo', 9);
INSERT INTO BEINGS(ID, NAME, EMPLOYER_ID) VALUES (13, 'Sam', 12);
INSERT INTO BEINGS(ID, NAME, EMPLOYER_ID) VALUES (14, 'Aragorn', 9);
```

#### Konsekwencje takiego modelowania

Teraz chcąc np. wybrać wszystkie istoty podległe Iluvatarowi, możemy wywołać zapytanie:

```
-- Chcąc wydobyć wszystkich podwładnych danej istoty musimy skorzystać z EMPLOYER_ID
SELECT * FROM BEINGS where EMPLOYER_ID = (SELECT TOP 1 ID FROM BEINGS WHERE NAME = 'Iluvatar' )
```

[![](https://sgdev.pl/wp-content/uploads/2017/03/scrin01-300x75.png)](https://sgdev.pl/wp-content/uploads/2017/03/scrin01.png)

Chyba nie do końca o to nam chodziło… Interesują nas nie tylko bezpośredni podwładni, ale wszyscy. Możemy próbować rozbudowywać to zapytanie o kolejne poziomy, ale wydaje się to drogą ku zatraceniu.

```
SELECT * FROM BEINGS where EMPLOYER_ID = (SELECT TOP 1 ID FROM BEINGS WHERE NAME = 'Iluvatar' )
UNION ALL
SELECT * FROM BEINGS where EMPLOYER_ID IN (SELECT ID FROM BEINGS where EMPLOYER_ID = (SELECT TOP 1 ID FROM BEINGS WHERE NAME = 'Iluvatar' ) )
-- UNION ALL 
-- 
```

[![](https://sgdev.pl/wp-content/uploads/2017/03/Zaznaczenie_008-300x167.png)](https://sgdev.pl/wp-content/uploads/2017/03/Zaznaczenie_008.png)

Tutaj każda z baz proponuje swoje rozwiązanie (np. CONNECT BY autorstwa Oracle – swoją drogą całkiem przyjemna składnia), ale znam tylko jedne zgodne ze standardem języka SQL – właśnie użycie CTE. Ok, więc jak do tego podejść? Nie będę zdradzał rozwiązania od razu spróbujmy do tego dotrzeć do niego krok po kroku w naturalny sposób, a wyjdziemy zaczynając od błędnego kodu. Po tym krótkim spacerku mam nadzieję, że już nigdy nie będzie to dla was trudne.

```
WITH BEINGS_OF_ARDA_TREE AS (
    SELECT 1 as PLACE, * FROM BEINGS WHERE EMPLOYER_ID IS NULL 
    UNION ALL
    SELECT 2 as PLACE, * FROM BEINGS where EMPLOYER_ID in (SELECT TOP 1 ID FROM BEINGS WHERE EMPLOYER_ID IS NULL )
    UNION ALL
    SELECT 3 as PLACE, * FROM BEINGS where EMPLOYER_ID in (SELECT ID FROM BEINGS where EMPLOYER_ID in (SELECT TOP 1 ID FROM BEINGS WHERE EMPLOYER_ID IS NULL ))
    -- ... i tak dalej
)
SELECT * FROM BEINGS_OF_ARDA_TREE
```

[![](https://sgdev.pl/wp-content/uploads/2017/03/Zaznaczenie_009-300x143.png)](https://sgdev.pl/wp-content/uploads/2017/03/Zaznaczenie_009.png)

Zauważyliście, że dodałem kolumnę PLACE mówiącą na którym poziomie hierarchii jesteśmy? Druga rzecz, która powinna zwrócić waszą uwagę to to, że z każdym kolejnym wierszem dodajemy jakby jeden poziom. Ostatnie najważniejsze spostrzeżenie jakie się nasuwa to, że każde zapytanie poza pierwszym uwzględnia wynik zapytania wcześniejszego. A czy… zamiast składni WHERE… IN… nie możnaby użyć złączenia z już przygotowanym wyrażeniem tablicowym ?

```
WITH BEINGS_OF_ARDA_TREE AS (
    SELECT 1 as PLACE, * FROM BEINGS WHERE EMPLOYER_ID IS NULL 
    UNION ALL
    SELECT ba.PLACE+ 1  as PLACE, b.* FROM BEINGS b INNER JOIN BEINGS_OF_ARDA_TREE ba on ba.ID = b.EMPLOYER_ID --łączymy z samym sobą(!)
)
SELECT * FROM BEINGS_OF_ARDA_TREE
```

[![](https://sgdev.pl/wp-content/uploads/2017/03/Zaznaczenie_010-300x220.png)](https://sgdev.pl/wp-content/uploads/2017/03/Zaznaczenie_010.png)

BINGO! Chciałoby się zakrzyknąć. Czy w takim razie dałoby się zbudować tą hierarchię w druga stronę ? Np. poczynając od Sama przejść, aż po Iluvatara ? Żaden problem:

```
WITH BEINGS_OF_ARDA_TREE AS (
    SELECT 1 as PLACE, * FROM BEINGS WHERE NAME = 'Sam' 
    UNION ALL
    SELECT ba.PLACE+ 1  as PLACE, b.* FROM BEINGS b INNER JOIN BEINGS_OF_ARDA_TREE ba on b.ID = ba.EMPLOYER_ID
)
SELECT * FROM BEINGS_OF_ARDA_TREE

--WITH BEINGS_OF_ARDA_TREE AS (
-- SELECT 1 as PLACE, * FROM BEINGS WHERE NAME = 'Sam' 
-- UNION ALL
-- SELECT 2 as PLACE, b.* FROM BEINGS b where b.ID = (SELECT EMPLOYER_ID FROM BEINGS WHERE NAME = 'Sam' )
-- -- UNION ALL
--)
--SELECT * FROM BEINGS_OF_ARDA_TREE
```

Ale zaraz! Zapytacie skąd wiedziałem, że trzeba zmienić relację i w jaki sposób ? Spójrzcie proszę na zakomentowany kod. Gdy spróbujecie go wykonać i przeanalizować wyda się wam oczywiste, że relacja musi wyglądać trochę inaczej. Jeżeli jeszcze macie jakieś wątpliwości, polecam wam stworzyć własny model hierarchiczny np. folder – katalog i na nim poćwiczyć te zapytania, a już wkrótce wyda wam się to bardzo naturalne.

Z rzeczy które jeszcze warto dodać… Dla wywołań rekurencyjnych bardzo istotne jest by między jednym, a drugim zapytaniem był UNION ALL, w przeciwnym razie baza wywali błąd. Ponadto pomimo że WITH znajduje się w standardzie różnie bywa ze wsparciem w zależności od dostawcy SZBD. Niektórzy jak np. Postgres wymagają słowa RECURSIVE (WITH RECURSIVE cte AS… ). Zawsze sprawdzajcie na co pozwala wam dostawca zanim oprzecie o coś swoje rozwiązanie.

To tyle w temacie CTE. Dziękuję wszystkim, którzy dotarli aż tutaj 🙂