# Runner

## Nmap

Nmap vindt naast de standaard poorten ook een poort 8000. Dit wordt gedetecteerd
als `nagios-nsca`. Een http call geeft puur een 404 terug. Wat betekend dat er
wel iets van een http server draait.

## Company website

Met ffuf en rondklikken vind ik hier niks interessants. Er zijn in ieder geval
geen overduidelijke hints dat hier meer draait dan verwacht.

## Nagios

Dit lijkt op een _monitoring-agent_ voor linux. Lijkt ook op een mooie om verder
te onderzoeken. Puur spelend met wat mogelijke endpoints kom ik al de volgende
tegen:

- `/health` Lijkt op een standaard health endpoint
- `/version` Geeft `0.0.0-src` terug.

## Speciale wordlist

Met nagios kwam ik nergens, op aanraden een custom wordlist gemaakt voor sub
domain enumeratie en daarmee een hit gekregen op `teamcity.runner.htb`.

## Teamcity

[Om te
beginnen](https://blog.jetbrains.com/teamcity/2024/03/additional-critical-security-issues-affecting-teamcity-on-premises-cve-2024-27198-and-cve-2024-27199-update-to-2023-11-4-now/)
kwam ik dit tegen. En ook een [linkje naar
rapid7](https://www.rapid7.com/blog/post/2024/03/04/etr-cve-2024-27198-and-cve-2024-27199-jetbrains-teamcity-multiple-authentication-bypass-vulnerabilities-fixed/).

CVE-2024-27198 geeft volle toegang tot alles waaronder RCE. Wat een feest.

Kleine exploit later en we hebben toegang tot het admin dashboard. Daarnaast is
er ook een metasploit dingetje hiervoor om het te automatiseren. Kort komt er op
neer dat we toegang tot het admin dashboard krijgen en daarna een plugin
uploaden die een reverse shell neerzet.

## De teamcity container

Na exploiteren komen we in de teamcity container terecht. In de
`/data/teamcity_server/datadir/system` vinden we een `hsqldb` dabase met daarin
de users die gebruikt zijn voor de teamcity omgeving. Hierbij ook gelijk de
password hashes buitgemaakt. 

Van de _matthew_ user het wachtwoord kunnen kraken: `piper123`. Hiermee geen SSH
toegang :(. Inloggen als de matthew user in teamcity geeft ons niet echt extra
info die we nog niet hadden.

## id_rsa

Met matthew user kwam ik niet veel verder maar in de data folder was ook een
`id_rsa` te vinden. 

> Note to self, meer en betere enumeratie voor bestanden

Met deze key kan ik inloggen als `john` via ssh. Hierbij ook gelijk de `user` flag te pakken.

## Privesc

Met netstat een aantal open poorten gevonden waaronder poort `9443`, na het
opzetten van een port-forward kom ik op een inlog voor portainer terecht waar ik
in kan loggen met de eerder ontdekte gegevens van Matthew.

In portainer vind ik twee containers, de eerste draait _teamcity_ met een
standaard user. De tweede, `test_pe` draait als root. Als ik hier dus een volume
aan kan hangen dan heb ik privesc te pakken. 

Uiteindelijk een volume aangemaakt met de opties:
- type: none
- o: bind
- device: /

Een extra container aangemaakt waar ik dit volume in heb gemapped en we hebben
toegang tot alle bestanden in root en dus ook de root flag.

> Note voor volgende keer, HTB boxes ruimen `/tmp` en `/dev/shm` automagisch op.
> Daar dus bestanden in pleuren is echt vragen om issues met het aanmaken van
> volumes
