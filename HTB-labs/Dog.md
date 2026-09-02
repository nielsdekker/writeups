# Dog

Op het eerste gezicht een vrij simpele webpagina met weinig interactiviteit. Wat
wel aanwezig is is een login pagina. Met `ffuf` en een username wordlist kunnen
we snel een user achterhalen die bestaat. Namelijk `john`.

> _De pagina geeft aan of een username valide is of niet_

Hierna worden we wel geblokt omdat er teveel pogingen zijn gedaan. Het is dus
goed mogelijk dat er nog meer namen zijn.

Via de post history komen we ook achter de naam: `dogBackDropSystem`. Wat
schijnbaar ook een valide user is.

## Brute force password

Het lijkt er op dat er een ip-blok zit op het brute-forcen van een wachtwoord.
Dit is dus geen goede attack-vector. (_Het is bijna nooit een attack vector
met HTB_).

# FFUF common.txt

Een wordlist die ik te weinig gebruik is de `common.txt`, het lijkt er op dat er
een hele stapel open staat qua git en file folders.

Hierin vinden we twee dingen die interessant zijn, een user genaamd
`tiffany@dog.htb` en een wachtwoord voor de database namelijk
`BackDropJ2024DS2024`. Dit is ook het wachtwoord voor `tiffany@dog.htb` als we
proberen in te loggen op de website en nu hebben we dus admin access.

```
tiffany@dog.htb:BackDropJ2024DS2024
```

# RCE

Met de login hebben we een RCE te pakken waarbij we een eigen shell kunnen
uploaden (_een bog-standard "upload plugin" met een php revshell erin_). Hierbij
loggen we in als `www-data` die niet heel veel rechten heeft. Wel kunnen we bij
de database waar we een mooie collectie aan usernames en passwords hebben. Tijd
voor hashcat.

Terwijl hashcat draait gekeken of het wachtwoord voor meer werd hergebruikt en
het lijkt er op dat `johncusack` ook hetzelfde `BackDropJ2024DS2024` wachtwoord
gebruikt.

```bash
ssh johncusack@dog.htb
```
# Privesc

De `johncusack` user mag `bee` met root rechten uitvoeren. Nu heeft deze
applicatie ook de mogelijkheid om een willekeurig PHP script uit te voeren. We
kunnen dus onze rce upload die we eerder hebben gebruikt hiervoor gebruiken om
een root revshell te krijgen.
