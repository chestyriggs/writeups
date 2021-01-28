####### Matematikk ####################################################
# Jeg har laget en matematikk-tjeneste.
# Vil du prøve å logge inn på systemet?
#
# Adresse: http://math:7070
#
# PS: Det er 2(TO) flagg i denne oppgaven
#
# ----
#
# Flag 1 kan leses rett med:
# $ curl http://math:7070/flag_1
#
# Dette scriptet er for å lese ut Flag 2
#######################################################################

import requests
from pwn import *

URL = 'http://math:7070/flag_2'

HEADERS = {'User-Agent': 'Math Calculator (Python 3)'}
COOKIES = dict()

while True:
    res = requests.get(URL, headers=HEADERS, cookies=COOKIES)
    if len(COOKIES) < 1:
        COOKIES = res.cookies

    log.info(f"> HEADERS: \n{res.request.headers}")
    log.info(f"> Recieved: \n{res.text}")

# http://math:7070/challenge
