# Eerste indruk

Het is een tool waarmee markdown bestanden geupload kunnen worden en daarnaast
ook gedeeld kunnen worden. De volgende dingen zijn dus mogelijk:

- SSTI via een markdown
- XSS via een geuploade markdown
- LFI via de share link
- Er is een contact form die volgens de _about_ page altijd door een admin wordt gecheckt.
- De main page gebruikt een `?page=` param. Mogelijk kunnen we hiermee en de upload iets moois doen
- Er is ook een `messages` page die niks toont?

Via `ffuf` vinden we ook een `statistics.alert.htb` subdomain waar een basicauth voor hangt.

# XSS

We kunnen zonder problemen script tags embedden in de markdown die dan
uitgevoerd worden, er is dus een XSS mogelijkheid als we dit via de share link
delen.

Dit lijkt ook de attack vector te zijn. Als je via de contact page een link
meegeeft dan wordt deze door de admin geopend.

# Uploads

Er is een `/uploads/...` folder waar geuploade bestanden in eindigen. Via de
share link kunnen we de bestandsnaam achterhalen.

Het lijkt er verder ook op dat geuploade bestanden altijd gerenamed worden naar
`.....md` wat het lastiger maakt om dit via de `?page=` param te misbruiken.

# Cookie stealing

Zo te zien is er geen local/session storage of cookie beschikbaar via
javascript. Waarschijnlijk is dit omdat de waarde op `httponly` staat. Maar wat
we wel kunnen doen is calls maken naar data waar we eerst niet bij konden. Zoals
de eerder gevonden `messages` pagina.

Dit geeft een stukje html terug in de trant van:

```html
<a href='messages.php?file=2024.txt'>2024.txt<a>
```

Met verder testen is het mogelijk om via de geuploade markdown als admin een
call te maken naar `messages.php?file=../../../etc/passwd` en hiermee hebben we
een LFI te pakken.

Nu kunnen we een payload maken waarmee we alle server code kunnen ophalen zodat
we wat gerichter naar exploits kunnen zoeken.

# Apache.conf

Met de code kwam ik niet heel veel verder maar na het uitlezen van verschillende
bestanden, waaronder de apache.conf (_hiervoor een LFI wordlist gebruikt_) kwam
ik het pad tegen voor het `.htpasswd` bestand die de `statistics.alert.htb`
pagina beschermde.

Hierin staat:

```
albert:$apr1$bMoRBJOg$igG8WBtQ1xYDTQdLjSWZQ/
```

Nu weet ik van de `/etc/passwd` die ik heb opgehaald dat er een `albert` user
aanwezig is. Met een beetje mazzel is hier dus een password reuse.

En een snelle hashcat later en we hebben het wachtwoord `manchesterunited`

# SSH

Met die gegevens hebben we ook SSH access op de server! Voor privesc is er zo
snel geen al te duidelijke exploit die we kunnen doen, geen sudo rechten en
dergelijke. Wat er wel is is dat we in de `management` group zitten. Dit is niet
een group die ik zo snel herken.

Met `find / -group management 2>/dev/null` kunnen we bestanden zoeken waar we
bij mogen en hierin zit ook de `/opt/website-monitor`. Hier mogen we aan de
configuratie bestanden zitten. De rest hier is onder beheer van de root user dus
dit is een interessante om naar te kijken.

# Website-monitor

Deze site draait op poort 8080 en is alleen te benaderen via localhost. Met ssh
kunnen we een port forward opzetten en de data bekijken. Omdat we in de config
folder spul kunnen wegschrijven kunnen we ook onze eigen `revshell.php` hierin
plaatsen. Een call naar `localhost:8080/config/revshell.php` en we hebben root
access.
