# BoardLight

In eerste instantie lijkt dit een vrij simpele company page met weinig
interactie. Er is wel een verwijzing naar `portfolio.php` maar dat lijkt nergens
op uit te komen. Wel zien we dat ipv het _ip_ de pagina ook als `board.htb`
wordt gerefereerd. 

Een `ffuf` later en we vinden ook een `crm.board.htb` subpagina die een
verouderde versie van dolibarr draait. 

Hier is een exploit voor die we kunnen uitbuiten alleen moeten we eerst
inloggen. Gelukkig werkt `admin:admin` gewoon. [Volgen van deze blog en we
hebben RCE](https://www.swascan.com/security-advisory-dolibarr-17-0-0/).

Op de server komen we een aantal dingen tegen:
- Een mysql server
- Via etc/passwd weten we dat er een `larissa` user is
- In `html/board.htb/*` zien we geen `application.php`. Dit was dus een red-herring
- In `html/crm.board.htb/htdocs/conf/*` vinden we config bestanden met daarin ook de mysql gegevens.

In de database vinden we wel een password hash maar deze kraken leid tot niks.
Wat wel werkt is het DB wachtwoord gebruiken voor de `larissa` gebruiker en nu
hebben we SSH access met: `larissa:serverfun2$2023!!`

## Privesc

Linpeas gaf een aantal interessante bestanden met een SUID permissie gezet die
niets "standaard" zijn in een linux omgeving. Het ging hier om de
`enlightenment` desktop omgeving en hier is een bekende CVE voor:
[github](https://github.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit).
