# 2.1 Beslag

Spoler alert - alle de tre flaggene i denne kategorien må finnes ved hjelp av LT3000-appen.

## "Happy Path"

Første order-of-business blir å forsøke kjøre applikasjonen på enten hardware eller emulator for å få en følelse av hvordan applikasjonen fungerer fra et bruker-perspektiv. For å gjøre dette bruker jeg `Android Studio` med emulator for så å laste over og installere APK'en på emulator-telefonen.

Det vil trolig være av interesse å ha et miljø så likt den annholdtes telefon som mulig. Derfor laster jeg også over alle de andre filene vi har i dump'en og gjennskaper hele filstrukturen.

Vi finner at applikasjonen har tre funksjoner; `Tuner`, `Chat` og `Vault`. Ingen av disse gir oss veldig mye foreløpig - det meste virker låst - men vi ser igjen filnavnene vi decoded [tidligere](README.md) inne i `Vault`.

## Vi bretter opp ermene

Med det mener jeg - påtide å decompile koden for å se hva som foregår under the hood. For å gjøre dette bruker jeg [APKLab til VSCode](https://marketplace.visualstudio.com/items?itemName=Surendrajat.apklab) som igjen bruker [jadx](https://github.com/skylot/jadx). Nå har vi muligheten til å lese koden i JAVA format, endre den (smali), recompile og generelt gjøre mye morsomt.

_OBS OBS OBS #1: Det har seg slik at om du kjører jadx med default instillinger så vil den ikke klare å decompile et par viktige biter med kode. Derfor satte jeg preferences->show inconsistent code til PÅ i jadx_gui og lagret koden på nytt. Det finnes et flag du kan sette for å få dette til i jadx også._

_OBS OBS OBS #2: Hvis du satt fast, og du forsøkte løse oppgavene på en emulator - vit at flere av sjekkene i koden SJEKKER om du sitter på en emulator. Alt ettersom, vil applikasjonen i enkelte tilfeller med vilje ikke gi deg rett svar._

## 2.1_beslag_1

Det finnes egentlig to måter å løse denne på. Men flagget her ligger i `Tuner` modulen til `lt3000` applikasjonen.

Den enkle metoden virker egentlig ikke være ment å fungere dersom du kjører på en emulator. Men jeg fikk det faktisk til å fungere fordi om.

1. Gjenskap miljøet på emulatoren - særlig `data.bin` filen må ligge i rett mappe.
2. Gå til `Tuner` og logg inn med `hungry_hippo_123:secretpassword` som vi fant tidligere
3. Aktiver, og velg kanal 7

Du vil da høre en stemme lese flagget til deg. Det vil også dukke opp en `data.mp3` sammen med `data.bin` filen du kan laste over og spille av på host-maskinen.

Løsningen ble funnet ved å tolke hva programmet gjør i filen `TunerActivity.java` (decompiled kode). Og en alternativ løsning ville være å skrive et program eksternt som gjør de samme stegene. Faktisk vil det vise seg at funksjone som senere brukes for å dekryptere filene i `Vault` mer eller mindre er de samme som brukes av `Tuner`:

```java
public static final byte[] SOME_SALT = "saltpaamaten".getBytes();
public static final char[] CHARR_ARRAY = "0123456789ABCDEF".toCharArray();

public static String a(byte[] bArr) {
    char[] cArr = new char[(bArr.length * 2)];
    for (int i = 0; i < bArr.length; i++) {
        int i2 = bArr[i] & 255;
        int i3 = i * 2;
        char[] cArr2 = CHARR_ARRAY;
        cArr[i3] = cArr2[i2 >>> 4];
        cArr[i3 + 1] = cArr2[i2 & 15];
    }
    return new String(cArr);
}

// Dette er hovedfunksjonen som kalles i TunerActivity.java og i VaultActivity.java
public static void b(File file, File file2, byte[] bArr) {
    try {
        // Instansierer et Ciper-objekt med ønsket transformasjon (se dokumentasjonen for javax.crypto.Cipher)
        Cipher instance = Cipher.getInstance("AES/CBC/PKCS5Padding");

        // Leser inn mål-filen som bytes
        byte[] readAllBytes = Files.readAllBytes(file.toPath());

        // Lager først et array som inneholder de første 16-bytene av `file`, og så et annet array som holder de resterende bytene i `file`.
        byte[] copyOfRange = Arrays.copyOfRange(readAllBytes, 0, 16);
        byte[] copyOfRange2 = Arrays.copyOfRange(readAllBytes, 16, readAllBytes.length);

        // Genererer en secret-key med funksjonen c() som da mottar passordet eller PIN-en.
        SecretKey c2 = c(bArr);
        if (c2 == null) {
            Log.e(a.class.getSimpleName(), "Failed to generate key");
        }

        // Initialiserer Cipher-objektet vårt i DECRYPT_MODE (2), med secret-keyen og en initialiserings vektor basert på de 16 første bytes fra `file`
        instance.init(2, c2, new IvParameterSpec(copyOfRange));

        // Fullfører dekrypteringen de gjenværende bytene fra `file` og putter resultatet inn i en variabel som bytes
        byte[] doFinal = instance.doFinal(copyOfRange2);

        // I `Tuner` sitt tilfelle så skrives resultatet ut igjen til en data.mp3 fil. Akkuratt dette steget ser litt annerledes ut i `Vault` men prinsippet er det samme
        Files.write(file2.toPath(), doFinal, StandardOpenOption.WRITE, StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING);
    } catch (Exception e) {
        e.printStackTrace();
    }
}

// Denne funksjonen er en sekvens av flere krypterings algoritmer som til slutt spytter ut en key
public static SecretKey c(byte[] bArr) {
    try {
        // Generer en MD5 hash av bArr (altså av passord eller PIN)
        MessageDigest instance = MessageDigest.getInstance("md5");
        instance.update(bArr);
        byte[] bArr2 = instance.digest();

        // Hash MD5 koden vår en gang til i SHA256, generer en Password-Based Encryption Key (PBE) ved hjelp av a() - og returner resultatet som en AES kryptert secret-key
        return new SecretKeySpec(SecretKeyFactory.getInstance("PBKDF2WithHmacSHA256").generateSecret(bArr2 != null ? new PBEKeySpec(a(bArr2).toCharArray(), SOME_SALT, 65536, 256) : null).getEncoded(), "AES");
    } catch (NoSuchAlgorithmException | InvalidKeySpecException e) {
        e.printStackTrace();
        return null;
    }
}
```

_Merk at utsnittet her er kun av selve koden som håndterer dekrypteringen for `Tuner`, men `Vault` fungerer i prinsippet likt. UX kode osv. er ikke tatt med._

Koden for å komme dit, samt verdiene på argumentene de sender til funksjonen er litt forskjellig mellom `Vault` og `Tuner`, men metoden er lik mellom de. Funksjonen `b(File file, File file2, byte[] bArr)` tar to filer hvor `file` filen man ønsker dekryptere og `file2` er enten `xtra` eller `data.bin` filene vi finner. Disse fungerer som nøkler for å kunne dekryptere `file`. I `Tuner` sitt tilfelle vil `bArr` være en byte-representasjon av passordet i passord-feltet, og i `Vault` sitt tilfelle inneholde verdien av PIN tumblerne.

## 2.1_beslag_2

_Disclaimer: Måten jeg løste denne på ser ikke ut til å fungere senere i CTF'en. Jeg vet ikke helt hvorfor, men heldigvis trengte jeg ikke tilgang til chatten senere og dermed hadde jeg ikke behov for å løse denne på den "rette" (tilsynelatende) måten._

Når vi åpner `Chat` får vi beskjed om å gå til `https://mobile.cybertalent.no` og så logge inn - med noen videre instrukser i appen på hva vi skal gjøre etterpå.

Vi kjører happy path igjen, og dersom vi bare prøver brukernavn:passord `test:test` får vi beskjed tilbake om at brukeren ikke eksisterer. Tester med `hungry_hippo_123:secretpassword` som vi fant tidligere, og vi kommer videre til `https://mobile.cybertalent.no/challenge`!

Vi får beskjed om at her er et tidspress, og vi kan se at challenge-strengen bare er synlig i 10 sekunder. Dermed kan vi ikke bare copy-paste mellom browser og emulatoren så vi må finne en alternativ metode.

Dersom vi bare putter inn `test` i input-feltet så ser vi at url'en bytter til
https://mobile.cybertalent.no/challenge?response=test#

Vi kan fyre opp OWASP ZAP eller BURP, for å ta en titt på HTTP requestene/responsene som går, og da ser vi at når vi prøver med 'test' så sender vi en request med følgende body:

```json
{ "response": "test" }
```

og får tilbake:

```json
{
  "corr": "b'7755a0a354aeab4257575875a0a95a5461a947ae54'",
  "error": "Bad response code"
}
```

Her sendes det rett og slett en string (`corr`) tilbake med hva koden SKULLE vært - slik at funksjonen har å sammenligne mot.

Vi smetter inn `7755a0a354aeab4257575875a0a95a5461a947ae54` (i mitt tilfelle) og får tilbake et token `07de8bf37bb5511229a9` som vi skal bruke i lt3000-appen med `/token 07de8bf37bb5511229a9`.

Når det er gjort har vi tilgang til appen - og øverst opper ser vi flagget!

_Alternativ metode:
Først, dersom du prøver å bruke `/response <challenge>` i Chat på en emulatoren så fikk du sansynligvis en streng tilbake som startet med `a1deadbeef`. Desverre for deg, så sjekker applikasjone her om du kjører på en emulator og virker sabotere deg dersom det er tilfellet. Her må det en kombinasjon av å lese kode og nettrafikk til for så å skrive et program som snakker med mobile.cybertalent.no-endepunktene programatisk._

Vi kan etter dette lese chatten, og vi noterer oss følgende:

```
- Chat-loggen går fra 02.12 til 08.12 når `hungry_hippo_123` arresteres. Angrepet skjer den 16.12
- `B4tCracK` virker være lederen, hvertfall utvikleren av appen
- `RADAR` har ansvaret for det taktiske
- `eireen89` virker være mest politisk engasjert
- `hungry_hippo_123` virker ikke å henge helt med på notene, eller være seriøs.

  02.12 - `B4tCracK`: it encrypts stuff six times, and might be slow
  => Kan bety at filer etc. må decryptes flere ganger!

  03.12 - `B4tCracK`: Sitting ducks on koldbjorn around here:
  https://www.google.com/maps/@-70.6927673,-0.2845782,14.8z
  => Datoen tatt i betraktning så er det her Kolbjørn er fryst inn i isen

  04.12 - `B4tCrack`: I have deviced a cunning method to book flight tickets anonymously.
  Our names will not show up anywhere except in the Sea the World systems.
  => Bekrefter mistanken mot Sea the World, og også at vi kan finne navnene deres her!
  => Ordene `HUMMINGBIRD` og `OPERATION EXPLODING HUMMINGBIRD` blir nevnt

  07.12 - `hungry_hippo_123`: btw can't find my booking reference
  `B4tCrack`: WJWQX
  => Da kan vi knytte hippo til en billet dersom vi kommer inn i StW systemet
  => Det hintes også litt senere til at systemet til StW er `some old crap that system`

  `B4tCrack`: for future reference; your username to the backdoor is the first three letters
  of your first name followed by the length of your username here and ending with your
  last name. The capital letters of your name stays capital.
  => Heisann! En bakdør i StW sine systemer mon tro?
  => `hungry_hippo_123` blir da `Fra16Shorius` om navnet hans i users.db stemmer

```

## 2.1_beslag_3

_Disclaimer: Denne oppgaven var en nøtt, og jeg endte opp med å sammarbeide med `mutex` for å finne rett rabit-hole, samt diskutere oss frem til rett løsning. Sammen dunket vi våre fellers hoder i veggen helt til vi forstod vi måtte sette rett flagg i Jadx decompileren slik at vi fikk se **hele** koden. Han skrev programmene vi trengte for å knekke pin og filer utrolig mye fortere enn meg, så all ære for at jeg kom så langt som jeg gjorde i år innen tidsfristen går til `mutex`._

Fra `VaultActivity.xml` har vi Frank Shorius sin `pin_hash`. Vi skjønner ganske fort at PIN koden består av 4 siffer, og vi vet da hvilken hash disse fire siffrene skal ende opp i:

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <boolean name="locked" value="true" />
    <string name="pin_hash">BSzN6uy2dDcolImpMKxffg==</string>
</map>
```

Metoden for å generere denne hashen kunne vi finne i koden vi nå hadde tilgang til, og da var det egentlig bare å skrive en applikasjon som bruteforcet tall mellom 0000 til 9999 helt til hash===pin_hash. `Vault` viste seg også å laste ned en binærfil fra https://mobile.cybertalent.no/xtra. Fra denne kom vi til å trenge funksjonen getData() som først var tilgjengelig etter at filen var "descramblet".

```java
package com.example.extra;

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;

public class extra {
    private static final String SALT = "y_so_salty";

    public static byte[] getData(String str) {
        StringBuilder sb = new StringBuilder();
        sb.append(str);
        sb.append(SALT);
        byte[] bArr = new byte[0];
        byte[] bArr2 = new byte[0];
        try {
            MessageDigest instance = MessageDigest.getInstance("md5");
            instance.update(sb.toString().getBytes());
            bArr = instance.digest();
            MessageDigest instance2 = MessageDigest.getInstance("sha256");
            instance2.update(sb.toString().getBytes());
            bArr2 = instance2.digest();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        }
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        try {
            byteArrayOutputStream.write(bArr);
            byteArrayOutputStream.write(bArr2);
        } catch (IOException e2) {
            e2.printStackTrace();
        }
        return byteArrayOutputStream.toByteArray();
    }
}
```

Da hadde vi etterhvert det vi trengte for å knekke PIN-Koden, og nå kunne vi muligens bare brukt denne direkte i Android applikasjonen for å dekryptere vault-filene. Men vi valgte å kombinere de forskjellige bitene med kode vi nå hadde funnet fra `xtra`, `VaultActivity.java` og koden vist i `beslag_1` til å lage et eget program som knakk disse.

Til slutt ender man opp med følgende data ut fra filene:

```md
## flag:

d02582..................................

## important_links:

Hidden service URL:
http://fileserver/files/onion_name.txt

Scada client binary:
http://fileserver/files/082391170510954df0c28af1ebb9380a
Rembember this only works on the internal network, not from the outside.

## shopping_list:

1 Milk
1 Bread
6 Eggs
1000 rounds 7.76mm
10 HE grenades
4 Yoghurt
1 Butter

## todo:

Buy xmas presents
Find an xmas tree
Order airline tickets
```
