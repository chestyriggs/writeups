# 2_Oppdrag

Følgende write-up dekker stort sett de stegene jeg tok for å løse 2_oppdrag biten av årets CTF.

Ha litt i bakhodet at selv om ting virker ganske rett frem her, så er det mye søking, lesing, prøving og feiling som ligger bak :)

Merk også at måten jeg løste beslag_2 på ikke ser ut til å fungere senere i CTF'en.

## Innholdsfortegnelse

- [2.1 beslag](2_1_beslag.md)
- [2.2 seatheworld](2_2_seatheworld.md)
- [2.3 onion/scada](2_3_scada.md)

### Brief

_"Vi har mottatt en etterretningsrapport (INTREP) som krever umiddelbar oppmerksomhet. Rapporten ligger vedlagt."_

```
16 13:37Z DES 2020, INTREP
GRADERING:     TEMMELEG HEMMELEG
VÅR REFERANSE: SNTA1337N
PRIORITET:     KRITISK // FLASH

1. SITUASJONSSKILDRING

EI GRUPPE TERRORISTAR HAR I LØPET AV NATTA 16. DESEMBER TATT EIT
UKJEND TAL PÅ GISSEL OM BORD PÅ DET NORSKE FORSKINGSFARTYET MS
KOLDBJORN.  FARTYET VAR PÅ VEG TIL FORSKINGSBASEN TROLL, MEN HAR STÅTT
FAST I ISEN 177 NAUTISKE MIL NORD-AUST FOR BASEN SIDAN 30. NOVEMBER.

BAK AKSJONEN STÅR HØGST TRULEG PERSONAR KNYTTA TIL UTANLANDSKE
SEPARATISTGRUPPERINGAR SOM YNSKJER Å UNDERGRAVE NORSK SUVERENITET OVER
DRONNING MAUD LAND.  DET VART, RETT FØR AKSJONEN, RINGT INN TRUSLAR OM
KONKRETE HANDLINGAR I NÆR FRAMTID.

EIN PERSON SOM KAN KNYTTAST TIL ÅTAKET VART STOPPA AV EI SAMARBEIDANDE
TENESTE GRUNNA MISTENKELEG ÅTFERD.  I AVHØYR HAR PERSONEN INNRØMMA Å
VERA EIN DEL AV GRUPPA "FREE ANTARCTICA", MEN PERSONEN VART SLEPPT FRI
ETTER AVHØYR OG HAR REIST VIDARE TIL UKJEND STAD.  PERSONEN HAR TRULEG
REIST MED FALSKT NAMN.

AV DI DET KUN GÅR TO (2) ORDINÆRE RUTEFLY TIL OMRÅDET I ÅRET ER DET
TRULEG AT GISSELTAKARANE KOM TIL FARTYET VIA TROLL INTL AIRPORT (TXA)
FRÅ AMSTERDAM AIRPORT SCHIPHOL (AMS) DEN 14. DESEMBER.

EITT ELLER FLEIRE GISSEL ER OM BORD PÅ MS KOLDBJORN UTANFOR DRONNING
MAUD LAND.  SITUASJONEN ER UOVERSYNLEG.

2. GRUNNLAGSETTERRETNING

DET FINS LITE GRUNNLAGSETTERRETNING OM ORGANISASJONAN(E) ME REKNAR MED
STÅR BAK.  ME HAR OVER LENGRE TID FYLGT SEPARATISTGRUPPERINGAR SOM
YNSKJER EIT FRITT OG SJØLVSTENDIG ANTARKTIS, MEN ME HAR IKKJE VURDERT
AT DEI VIL VERA I STAND TIL Å UTFØRA VALDELEGE ÅTAK.  ME ER KJEND MED
TO GRUPPERINGAR SOM TRULEG KAN STÅ BAK:

    (1) ANTARKTIS OG FRIDOM (AF)
    (2) FREE ANTARCTICA

ORGANISASJONANE HAR OVER FLEIRE ÅR NYTTA REISEOPERATØREN "SEA THE
WORLD", SOM OPERERER FRÅ FYRSTEDØMET SEALAND, FOR Å LEGGJA TIL RETTE
FOR REISEVERKSEMDA.  OPERATØREN ER ILLGJETEN, DÅ DEI IKKJE GODTAR PASS
SOM LEGITIMASJON, MEN HELLER SVER PÅ "MOR OG FAR I DØDEN".  OPERATØREN
AKSEPTERER KUN KRYPTOVALUTA, OG ME SER IKKJE VEKK FRÅ AT
REISEOPERATØREN ER EIT DEKKSELSKAP FOR NEMNTE GRUPPERINGAR.
OPERATØREN HAR SLÅTT SEG OPP VED Å KAPRA OG RØVA NORSKE SKIP SOM
FRAKTAR KASSERT EDB-UTSTYR FOR EI REKKE NORSKE SELSKAP.

3. OPPDRAG

PRIORITERINGANE VÅRE ER:

    (1) FRIGJEVA GISSELET ELLER GISSELA
    (2) IDENTIFISERE GISSELTAKARANE
    (3) HALDA FAST PÅ NORSK SUVERENITET OVER DRONNING MAUD LAND

4. KVERRSETJING

EI SAMARBEIDANDE TENESTE HAR KVERRSETT FLEIRE TELEFONAR OG MINNEKORT
DISPONERT AV PERSONEN SOM VART AVHØYRT. DEI HAR OGSÅ FANGA NOKO
TRAFIKK FRÅ EIN AV TELEFONANE.

DET ER FØREBELS IKKJE GJORT ANALYSE AV KVERRSETJINGA.

SLUTT
```

### Undersøkelser

Vi får utlevert en pcap-fil med noe nettverkstrafikk og en data-dump fra telefonen som ble konfiskert.

Første steg blir å skaffe seg en oversikt over hva vi sitter med, og hvilken informasjon vi kan skaffe oss fort (lavthengende frukt).

Fra pcap filen er det ikke umiddelbart så veldig mye åpenbart - men vi finner to hostnames: `www.seatheworld.tld` og `ns1.seatheworld.tld`.

Fra telefondata dump'en finner vi filen `users.db`. Denne kan vi åpne i en eller annen SQLite browser og finner:

```md
| \_id | username         | firstname | lastname | password_hash                    |
| ---- | ---------------- | --------- | -------- | -------------------------------- |
| 1    | hungry_hippo_123 | Frank     | Shorius  | 2034F6E32958647FDFF75D265B455EBF |
```

Vi sier aldri nei-takk til litt god gammeldags brukerinformasjon, men det hadde jo vært fint å knekke den passord-hash'en også. Heldigvis finnes det verktøy for slikt!

```sh
hashid -m 2034F6E32958647FDFF75D265B455EBF
Analyzing '2034F6E32958647FDFF75D265B455EBF'
[+] MD2
[+] MD5 [Hashcat Mode: 0]
[+] MD4 [Hashcat Mode: 900]
[+] Double MD5 [Hashcat Mode: 2600]
[+] LM [Hashcat Mode: 3000]
[+] RIPEMD-128
...
```

Vi anntar dette er en MD5 hash og ser hva `hashcat` sier om det:

```sh
$ hashcat -m 0 -a 0 -r /usr/share/hashcat/rules/best64.rule 2034F6E32958647FDFF75D265B455EBF CTFtime/resources/wordlists/user/rockyou_fixed.txt

Hash:Password
2034f6e32958647fdff75d265b455ebf:secretpassword
```

Vi noterer oss det vi har funnet så langt, og så går vi videre til å se hva annet vi kan lære av dataene vi har per nå.

I mappen `/sdcard/../files` ligger det fire filer med navn som kan minne om base64-strenger. Jovisst, hvis vi kjører de igjennom en eller annen decoder ender vi opp med:

```
!aW1wb3J0YW50X2xpbmtz # important_links
!c2hvcHBpbmdfbGlzdA== # shopping_list
!dG9kbw==             # todo
!ZmxhZw==             # flag
```

Men tilgang til selve fil-innholdet ser ikke ut til å være rett frem.

Vi har også filene `ChatActivity.xml` og `VaultActivity.xml`. Dette ser ut som bruker-settings, trolig for Android app'en `lt3000.apk` som vi også kan finne.

Neste steg blir å se hva `lt3000` applikasjonen kan gi oss. Les mer i [2.1 Beslag](2_1_beslag.md)
