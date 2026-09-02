# Instant

Via de website is een APK te downloaden en deze kan via apktool uitgelezen
worden. 

In de data komen we het volgende tegen:
- Een subdomein genaamd `mywalletv1.instant.htb`
- Een autorisatie token `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpZCI6MSwicm9sZSI6IkFkbWluIiwid2FsSWQiOiJmMGVjYTZlNS03ODNhLTQ3MWQtOWQ4Zi0wMTYyY2JjOTAwZGIiLCJleHAiOjMzMjU5MzAzNjU2fQ.v0qyyAqDSgyoNFHU7MgRQcDA0Bw99_8AEXKGtWZ6rYA`
- Met `rg` vinden we de volgende endpoints:
	- `/api/v1/initiate/transaction`
	- `/api/v1/confirm/pin`
	- `/api/v1/register`
	- `/api/v1/view/profile`
	- `/api/v1/login`
- Verder zoeken en we vinden ook `swagger-ui.instant.htb`

# Swagger

Met swagger krijgen we een hele stapel extra endpoints waar we leuke dingen mee kunnen doen.

- Met `/view/logs` krijgen we een user terug `/home/shirohige`
- `/read/log` verwacht een log file name en hier zit een LFI in

We kunnen `id_rsa` uitlezen en hiermee zouden we SSH toegang moeten hebben. En
bam toegang en gelijk de user.flag

# Privesc

In de `instant.db` file die gebruikt wordt door de service die draait is een
password hash te vinden:
`pbkdf2:sha256:600000$I5bFyb0ZzD69pNX8$e9e4ea5c280e0766612295ab9bff32e5fa1de8f6cbb6586fab7ab7bc762bd978`

Hier kwam verder niks uit maar met verdere enumeratie komen we bij een
`/opt/backup/putty_session.dat` bestand uit. Met `sp_crack`
[link](https://github.com/RainbowCache/solar_putty_crack) kunnen we dit kraken.

Hiermee krijgen we een root login met wachtwoord `12**24nzC!r0c%q12`
