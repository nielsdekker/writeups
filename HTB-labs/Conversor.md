# Conversor

Na het aanmaken van een account op de webpagina lijkt het op een tool om mooie
NMAP output te krijgen. Na een beetje rondklikken is ook de source code te
downloaden van de applicatie. Mijn eerste ingeving is iets met XXE.

## Source code observaties

- [ ] In de source code staat ook een users.db

# PFF

Na wat cheaten geleerd dat xslt ook bestanden kan wegschrijven via een extensie.
[doclinkje](https://swisskyrepo.github.io/PayloadsAllTheThings/XSLT%20Injection/#write-files-with-exslt-extension)
Waarom XSLT dit kan via een extra extensie geen idee. Maar dit kunnen we
natuurlijk gebruiken om extra dingen weg te schrijven in de scripts folder die
we in de source code tegenkwamen.

Het volledige path is in ieder geval: `var/www/conversor.htb/scripts` (_uit een
eerdere error gehaald_).

En nu hebben we een revshell te pakken naar `www-data`. Hiermee hebben we ook
gelijk de users tabel met alle hashes te pakken. Hierin zijn twee users
interessant:

```
1|fismathack|5b5c3ac3a1c897c94caad48e6c71fdec
5|admin|21232f297a57a5a743894a0e4a801fc3
```

> fismathack staat als developer op de about pagina ;)

En hiermee hebben we een user te pakken met SSH

```
fismathack:Keepmesafeandwarm
```

# Privesc

We konden sudo uitvoeren op een executable waar een bekende CVE en pocs voor
waren

