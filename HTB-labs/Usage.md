# Usage

Op poort 80 is een server te vinden (vhost `usage.htb`) waarop een login pagina
te zien is. We hebben hier meerdere links naar meerdere pagina's en we kunnen
een account aanmaken.

```
uname:  foo@test.test
ww:     foofoo
```

Jammer genoeg kunnen we hiermee niet inloggen op de `admin.usage.htb`

Cookie is wel een base64 ding (_geformat_):

```
# session
{
    "iv":"3eivgyrd25v2o5mfQa1sWg==",
    "value":"knvthX6+wzK4Zimn50/j4A7upc6T4r9nfsPL81oZtGGQVtqnlWiLF67K5nHJaV7M8JRTSJG+hgDRWcoS1bgfRujY16JTVNTyYYz721lRk/9e40AC4u0ntzmttYDzI4cA",
    "mac":"4730f94a8b3b20f8758e8caf72412e250774480bddaee00bc2ca1814807bb0da",
    "tag":""
}

# xsrf
{
    "iv": "4jzNScp/rUovSG8r4XlFfw==",
    "value":"ye+XvCdKn0I/0hwxSClbvAtIDeCTTGDDDQG/e6c4xyzGdCPd4yhddlNg1Rw+/UQ5vMfa7lc1O0gh+fMivkUPMz4VFsYXkG9CJ4RZR2ZCU6vCM5UFou0dP2jBDfAnpgy9",
    "mac":"ab3971f635697277bd4cefebd71bf59c9b928cd023f26bda246691ca721ea905",
    "tag":""
}
```

Op de `reset password` pagina kunnen we users enumeraten. We krijgen namelijk
terug of een user bestaat of niet.

# SQLI

Verder testen en het lijkt erop dat sqli mogelijk is via de _forgot password_
pagina. Een call naar `foo@test.test' ORDER BY 1 -- ; ` geeft namelijk terug dat
de email gevonden is en dus klopt. Nu is het wel zo dat we alleen de input terug
krijgen en niet de daadwerkelijke data. Het is dus waarschijnlijk een
blind-sqli.

## YOLO SQLI

Met blinde SQLi kan ik al een heel eind komen. Achterhaalde data is te vinden in
de sqlmap.md

- De select is op acht kolommen, via union achterhaald
- De tabel heet `users` en de volgende velden bestaan:
  - `id`
  - `name`
  - `email`
  - `password`
  - `remember_token` (nergens ingevuld)
  - `verified_at`
  - `created_at`
  - `updated_at`

Gevonden users:

> Via `foo@test.test' AND EXISTS (SELECT email FROM users WHERE email LIKE 'n%') -- ; ` kan ik over mogelijke emails loopen

- `raj@raj.com:$2y$10$7almtteyfrvd8rnyep/ck.bsfkfxfsltplkyqqsp/tt7x1wapjt4.` <-- De box is gemaakt door iemand genaamd raj
- `raj@usage.htb:$2y$10$rbncgxpwp1hspo1gqx4upo.pdg1nszoi/uhwhvfhddfdfo9vmdjsa`
- `foo@test.test:`, hieronder mogelijk password hashes, dit verschilt tussen
  restarts dus hier zit iets van een salt overheen
  - `foofoo:$2y$10$safhbfzgriuwxqlg0ooi4.f0tiiyfc44s7tgj5uhjsxwlaohed71q`
  - `foofoo:$2y$10$vnhtjpyfhfa3zwykf7j/p.ew/zgkjda0ekfuhy59zw/jkhrorg4rs`

Gevonden tables:

- `admin_menu` (id, order, icon, permission, parent_id, uri, title, created_at, updated_at)
- `admin_operation_log` (id, ip, input, method, path, user_id, created_at, updated_at)
- `admin_permissions`
- `admin_roles`
- `admin_role_menu`
- `admin_role_permissions`
- `admin_role_users`
  - `user_id`
  - `role_id`
  - `updated_at`
  - `created_at`
- `admin_users` `SELECT table_name FROM information_schema.tables WHERE table_name LIKE 'admin%'`
- `admin_users_permissions`
- `blog` (id, title, content, created_at, updated_at)
- `failed_jobs`
- `global_status`
- `global_variables`
- `migrations`
- `password_reset_tokens` (email, token, created_at)
- `persisted_variables`
- `personal_access_tokens` (id, name, abilities, tokenable_id, tokenable_type, created_at, updated_at)
- `processlist`
- `session_account_connect_attrs`
- `session_status`
- `session_variables`
- `users` (id, name, email, password, remember_token, verified_at, updated_at, created_at)
- `variables_info`

## raj@raj.com

Het volgende record bestaat, raj@raj.com heeft user_id `1`

```
admin_role_users
user_id | role_id | created_at | updated_at
1       | 1
```

Wij hebben user_id = 7

# Cracking

Na veel te veel tijd besteed te hebben aan van alles en nog wat maar besloten om
een `hashcat` (nog een keer) aan te zetten op de achtergrond om daarna nog veel
meer tijd te stoppen in andere dingen.

`hashcat` Vond een wachtwoord en nu kunnen in de admin omgeving met
`admin:whatever1`

# Admin dashboard

Niet heel spannend maar er is een file upload mogelijk.... You're thinking what
i'm thinking. Uploaden van een php script als plaatje ging heel makkelijk. Kwam
terecht in `http://admin.usage.htb/uploads/images/rce2.php`. Kreeg in het eerste
geval wel een error dat het alleen plaatjes uploaden mogelijk was maar via
_continue editing_ knop trad dit niet op.

Ik heb SSH keys dus laat ik die maar toevoegen. Staan in SSH.md

# Privesc

We loggen in als de `dash` user maar er is nog een user genaamd `xander`. Dit
kan interessant zijn om te checken.

In de user folder zien we ook een `.monitrc` waar info instaat om met de remote
tool van monit te connecten `usage.htb:2812` met gegevens `admin:3nc0d3d_pa$$w0rd`.
Hiervoor moet er wel een SSH tunnel opgezet worden.

Nog beter is dat dat wachtwoord ook voor de `xander` user is. Wat ook gelijk
betekend dat we een mooie checkpoint hebben om later in te loggen. Xander is
lekker bezig en mag `/usr/bin/usage_management` als root uitvoeren.

Deze tool kan zo te zien het een en ander uitvoeren waaronder alles in de
`project_admin` folder. Kunnen dit natuurlijk misbruiken om de root flag te
pakken te krijgen.

Na wat uitpluizen met ghidra de commando's achterhaald die uitgevoerd worden:

- `/usr/bin/7za a /var/backups/project.zip -tzip -snl -mmt -- *` voor de backup
- `/usr/bin/mysqldump -A > /var/backups/mysql_backup.sql` voor mysql
- `echo "updated"` voor het resetten van het wachtwoord

Die laatste is dus onzin maar de eerste heeft een interessant artikel op
hacktricks: `https://book.hacktricks.xyz/linux-hardening/privilege-escalation/wildcards-spare-tricks#id-7z`.

Hiermee uiteindelijk de root flag te pakken gekregen.
