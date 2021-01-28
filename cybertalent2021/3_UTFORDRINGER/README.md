# 3_Utfordringer

## 3.1_lett

### 3.1.1_clmystery

Ta en titt på åstedet:

```sh
$ grep "CLUE" crimescene
CLUE: Footage from an ATM security camera is blurry but shows that the perpetrator is a tall male, at least 6'.
CLUE: Found a wallet believed to belong to the killer: no ID, just loose change, and membership cards for AAA, Delta SkyMiles, the local library, and the Museum of Bash History. The cards are totally untraceable and have no name, for some reason.
CLUE: Questioned the barista at the local coffee shop. He said a woman left right before they heard the shots. The name on her latte was Annabel, she had blond spiky hair and a New Zealand accent.
```

1. Hvem er denne `Annabel`?

Finner ut at dette er `Oluwasegun Annabel` som ikke er over 6 fot. Sansynligvis ikke morderen, og virker heller ikke til å være vitnet fra kaféen.

2. Kryssreferer alle medlemskort med folk over 6 fot

Først lager vi en liste over alle som er over 6 fot høy:

```sh
$ grep -i -B 1 "height: 6." vehicles >> people-over-6-inches-NAMES.txt
```

Kryssrefererer people-over-6-inches-NAMES.txt mot medlemslistene:

```sh
cat people-over-6-inches-NAMES.txt| while read line; do grep "$line" memberships/AAA >> members_aaa.txt; done

cat members_aaa.txt| while read line; do grep "$line" memberships/Delta_SkyMiles >> members_delta.txt; done

cat members_delta.txt| while read line; do grep "$line" memberships/Terminal_City_Library >> members_library.txt; done

cat members_library.txt| while read line; do grep "$line" memberships/Museum_of_Bash_History >> members_bash.txt; done
```

Relativt få mistenkte i members_bash.txt, så vi gjetter oss litt frem og ender til slutt opp med flagget O:)

---

### 3.1.9_secret

Denne ligger i bildet:

```sh
$ cat /dev/shm/.secret
b4c7c234fe......................
```

FLAGG: `b4c7c234fe......................`
