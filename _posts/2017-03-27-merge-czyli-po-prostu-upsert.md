---
id: 76
title: 'Nie taki znów zwykły SQL Server cz. 3 &#8211; Merge czyli po prostu UPSERT'
date: '2017-03-27T18:00:40+02:00'
author: sg
layout: post
guid: 'http://sgdev.pl/?p=76'
permalink: /2017/03/27/merge-czyli-po-prostu-upsert/
categories:
    - 'Nie taki znów zwykły SQL'
tags:
    - sql
    - 'sql server'
---

#### Merge, upsert – o co chodzi ?

Merge ? Nigdy tego nie używałem, z tego w ogóle się korzysta ?! Mniej więcej, taka moja była pierwsza reakcja, kiedy napotkałem kod zawierający składnię owego wyrażenia. Otóż używa się i ma wiele zastosowań, gdyż potrafi czasem zastąpić aż trzy różne operacje INSERT, UPDATE, DELETE. Czy istnieje jakieś konkretny uniwersalny przypadek użycia dla MERGE ? Szczerze mówiąc ciężko mi takowy przytoczyć. Podobnie jak w przypadku wcześniej opisywanego WITH potrafi ułatwić rozwiązywanie pewnej klasy problemów, bez angażowania rozbudowanej procedury T-SQL lub języków programowania.

[Składnia](https://docs.microsoft.com/en-us/sql/t-sql/statements/merge-transact-sql#syntax) wyrażenia jest dość skomplikowana na pierwszy rzut oka, jednak po parokrotnym użyciu można szybko poczuć co można uzyskać korzystając z niego, a co jest niemożliwe. Najlepiej zresztą poczuć moc na przykładzie.

#### Merge – przykład

Wyobraźmy sobie, że mamy przestarzały system zgłoszeniowy, którego statusów potrzebuje nasza aplikacja (np. do wyświetlenia). Oczywiście ten stary system wciąż działa, zatem co jakiś czas muszą przychodzić aktualizacje.

```
USE ShowCaseDb;

-- Stwórzmy tabelkę reprezentującą dane z systemu zgłoszeniowego
CREATE TABLE LS_TICKETS (
    ID INT IDENTITY NOT NULL PRIMARY KEY, 
    TICKET_NO NVARCHAR(40) NOT NULL UNIQUE,
    TICKET_ASSIGNEE NVARCHAR(40) NOT NULL,
    TICKET_STATUS NVARCHAR(10) NULL
)
GO

INSERT INTO LS_TICKETS (TICKET_NO,TICKET_ASSIGNEE,TICKET_STATUS) 
    VALUES(N'00001001', N'John', N'Resolved');
INSERT INTO LS_TICKETS (TICKET_NO,TICKET_ASSIGNEE,TICKET_STATUS) 
    VALUES(N'00001002', N'Paul', N'Closed');
INSERT INTO LS_TICKETS (TICKET_NO,TICKET_ASSIGNEE,TICKET_STATUS) 
    VALUES(N'0001000l', N'Ringo', N'Invalid');    
INSERT INTO LS_TICKETS (TICKET_NO,TICKET_ASSIGNEE,TICKET_STATUS) 
    VALUES(N'0000O201', N'George', N'Resolved');
```

Normalną i częstą praktyką jest, że dane przychodzące z zewnętrznego systemu lądują nie od razu w docelowej tabeli, z której korzysta aplikacja, ale z tabeli pośredniej (staging’owej) z której dopiero są ładowane do tabeli roboczej. Przykładowo szyna integracyjna przysyła nam nową partię danych:

```
-- Nie zapomnijmy o przygyotowaniu sobie wcześniej tabeli :)
CREATE TABLE LS_TICKETS_STAGING (
    ID INT IDENTITY NOT NULL PRIMARY KEY, 
    TICKET_NO NVARCHAR(40) NOT NULL UNIQUE,
    TICKET_ASSIGNEE NVARCHAR(40) NOT NULL,
    TICKET_STATUS NVARCHAR(10) NULL
)
GO

INSERT INTO LS_TICKETS_STAGING (TICKET_NO,TICKET_ASSIGNEE,TICKET_STATUS) 
    VALUES(N'00001001', N'John', N'Reopened');
INSERT INTO LS_TICKETS_STAGING (TICKET_NO,TICKET_ASSIGNEE,TICKET_STATUS) 
    VALUES(N'0000O201', N'George', N'Reopened');    
INSERT INTO LS_TICKETS_STAGING (TICKET_NO,TICKET_ASSIGNEE,TICKET_STATUS) 
    VALUES(N'0000O203', N'Keith', N'Open');
```

A więc na naszej tabeli mamy:

[![](https://sgdev.pl/wp-content/uploads/2017/03/scrin02-300x93.png)](https://sgdev.pl/wp-content/uploads/2017/03/scrin02.png)

A tabela staging’owa z aktualizacjami wygląda mniej więcej tak:

[![](https://sgdev.pl/wp-content/uploads/2017/03/Zaznaczenie_012-300x73.png)](https://sgdev.pl/wp-content/uploads/2017/03/Zaznaczenie_012.png)

Chcąc wyrównać te dane standardową drogą powinniśmy. Napisać SELECT’a który znajdzie odpowiadające sobie wiersze, a następnie wykona UPDATE kolumn statusowych, a w przypadku gdy danego id nie ma na bazie zostanie wykonany INSERT. Trochę kodu do napisania przed nami… Otóż nie!

```
MERGE INTO LS_TICKETS lt USING LS_TICKETS_STAGING lts ON (lt.TICKET_NO = lts.TICKET_NO) -- merge pozwala nam połączyć ze sobą dwie tabele
WHEN MATCHED THEN -- w przypadku, gdy mamy odpowiadające sobie wiersze to wykonajmy update:
UPDATE SET lt.TICKET_ASSIGNEE = lts.TICKET_ASSIGNEE, lt.TICKET_STATUS = lts.TICKET_STATUS 
WHEN NOT MATCHED THEN -- gdy takowych nie ma - znaczy że mamy nowe zgłoszenie!
INSERT (TICKET_NO,TICKET_ASSIGNEE,TICKET_STATUS) 
    VALUES(lts.TICKET_NO, lts.TICKET_ASSIGNEE, lts.TICKET_STATUS);
```

W wyniku otrzymamy syntezę dwóch tabel:

[![](https://sgdev.pl/wp-content/uploads/2017/03/Zaznaczenie_013-300x108.png)](https://sgdev.pl/wp-content/uploads/2017/03/Zaznaczenie_013.png)

#### Podsumowanie

Merge potrafi scalić kilka instrukcji w jedną, jednak niestety nie jest super wydajny – konieczny jest każdorazowy pełny skan tabeli. Z drugiej strony, czy powierzając te same zadania kilku insert’om, update’om uzyskamy lepszą wydajność ? Warto zwrócić uwagę na jeszcze jedną ciekawą rzecz. To wszystko jest część standardu SQL:2003! Niestety z implementacją w różnych SZBD nie ma już tak dobrze – najgorzej jest w tym przypadku z Postgres’em. Poszczególni dostawcy zadbali jednak o swoje odpowiedniki i rozszerzenia, zatem gdy musicie znaleźć coś podobnego wpisujcie UPSERT &lt;Nazwa waszego dostawcy&gt;.

To by było na tyle. Standardowo dzięki wszystkim, którzy dotarli aż tutaj! 🙂