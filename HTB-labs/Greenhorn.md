# GreenHorn 

Naast een hele grote hoeveelheid bestanden die andere achtergelaten hebben
vinden we een `/login.php`. Daarnaast is er ook een grote hoeveelheid poorten
open.

# :3000

Op poort 3000 vinden we een git repo met daarin de gehoste cms voor de mainpage,
hier vinden we in de settings ook een password hash

```
<?php
$ww = 'd5443aef1b64544f3685bf112f6c405218c573c7279a831b1fe9612e3a4d770486743c5580556c0d838b51749de15530f87fb793afdcc689b6b39024d7790163';
?>
```

Met hashcat krijgen we hier het wachtwoord `iloveyou1` uit. Hiermee kunnen we
inloggen op de CMS.

# pluck cms

Na wat rondzoeken komen we erachter dat er een CVE is voor pluck v4.7.18,
namelijk `CVE-2023-50564`. In het kort komt het er op neer dat je via de modules
upload php files kan uploaden.

1 Revshell later en we hebben toegang tot de server.

# Junior

Op de server vinden we 3 users die interessant zijn:

- root
- www-data (hiermee krijgen we de revshell)
- junior

Via `su junior` en het eerder genoemde wachtwoord hebben we user access te
pakken. Na het toevoegen van onze SSH-key hebben we ook gelijk een makkelijke
manier om in te loggen mocht de shell om wat voor reden dan ook crashen.

# Privesc

In de `~` van junior vinden we een PDF over `OpenVAS`. Wat interessant is is dat
hier een obfuscated password in staat. Met
[depix](https://github.com/spipm/Depix) kunnen we dit deubfuscaten en de
daadwerkelijke tekst krijgen. 

Uiteindelijk kunnen we als root inloggen met
`sidefromsidetheothersidesidefromsidetheotherside`
