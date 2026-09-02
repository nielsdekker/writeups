# Sea

Er is een contact formulier beschikbaar op `/contact.php`. Hier kan ook een website ingevuld worden en er wordt dan een request gemaakt naar deze url.

```
Ncat: Connection from 10.10.11.28:54736.
GET / HTTP/1.1
Host: 10.10.16.40:9001
Connection: keep-alive
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) HeadlessChrome/117.0.5938.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate
```

Na wat verder zoeken achterhaald dat dit om wonderCMS gaat waar een exploit in zit die uit te buiten is. Gelukkig is hier al een [POC](https://github.com/prodigiousMind/CVE-2023-41425) voor.

# Shell

Volgens mij heb ik de revshell die iemand anders geupload heeft hergebruikt maar
dat terzijde. Er is een foothold en in de data vinden we een paar interessante
dingen. Namelijk een `database.json` met het wachtwoord `mychemicalromance`.
Hiermee kunnen we inloggen op `sea.htb/loginURL`.

We vinden in `/etc/passwd` de volgende users:

- `root`
- `www-data`
- `amay` (_heeft ook `mychemicalromance` als wachtwoord_)
- `geo`
