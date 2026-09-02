---
tags:
  - web
  - cve-2025-54309
  - linux
  - privesc
  - api
---

Met fuzzing een `ftp.soulmate.htb`  subdomein gevonden. Hier draait een crush ftp server op en deze is vatbaar voor CVE-2025-54309. Hiermee een admin user kunnen aanmaken zodat ik bij de data op de server kan. 

In de UI een `revshell.php` geupload waarmee ik een reverse shell te pakken heb voor `www-data`. Hier nog niet heel veel kunnen vinden. Wel een aantal dingen buit kunnen maken qua hashes maar nog niks concreets.

In de config van de standaard website vind ik wel de volgende gegevens:

```
admin:Crush4dmin990
```

Hiermee kunnen we inloggen op de hoofdwebsite.

# Ben
Na lang zoeken staat er schijnbaar een wachtwoord in een random erlang script voor de ben user.

```
ben:HouseH0ldings998
```

# Privesc
Al eerder een sftp server tegengekomen op poort 2222 die SSH gebruikt. Hier is schijnbaar een exploit voor en eigenlijk deze writeup gevolgd: [writeup](medium.com/@RosanaFS/erlang-otp-ssh-cve-2025-32433-tryhackme-e410df5f1b53)
