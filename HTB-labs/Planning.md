# Planning

Na wat fuzzing de grafana.planning.htb subdomein gevonden. Hier kon ik inloggen
met de aangeleverde credentials. Namelijk `admin:0D5oT70Fq13EvB5r`. Er draait
hier grafana v11 die vatbaar is voor een rce exploit. Snelle google later en heb
een [pocje](https://github.com/nollium/CVE-2024-9264) gevonden.

Hiermee komen we wel op een docker container uit maar in de configs/env vinden
we een aantal secrets. Namelijk:
```
# used for signing
secret_key = SW2YcwTIb9zpOOhoPsMm

# in env
GF_SECURITY_ADMIN_PASSWORD=RioTecRANDEntANT!
GF_SECURITY_ADMIN_USER=enzo
```

# privesc

In `/opt/crontabs` komen we een script tegen met daarin een password
`P4ssw0rdS0pRi0T3c`, daarnaast draait er ook op port 8000 iets met basic-auth
ervoor. 

Met een beetje gokken hier kunnen inloggen als `root:P4ssw0rdS0pRi0T3c`. Dit is
een admin omgeving voor crontabs en zo te zien kunnen we hier ook een command
toevoegen. Tijd voor een revshell.
