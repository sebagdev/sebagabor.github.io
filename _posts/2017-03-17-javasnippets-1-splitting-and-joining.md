---
id: 82
title: 'WeeklyJavaSnippets#1 Splitting and joining Strings in Java'
date: '2017-03-17T20:00:13+01:00'
author: sg
layout: post
guid: 'http://sgdev.pl/?p=82'
permalink: /2017/03/17/javasnippets-1-splitting-and-joining/
categories:
    - 'Java Snippets'
tags:
    - guava
    - java
    - snippet
---

Wpadłem na pomysł kolejnego cyklu, który mógłby się przewijać na tym blogu, stanowiącego zbiór krótkich kawałków kodu czasem oczywistych, czasem nie, które… robią swoją robotę. Czy kiedykolwiek czułeś, że wynajdujesz koło po raz n-ty? Czy pozornie prosta operacja to kolejne i kolejne linijki kodu? Wezwij Drużynę A… Wybaczcie zapędziłem się. Po prostu zerknij na Java Snippets! A zatem nie przeciągając…

#### String – split i join

Założę się, że nie raz zazdrościliście Pythonowi:

```
>>> lst = ["Ania", "Basia", "Czesia"]
>>> dziewczyny = ",".join(lst)
>>> print(dziewczyny)
Ania,Basia,Czesia
>>> dziewczyny.split(',')
['Ania', 'Basia', 'Czesia']
```

Programując w Javie w mrocznych czasach Javy 6 i 7, nie było tak różowo:

```
public static String myjoin(Iterable<String> iterable, String delimeter){
    StringBuilder sb = new StringBuilder();
    for (String s : iterable){
        if (s!=null) {
            sb.append(s);
            sb.append(delimeter);
        }
    }
    if (sb.length()>0) {
        sb.delete(sb.lastIndexOf(delimeter), sb.length());
    }
    return sb.toString();
}
public static void main(String[] args){
    List<String>  lst = Arrays.asList("Ania", "Basia", "Czesia");
    String dziewczyny = myjoin(lst, ",");
    System.out.println(dziewczyny);
    System.out.println(Arrays.asList(dziewczyny.split(",")));
}
```

Tak brakowało prostego joina, a więc każdy projekt musiał mieć dodatkową bibliotekę lub własną koślawą implementację. Sytuacja zmieniła się po nadejściu Javy 8 (fanfary):

```
public static void main1(String[] args){
    // Java 8 - style
    List<String>  lst = Arrays.asList("Ania", "Basia", "Czesia");
    String dziewczyny = String.join(",",lst);
    System.out.println(dziewczyny);
    System.out.println(dziewczyny.split(","));
}
```

Myślę, że wielu Javowców odetchnęło wtedy z ulgą, ale chciałem napisać o czymś z czego korzystam niejako z przyzwyczajenia -&gt; [Guava](https://github.com/google/guava). Ta Googlowska biblioteka ma naprawdę wiele funkcjonalności o których zapomniano w bibliotece standardowej, ponadto wiele rzeczy które pojawiły się w niej jako pierwsze ostatecznie wylądowały w Javie 8. Trzymajcie Guavę w waszych projektach jak trzymacie gaśnicę w waszych samochodach!

A co oferuje w zakresie rozdzielania i łączenia Stringów ?

```
List<String>  lst = Arrays.asList("Ania", "Basia", "Czesia");
String dziewczyny = Joiner.on(',').join(lst);  // Joining Guava style
System.out.println(dziewczyny);
System.out.println(Splitter.on(",").splitToList(dziewczyny));
```

Co istotne poradzi sobie także w bardziej nietypowych sytuacjach, gdy dane są nieco zanieczyszczone nullami lub pustymi Stringami, za pomocą eleganckiego płynnego API konstruując obiekt obsługujący nawet nasz bardziej skomplikowany przypadek:

```
public static void main(String[] args){
    List<String>  lst = Arrays.asList("Ania", "", "Basia   ", null, "Czesia   ");
    String dziewczyny = Joiner.on(',').skipNulls().join(lst);  // Joining Guava style
    System.out.println(dziewczyny);
    System.out.println(Splitter.on(",").omitEmptyStrings().trimResults().splitToList(dziewczyny));
}
```

```
Ania,,Basia   ,Czesia   
[Ania, Basia, Czesia]
```

To tyle. Zakładałem, żeby artykuliki w tym cyklu były możliwie krótkie, ale i tak przekroczyłem zakładaną ilość słów 🙂 Dzięki, że dotarliście aż tutaj. Do zobaczenia!