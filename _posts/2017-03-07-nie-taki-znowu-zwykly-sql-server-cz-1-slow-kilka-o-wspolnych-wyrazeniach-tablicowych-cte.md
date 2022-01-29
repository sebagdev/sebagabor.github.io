---
id: 29
title: 'Nie taki znowu zwykły SQL Server cz. 1 &#8211; słów kilka o CTE'
date: '2017-03-07T20:47:18+01:00'
author: sg
layout: post
guid: 'http://sgdev.pl/?p=29'
permalink: /2017/03/07/nie-taki-znowu-zwykly-sql-server-cz-1-slow-kilka-o-wspolnych-wyrazeniach-tablicowych-cte/
categories:
    - 'Nie taki znów zwykły SQL'
tags:
    - sql
    - 'sql server'
    - with
---

Niniejszym artykułem chciałbym rozpocząć cykl poświęcony przydatnym, ale rzadziej wykorzystywanym elementom języka SQL jak i samego MS SQL Server’a. Podstawowe operacje CRUD wykonujemy bez większych trudności już po krótkim szkoleniu, jednak niejednokrotnie brakuje narzędzia upraszczającego życie oraz lepiej pasującego do aktualnych okoliczności.

  
Aby nie sprowadzać kolejnych artykułów do suchej wiedzy w miarę możliwości będę starał się nadawać im formę krótkich tutoriali, wraz z rzeczywistymi przypadkami użycia, na które można natrafić w codziennej pracy.

#### Środowisko

*Poniższy tekst będzie zawierał kilka przemyśleń na temat środowiska na jakim przeprowadziłem dalsze operacje. Śmiało możesz przejść dalej 🙂*

Przykłady będę uruchamiał na MS SQL Server – jest to najczęściej używana przeze mnie zawodowo baza danych, co więcej jej popularność na rynku rośnie z każdym rokiem. Jako, że prywatnie używam Linux’a myślę, że całkiem fajnym eksperymentem byłoby przetestowanie wersji Preview bazy. Zasadniczo wybór środowiska nie powinien mieć znaczenia.

Jeżeli chcecie podążyć moją ścieżką sugeruję instalację za pomocą dockera:

```
$ sudo docker run -e 'ACCEPT_EULA=Y' -e 'SA_PASSWORD=yourSAPassword1(!)' -p 1433:1433 -d microsoft/mssql-s
```

Ważna uwaga – stosujcie się do wymogów dot. hasła w przeciwnym wypadku może być tak, że kontener po prostu nie wystartuje i po kilku sekundach zobaczymy, że się po prostu wyłączył. Do weryfikacji, czy wszystko chodzi ok można użyć:

```
$ sudo docker ps -a
CONTAINER ID        IMAGE                          COMMAND                  CREATED             STATUS                  PORTS                    NAMES
b9c083f96995        microsoft/mssql-server-linux   "/bin/sh -c /opt/m..."   4 days ago          Up 2 seconds            0.0.0.0:1433->1433/tcp   affectionate_galileo
```

Niestety MS nie przygotował (zapewne jeszcze, choć nie śledzę wieści ze społeczności) Management Studio pod Linux’a, póki co zalecają korzystanie z Visual Studio Code z zainstalowaną wtyczką o jakże oryginalnej nazwie *mssql for Visual Studio Code.*

#### CTE – niezbędne minimum wiedzy

CTE, które można przetłumaczyć na język polski jako wspólne wyrażenia tablicowe stanowią element standardu języka SQL wprowadzony już w 1999 roku, przede wszystkim ze względu na konieczność obsługi modeli hierarchicznych. CTE łatwo dostrzec wśrod zapytań po słowie kluczowym WITH i należy o nich myśleć, bez zbędnego wchodzenia w szczegóły jako o zapytaniach budujących tymczasową pośrednią tabelę w pamięci z „podzapytań”. Stop, stop ktoś mógłby zakrzyknąć, ale czy nie mamy już podzapytań działających tak samo ?  
Otóż, i tak i nie – specjalną cechą CTE, jest możliwość obsługi rekurencji, ale do tego dojdziemy… w swoim czasie.

Jak wygląda zatem typowe wyrażenie tablicowe ? Najczęściej piszemy coś podobnego do kodu poniżej:

```
WITH nazwa_wyrazenia (kolumna_1, kolumna_2) AS (
 -- tu definicja wyrażenia, przeważnie SELECT
SELECT col_1 as kolumna_1, col_2 as kolumna_2 FROM tabela
)
--operacja na wyrażeniu np. SELECT
SELECT * FROM nazwa_wyrazenia
```

Warto zwrócić uwagę, że definicja kolumn nie jest tak naprawdę potrzebna, o czym przekonamy się już niedługo. Przechodząc szybciutko do praktyki.

#### Przypadek użycia nr 1 – usuwanie zduplikowanych wersji w przyjemny sposób

```
-- Na czystym SQL Server muszę stworzyć bazę

CREATE DATABASE ShowCaseDb;

USE ShowCaseDb;

-- Stwórzmy tabelkę
CREATE TABLE USERS (
    ID INT IDENTITY NOT NULL PRIMARY KEY, 
    NAME NVARCHAR(40) NOT NULL,
    SURNAME NVARCHAR(40) NOT NULL,
    ORIGIN NVARCHAR(40) NOT NULL
)

GO


-- I wypełnijmy ją danymi 
-- Do tych wierszy będziemy wracać w dalszej części przykładu, 
-- ale nie będę ich replikował w tym artykule

INSERT INTO USERS (NAME,SURNAME,ORIGIN) VALUES(N'Eddie', N'Dean', N'New York');
INSERT INTO USERS (NAME,SURNAME,ORIGIN) VALUES(N'Susannah', N'Dean', N'New York');
INSERT INTO USERS (NAME,SURNAME,ORIGIN) VALUES(N'Jake', N'Chambers', N'New York');
INSERT INTO USERS (NAME,SURNAME,ORIGIN) VALUES(N'Ey', N'Bumbler', N'MidWorld');
INSERT INTO USERS (NAME,SURNAME,ORIGIN) VALUES(N'Roland', N'Deschain', N'Gilead');
INSERT INTO USERS (NAME,SURNAME,ORIGIN) VALUES(N'Eddie', N'Dean', N'New York');
INSERT INTO USERS (NAME,SURNAME,ORIGIN) VALUES(N'Susannah', N'Dean', N'New York');
INSERT INTO USERS (NAME,SURNAME,ORIGIN) VALUES(N'Jake', N'Chambers', N'New York');
INSERT INTO USERS (NAME,SURNAME,ORIGIN) VALUES(N'Ey', N'Bumbler', N'MidWorld');
GO

-- Zweryfikujmy co otrzymaliśmy
select * from USERS;
GO

```

Jak widać troszkę omsknęła nam się dłoń podczas wstawiania danych i mamy dwa razy więcej wierszy niż byśmy chcieli:

![](https://sgdev.pl/wp-content/uploads/2017/03/scrin1-300x200.png)

Co poradzić w takiej sytuacji ? Intuicja podpowiada nam, że musimy znaleźć wiersze które chcemy usunąć i… je usunąć. Gdybyśmy np. ponumerowali powtarzające się wiersze i zostawili sobie tylko jeden, wtedy wszystko powinno być ok. Użyjmy zatem funkcji okienkowej, rozdzielającej numerację względem wszystkich kolumn poza PK.

```
SELECT ROW_NUMBER() OVER (PARTITION BY NAME,SURNAME,ORIGIN order by ID ) AS RN, *  FROM USERS
```

![](https://sgdev.pl/wp-content/uploads/2017/03/scrin2-300x160.png)

Jak widać na obrazku jest już znacznie lepiej, teraz tylko wystarczy pozostawić wiersz o RN=1 i na końcu odpalić DELETE.

```
SELECT ROW_NUMBER() OVER (PARTITION BY NAME,SURNAME,ORIGIN order by ID ) AS RN, *  FROM USERS 

-- SELECT ROW_NUMBER() OVER (PARTITION BY NAME,SURNAME,ORIGIN order by ID ) AS RN, *  FROM USERS  WHERE RN > 1
-- Msg 207, Level 16, State 1, Line 1
-- Invalid column name 'RN'. 

SELECT ROW_NUMBER() OVER (PARTITION BY NAME,SURNAME,ORIGIN order by ID ) AS RN, *  FROM USERS WHERE ROW_NUMBER() OVER (PARTITION BY NAME,SURNAME,ORIGIN order by ID ) > 1
-- Msg 4108, Level 15, State 1, Line 1
-- Windowed functions can only appear in the SELECT or ORDER BY clauses.
```

Niestety nie ma tak lekko… funkcja okienkowa zmusza nas do użycia podzapytania. Czy nie wspominałem, że CTE są bardzo zbliżone do podzapytań ? Nie uprzedzając faktów:

```
SELECT * FROM (
    SELECT ROW_NUMBER() OVER (PARTITION BY NAME,SURNAME,ORIGIN order by ID ) AS RN, *  FROM USERS
) INNER_QUERY WHERE INNER_QUERY.RN>1;
```

![](https://sgdev.pl/wp-content/uploads/2017/03/scrin3-300x88.png)

Czyż nie o to nam chodziło teraz tylko usuwać!

```
DELETE USERS FROM (
    SELECT ROW_NUMBER() OVER (PARTITION BY NAME,SURNAME,ORIGIN order by NAME ) AS RN, NAME,SURNAME,ORIGIN  FROM USERS
    )  INNER_QUERY 
WHERE INNER_QUERY.RN > 1;
```

Ale jak to *10 rows affected* ?! Cóż wygląda na to że pośpiech jest dobry przy łapaniu pcheł i wyczyściliśmy sobie całą tabelkę. Załadujmy wiersze jeszcze raz i… czy potraficie dostrzec pomyłkę ?

```
-- Tak naprawdę nie chodziło nam o tabelkę, a o wynik podzapytania 
-- z numeracją, co w konsekwencji spowoduje usunięcie wierszy 
-- z tabeli pod spodem

DELETE INNER_QUERY FROM (
    SELECT ROW_NUMBER() OVER (PARTITION BY NAME,SURNAME,ORIGIN order by NAME ) AS RN, NAME,SURNAME,ORIGIN  FROM USERS
    )  INNER_QUERY 
WHERE INNER_QUERY.RN > 1;
```

Okej okej, ale gdzie miejsce dla CTE w tym wszystkim, ktoś zapyta? Jak już wspominałem można je użyć jako podzapytanie.

```
WITH USERS_CTE AS (
    SELECT ROW_NUMBER() OVER (PARTITION BY NAME,SURNAME,ORIGIN order by NAME ) AS RN, NAME,SURNAME,ORIGIN  FROM USERS
    )   
SELECT * FROM USERS_CTE WHERE RN > 1;
```

![](https://sgdev.pl/wp-content/uploads/2017/03/scrin4-300x113.png)

A teraz pomyślmy jakby tu usunąć te rekordy ? Wystarczy SELECT \* zamienić na DELETE.

```
WITH USERS_CTE AS (
    SELECT ROW_NUMBER() OVER (PARTITION BY NAME,SURNAME,ORIGIN order by NAME ) AS RN, NAME,SURNAME,ORIGIN  FROM USERS
    )   
DELETE FROM USERS_CTE WHERE RN > 1;
```

*5 rows affected* czy nie o to nam właśnie chodziło? Sami przyznajcie czy ta droga nie jest o wiele łatwiejsza ? Moim zdaniem sam sposób dekompozycji problemu na samym początku na mniejsze podzapytania, jest najlepszym podejściem jakie może być. Odnosimy wrażenie, że operujemy na prawdziwej tabeli istniejącej gdzieś w bazie, a nie na zwykłym CTE’ku.

Nasz wpis zrobił się strasznie długi, a w temacie CTE do powiedzenia zostało jeszcze trochę. Prawdziwą potęgę wyrażeń tablicowych zostawię do następnego odcinka naszego cyklu.

Proszę miejcie na uwadze jedno. CTE nie są „złotym młotkiem”, nie ma czegoś takiego w IT, zatem jeśli macie zamiar przepisywać wszystkie wasze wyrażenia SQL z podzapytaniami, zastanówcie się czy na pewno warto. W najprostszych przypadkach nie uzyskacie raczej poprawy czytelności, a co do wydajności – cóż też często jest zbliżona do podzapytań, myślę że każdy przypadek wymagałby osobnej weryfikacji.  
Jedno jest pewne w tych najbardziej zakręconych zapytaniach na pewno nie zgubicie się tworząc kilka logicznych CTE’ków, które następnie odpowiednio połączycie.

Tyle na dziś. Dzięki wszystkim którzy dotarli aż tutaj 🙂 Do następnego razu!