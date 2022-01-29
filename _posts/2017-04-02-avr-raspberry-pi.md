---
id: 115
title: 'AVR + Raspberry PI = ?'
date: '2017-04-02T21:00:39+02:00'
author: sg
layout: post
guid: 'http://sgdev.pl/?p=115'
permalink: /2017/04/02/avr-raspberry-pi/
categories:
    - 'Raspberry PI'
tags:
    - avr
    - C
    - 'raspberry pi'
---

#### Grzebanie w starociach

Ostatnio przegrzebywałem się przez moje szuflady z elektroniką w poszukiwaniu cyny. Nie bawiłem się w lutowanie od czasów pierwszych lat studiów i technikum, ot takie porzucone hobby, nie hobby. Nagle w ręce wpadł mi scalak:

![](https://sgdev.pl/wp-content/uploads/2017/03/IMG_20170326_150558725-300x169.jpg)](https://sgdev.pl/wp-content/uploads/2017/03/IMG_20170326_150558725.jpg)

Po chwilę wpadło mi ręce jeszcze kilka podobnych, lecz niestety mój programator USB zaginął w akcji jakiś czas temu, Arduino nigdy nie posiadałem, a współczesne komputery nie mają już ani RS232 ani Centronics. Rozczarowany już przerzucałem je do pudełka, gdy me oczy spoczęły na… malince. Tak głupcze! Porty GPIO powinny się nadać. Zanurkowałem w internety jak do tego podejść. Avrdude… no jasne to powinno zadziałać! Niestety brak paczki w repo raspbiana oznaczał kompilację ze źródeł.

```
$ sudo apt-get install bison flex -y
$ wget http://download.savannah.gnu.org/releases/avrdude/avrdude-6.2.tar.gz
$ tar zxvf avrdude-6.2.tar.gz 
$ cd avrdude-6.2
$ ./configure --enable-linuxgpio
$ make
$ sudo make install
```

Make, trochę potrwał z uwagi, że kompilowaliśmy paczkę na malince. Dodatkowo w pliku/usr/local/etc/avrdude.conf konieczne były małe zmiany konfiguracyjne. Odkomentowanie sekcji z linuxgpio i ustawienie odpowiednich pin’ów.

```
  id    = "linuxgpio";
  desc  = "Use the Linux sysfs interface to bitbang GPIO lines";
  type  = "linuxgpio";
  reset = 4;
  sck   = 11;
  mosi  = 10;
  miso  = 9;
;
```

Spojrzałem na rozkład wyprowadzeń maliny: <https://www.raspberrypi.org/documentation/usage/gpio/README.md>. Z 3.3 V zasiliłem attiny, podpiąłem odpowiadające wyprowadzenie ustawione w AVR DUDE do pinów z interfejsu SPI, zgodnie z [notą katalogową](http://ww1.microchip.com/downloads/en/DeviceDoc/Atmel-2543-AVR-ATtiny2313_Summary.pdf).

Czyli podsumowując połączyłem jak niżej:

| Pin | Pi PIN | ATTiny PIN |
|---|---|---|
| VDD | 1 | 20 |
| GND | 25 | 10 |
| RESET | 4 | 1 |
| MISO | 9 | 18 |
| MOSI | 10 | 17 |
| SCK | 11 | 19 |

Przyszła pora na pogadanie z mikrokontrolerem:

```
$ sudo avrdude -c linuxgpio -p attiny2313 -v

avrdude: Version 6.2, compiled on Mar 25 2017 at 17:13:29
 Copyright (c) 2000-2005 Brian Dean, http://www.bdmicro.com/
 Copyright (c) 2007-2014 Joerg Wunsch

 System wide configuration file is "/usr/local/etc/avrdude.conf"
 User configuration file is "/root/.avrduderc"
 User configuration file does not exist or is not a regular file, skipping

 Using Port : unknown
 Using Programmer : linuxgpio
 AVR Part : ATtiny2313
 Chip Erase delay : 9000 us
 PAGEL : PD4
 BS2 : PD6
 RESET disposition : possible i/o
 RETRY pulse : SCK
 serial program mode : yes
 parallel program mode : yes
 Timeout : 200
 StabDelay : 100
 CmdexeDelay : 25
 SyncLoops : 32
 ByteDelay : 0
 PollIndex : 3
 PollValue : 0x53
 Memory Detail :

 Block Poll Page Polled
 Memory Type Mode Delay Size Indx Paged Size Size #Pages MinW MaxW ReadBack
 ----------- ---- ----- ----- ---- ------ ------ ---- ------ ----- ----- ---------
 eeprom 65 6 4 0 no 128 4 0 4000 4500 0xff 0xff
 flash 65 6 32 0 yes 2048 32 64 4500 4500 0xff 0xff
 signature 0 0 0 0 no 3 0 0 0 0 0x00 0x00
 lock 0 0 0 0 no 1 0 0 9000 9000 0x00 0x00
 lfuse 0 0 0 0 no 1 0 0 9000 9000 0x00 0x00
 hfuse 0 0 0 0 no 1 0 0 9000 9000 0x00 0x00
 efuse 0 0 0 0 no 1 0 0 9000 9000 0x00 0x00
 calibration 0 0 0 0 no 2 0 0 0 0 0x00 0x00

 Programmer Type : linuxgpio
 Description : Use the Linux sysfs interface to bitbang GPIO lines
 Pin assignment : /sys/class/gpio/gpio{n}
 RESET = 4
 SCK = 11
 MOSI = 10
 MISO = 9

avrdude: AVR device initialized and ready to accept instructions

Reading | ################################################## | 100% 0.00s

avrdude: Device signature = 0x1e910a (probably t2313)
avrdude: safemode: hfuse reads as DD
avrdude: safemode: efuse reads as FF

avrdude: safemode: hfuse reads as DD
avrdude: safemode: efuse reads as FF
avrdude: safemode: Fuses OK (E:FF, H:DD, L:64)

avrdude done. Thank you.
```

Hurra! 🙂 Czym by był mój eksperyment, gdybym nie próbował zamrugać diodą ?

#### Zaprogramujmy coś!

Napisałem krótki programik, jako ofiarę wybrałem sobie PIN6 z portu D, gdyż był wygodnie usytuowany. Podpiąłem diodę led i rezystor 1kOhm.

```
#include<avr/io.h>
#include<util/delay.h>

int main(void){
	DDRD |= _BV(6);
	while (1){
		_delay_ms(500);
		PORTD |= _BV(6);
		_delay_ms(500);
		PORTD &= ~(_BV(6));
	}
}
```

Nastepnie kolejno skompilowałem napisany kod, utworzyłem plik do wgrania na malinkę, i za pomocą avrdude przesłałem wszystko na malinę.

```
$ avr-gcc -mmcu=at90s2313 test.c -o test
$ avr-objcopy -O ihex  test test.hex
$ sudo avrdude -c linuxgpio -p attiny2313 -v -U flash:w:test.hex:i
```

Avr dude wyświetlił podsumowanie:

```
$ sudo avrdude -c linuxgpio -p attiny2313 -v -U flash:w:test.hex:i

avrdude: Version 6.2, compiled on Mar 25 2017 at 17:13:29
         Copyright (c) 2000-2005 Brian Dean, http://www.bdmicro.com/
         Copyright (c) 2007-2014 Joerg Wunsch

         System wide configuration file is "/usr/local/etc/avrdude.conf"
         User configuration file is "/root/.avrduderc"
         User configuration file does not exist or is not a regular file, skipping

         Using Port                    : unknown
         Using Programmer              : linuxgpio
         AVR Part                      : ATtiny2313
         Chip Erase delay              : 9000 us
         PAGEL                         : PD4
         BS2                           : PD6
         RESET disposition             : possible i/o
         RETRY pulse                   : SCK
         serial program mode           : yes
         parallel program mode         : yes
         Timeout                       : 200
         StabDelay                     : 100
         CmdexeDelay                   : 25
         SyncLoops                     : 32
         ByteDelay                     : 0
         PollIndex                     : 3
         PollValue                     : 0x53
         Memory Detail                 :

                                  Block Poll               Page                       Polled
           Memory Type Mode Delay Size  Indx Paged  Size   Size #Pages MinW  MaxW   ReadBack
           ----------- ---- ----- ----- ---- ------ ------ ---- ------ ----- ----- ---------
           eeprom        65     6     4    0 no        128    4      0  4000  4500 0xff 0xff
           flash         65     6    32    0 yes      2048   32     64  4500  4500 0xff 0xff
           signature      0     0     0    0 no          3    0      0     0     0 0x00 0x00
           lock           0     0     0    0 no          1    0      0  9000  9000 0x00 0x00
           lfuse          0     0     0    0 no          1    0      0  9000  9000 0x00 0x00
           hfuse          0     0     0    0 no          1    0      0  9000  9000 0x00 0x00
           efuse          0     0     0    0 no          1    0      0  9000  9000 0x00 0x00
           calibration    0     0     0    0 no          2    0      0     0     0 0x00 0x00

         Programmer Type : linuxgpio
         Description     : Use the Linux sysfs interface to bitbang GPIO lines
         Pin assignment  : /sys/class/gpio/gpio{n}
           RESET   =  4
           SCK     =  11
           MOSI    =  10
           MISO    =  9

avrdude: AVR device initialized and ready to accept instructions

Reading | ################################################## | 100% 0.01s

avrdude: Device signature = 0x1e910a (probably t2313)
avrdude: safemode: hfuse reads as DD
avrdude: safemode: efuse reads as FF
avrdude: NOTE: "flash" memory has been specified, an erase cycle will be performed
         To disable this feature, specify the -D option.
avrdude: erasing chip
avrdude: reading input file "test.hex"
avrdude: writing flash (1146 bytes):

Writing | ################################################## | 100% 1.38s

avrdude: 1146 bytes of flash written
avrdude: verifying flash memory against test.hex:
avrdude: load data flash data from input file test.hex:
avrdude: input file test.hex contains 1146 bytes
avrdude: reading on-chip flash data:

Reading | ################################################## | 100% 1.07s

avrdude: verifying ...
avrdude: 1146 bytes of flash verified

avrdude: safemode: hfuse reads as DD
avrdude: safemode: efuse reads as FF
avrdude: safemode: Fuses OK (E:FF, H:DD, L:64)

avrdude done.  Thank you.

```

Na koniec fotka otrzymanego pająka:

![](https://sgdev.pl/wp-content/uploads/2017/03/IMG_20170326_165219881-300x169.jpg)

#### Podsumowanie

No to tyle. Czemu w ogóle wrzucam ten wpis ? Po prostu ta przygoda uświadomiła mi, że czasem rozwiązania trudnych problemów są w zasięgu ręki, oraz ile zabawy można mieć z PI. Mam nadzieję, że was zachęci do przejrzenia starej elektroniki 🙂 To tyle, dzięki wszystkim którzy dotarli aż tutaj.

PS. jest wiele tutoriali ja w swojej zabawie z połączeniem tego wszystkiego do kupy opierałem się na świetnym http://ozzmaker.com/program-avr-using-raspberry-pi-gpio/. Jednak pamiętajcie, zawsze patrzcie na wyprowadzenia w zależności od układu który próbujecie podłączyć do maliny. Acha i rezystory 1kOhm na liniach GPIO używanych do SPI nie wydają się złym pomysłem 🙂 Ja po prostu nie miałem tylu rezystorów.