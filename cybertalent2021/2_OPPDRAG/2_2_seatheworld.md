# 2.2 Seatheworld

Vi har tidligere funnet `www.seatheworld.tld` og `ns1.seatheworld.tld` i pcap filen. Bruker vi `nmap` mot `www.seatheworld.tld` finner vi en webserver på port 80 som kjører en relativt lite givende Down-for-maintainance side. `ns1.seatheworld.tld` viser seg å være en DNS server.

## 2.2_seatheworld

Vi ser jo at både ns1 og www ligger på samme domene. Kanskje ligger det mere her?

```sh
$ dig AXFR @ns1.seatheworld.tld seatheworld.tld

; <<>> DiG 9.11.5-P4-5.1+deb10u2-Debian <<>> AXFR @ns1.seatheworld.tld seatheworld.tld
; (1 server found)
;; global options: +cmd
seatheworld.tld. 600 IN SOA ns1.seatheworld.tld. ns1.seatheworld.tld. 12345678 1200 180 1209600 600
seatheworld.tld. 600 IN NS ns1.seatheworld.tld.
seatheworld.tld. 600 IN MX 10 mail.seatheworld.tld.
\_flagg.nusse.seatheworld.tld. 600 IN TXT "ea330a8b........................"
mail.seatheworld.tld. 600 IN CNAME www.seatheworld.tld.
ns1.seatheworld.tld. 600 IN A 10.1.4.188
www.seatheworld.tld. 600 IN A 10.1.4.176
seatheworld.tld. 600 IN A 10.1.4.176
nusse.seatheworld.tld. 600 IN A 10.1.4.169
seatheworld.tld. 600 IN SOA ns1.seatheworld.tld. ns1.seatheworld.tld. 12345678 1200 180 1209600 600
;; Query time: 1 msec
;; SERVER: 10.1.4.188#53(10.1.4.188)
;; WHEN: Sat Jan 16 22:17:24 CET 2021
;; XFR size: 10 records (messages 1, bytes 450)
```

Der har vi flagger for 2.2, samt vi finner hostname `nusse.seatheworld.tld`

## 2.2_seatheworld_aksess

`nusse` har flere porter åpne; FTP på port `21`, SSH på `22`, en webserver på `80` og Telnet (!!) på `23`. Lang historie kort; så viser det seg at det eneste nyttige for oss er telnet porten. `nmap` sier dette er en `tn3270` service (IBM Telnet TN3270 (TN3270E)) - som er [en terminal emulator](https://en.wikipedia.org/wiki/TN3270_Plus). Man får også noen hint her dersom man besøker nettsiden på port 80.

Så flaks da.. at corax har et program som heter [c3270](http://x3270.bgp.nu/Unix/c3270-man.html) installert! Også veldig flaks at mainframe'en kjører default admin passord :D

```sh
c3270> help
c3270> connect nusse.seatheworld.tld
>>> LOGON IBMUSER --> passord 'sys1'
```

Vi får logget oss inn, og da får vi også flagget for dette steget 🥳

## 2.2_seatheworld_booking && 2.2_seatheworld_ekstra

Grunnen til at vi er inne å snoker her i utgangspunktet er at vi i chat-applikasjonen fikk hint nok til å skjønne at det er her vi kommer til å finne flybilletter - altså navnene på gjerningspersonene.

For å få til dette må vi nå lære oss hvordan IBM terminalen vi befinner oss i fungerer - spennende!

Når vi er logget inn kommer det opp et lite cheat-sheet med kommandoer. Det finnes fler, men alle kommandoene vi trenger vises faktisk her allerede.

Vi kjører `LOAD IBMUSER.SEATW.LOAD(BOOKING)` og kommer inn i bookingsystemet. Her kan vi lete på booking referanse, navn eller dato. Vi fikk referansen `WJWQX` fra chatten, og da kommer vi som ventet til Frank Shorius sin reservasjon:

```
WJWQX FRANK SHORIUS AMS 14.12.2020 18:20 - TXA 14.12.2020 22:40
```

Vi kan tenke oss at resten av gjerningsmennene også reiste den 14.12, men hvis vi forsøker søke på denne datoen får vi en melding om at vi ikke får lov å søke på historiske reiser.

Litt mer leting rundt omkring så finner vi data settene (vi kan tenke på de som filer):

```
IBMUSER.SEATW.BOOKINGS
IBMUSER.SEATW.LOAD
IBMUSER.SEATW.SRC
```

`BOOKINGS` et data-sett (fil) med ALLE bookingene, og `LOAD` er programmet som "laster" BOOKINGS programmet. `SRC` derimot viser seg å være kildekoden!

Vi kan bruke terminal programmet `ISPF` til å editere kildekoden. Endrer:

```c
000005 #define SKIP FLAG 1 // sette denne til 0
```

Kommenterer vekk hele blokken som sjekker om vi søker for en dato tilbake i tid:

```c
000225 if (year < current_time->tm_year ...
...
000231    printf("WONT SEARCH FOR HISTORICAL JOURNEYS\n");
...
000233 }
```

Vi compiler koden med kommandoen `SUBMIT IBMUSER.SEATW.SRC(JCL)` og når vi nå LOAD'er booking programmet igjen får vi et flagg helt øverst i programmet vårt - samt muligheten til å søke frem alle andre som fløy den 14.12.2020:

```
WTHHX JOE FARNABY     AMS 14.12.2020 18:20 - TXA 14.12.2020 22:40
DY6MY EIREEN FARNABY  AMS 14.12.2020 18:20 - TXA 14.12.2020 22:40
UHKVP DILLON CRUISE   AMS 14.12.2020 18:20 - TXA 14.12.2020 22:40
WJWQX FRANK SHORIUS   AMS 14.12.2020 18:20 - TXA 14.12.2020 22:40
```

Dermed har vi flere navn, og dersom vi sender inn ett av de får vi siste flagget på denne delen av oppdraget.
