# WifineticTwo

Een eerste call naar het ip geeft een error. Er draait dus niks op poort `80`.
Nmap scan later en het lijkt erop dat poort `8080` wel open is en een webservice
draait.

Navigeren naar deze poort geeft een login pagina voor iets dat `OpenPLC` heet.
Nog geen idee wat dit is. Snelle google later en dit lijkt een stukje legacy
software dat niet meer gesupport wordt.

De default credentials `openplc:openplc` lijken te werken. Daarnaast is er een
RCE exploit: https://www.cvedetails.com/cve/CVE-2021-31630/ die we
waarschijnlijk kunnen gebruiken.

Via die CVE aan een C programma gekomen die we kunnen draaien in openplc en dit
geeft ons een shell, zie ook `shell.c`.

We zitten hier in een PLC/VM/ding/iets. In ieder geval is er in de `/root/` een
`user.txt` te vinden.

> Meerdere dingen geprobeerd rondom container escape en ik kom nergens

`ifconfig` Toont een wlan0 interface en de box heet `wifinetictwo`. Waarom heeft
mijn box een wifi connectie? Dit is interessant om uit te zoeken.

> Een hele tijd en vele resets later

Het ging toch wel om het wifi network. Ik heb via een proxy reaver geinstalleerd
maar dit leverde niks om. Een `oneshot` script van github geeft wel resultaat en
hiermee weten we gelijk het wachtwoord voor het wifi netwerk: `NoWWEDoKnowWhaTisReal123!`

Na het opzetten van de connectie kunnen we inloggen via ssh op de andere host op
dit netwerk. Dit geeft ons de root flag

```bash
wpa_passphrase "NoWWEDoKnowWhaTisReal123!" > conf
wpa_supplicant -B -i wlan0 -c ./conf

# Haal ip op
ifconfig wlan0 192.168.1.69 netmask 255.255.255.0

# Zoek andere hosts
seq 1 254 | xargs ping -c 1 {} | grep "bytes from"
```

