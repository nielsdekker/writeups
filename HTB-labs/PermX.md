# PermX

`http://permx.htb` Ziet er uit als een elearning website. Snel scrollen over de
website toont niet echt onderdelen waar interactie mogelijk is. Er zijn wel
forms maar dit lijkt naar niks te gaan.

# FFUF

Met vhost fuzzing vinden we een extra subdomain genaamd `lms.permx.htb`. Deze
pagina toont een simpele login form en iets genaamd `Chamilo`. Dit lijkt op een
opensource LMS (_learning management system_) oplossing.

Met wat rondzoeken vinden we een [RCE
CVE](https://nvd.nist.gov/vuln/detail/CVE-2023-4220). Omdat we nog niet weten
welke versie van chamilo draait is er geen garantie dat het werkt maar waard om
te proberen.

# CVE-2023-4220

Met een de volgende
[POC](https://github.com/Ziad-Sakr/Chamilo-CVE-2023-4220-Exploit) hebben we een
reverse shell te pakken en toegang tot de server.

- We krijgen een shell voor `www-data`
- Website is gehost in `/var/www/chamilo`
- Zo te zien zijn er twee users `root` en `mtz`

# Configuration.php

Op de server is ook een `configuration.php` bestand te vinden met daarin wat
interessante dingen. Zoals de [db user](./users.md). Het wachtwoord van deze
user komt overeen met de `mtz` user.

# MTZ

De mtz user geeft ons de user flag. Met `sudo -l` kunnen we ook zien dat deze
user, zonder wachtwoord, `/opt/acl.sh`. Met dit script kunnen we alleen in de
home folder extra permissies toevoegen. 

De truk hier is om een `symlink` te maken naar `/etc/passwd`. Waarna we een
eigen root user kunnen aanmaken waarmee we kunnen inloggen.

```bash
ln -s /etc/passwd pass

sudo /opt/acl.sh mtz rwx pass
```
