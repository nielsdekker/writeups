# Facts

Rond klikken op de pagina toont niet echt dingen om te exploiteren, er zijn wel
comments maar het lijkt er niet op dat ik zelf comments kan plaatsen. Met
fuzzing en nmap vind ik wel het volgende:

- Poort 54321 die open staat
- Een `/admin` met daarop `Camaleon CMS`

# Camaleon

Voor Camaleon kan ik een account aanmaken en registreren. Dit is wel een account
zonder enige admin rechten. Er is wel CVE-2025-2304 die misbruikt kan worden om
admin te worden.

Eenmaal admin zie ik dat er een filestorage is ingesteld via S3. Wanneer ik de
secret en key hiervan gebruik kan ik bij de bucket (_op poort 54321 ;)_) en hier
is ook een `/internal` folder met daarin een SSH key.

# SSH

De SSH key is niet de gouden oplossing die ik hoopte, ik weet namelijk niet
welke user bij deze key hoort. Uiteindelijk kom ik terecht op CVE-2024-46987
voor Camaleon die opgelost zou moeten zijn in V2.8.2 maar gek genoeg wel werkt.
Hiermee kan ik `/etc/passwd` uitlezen en nu heb ik de user die ik zoek. Het is
of:

- `william`
- `trivia`

Met proberen blijkt de key voor `trivia` te zijn maar de key heeft wel een
password :(

# JohnTheRipper

Via `john` kan ik ook SSH keys brute-forcen en dit blijkt de oplossing te zijn.
Ik vind wachtwoord: `dragonballz` en hiermee heb ik SSH toegang op de `trivia`
user.

Deze user kan ook bij de `william` user en nu heb ik de user flag te pakken!

# Privesc

Met `sudo -l` zie ik dat ik `facter` kan uitvoeren. Op [gtfobins](https://gtfobins.org/gtfobins/facter/)
zie ik dat ik via `facter` als root een ruby bestand kan uitvoeren.

Kleine command later en we hebben root.

```ruby
exec("/bin/bash")
```
