# Keeper

```
\u276f sudo nmap -sC 10.10.11.227 -oA sync -Pn -n --disable-arp-ping
[sudo] password for freckles:
Sorry, try again.
[sudo] password for freckles:
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-22 23:47 CEST
Nmap scan report for 10.10.11.227
Host is up (0.014s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
| ssh-hostkey:
|_  256 3539d439404b1f6186dd7c37bb4b989e (ECDSA)
80/tcp open  http
|_http-title: Site doesn't have a title (text/html).

Nmap done: 1 IP address (1 host up) scanned in 1.37 seconds
```

On port 80 we find a redirect to `tickets.keeper.htb`. This page hosts version
`4.4.4` of _request tracker_. On this page a login is shown and login in with
admin:admin and other obvious attempts results in nothing.

# SQLmap

Using sqlmap to check for sql injection yields nothing

# Defaults

Googling the default login for request tracker gives us `root:password`. Using
this allows us to login 

> I should have found this without googling :facepalm:

# Info from request tracker

In the settings we find

```
DatabaseAdmin	'postgres'
DatabaseExtraDSN	{}
DatabaseHost	'localhost'
DatabaseName	'rtdb'
DatabasePassword	Password not printed	
DatabasePort	'3306'
DatabaseRTHost	'localhost'
DatabaseType	'mysql'
DatabaseUser	'rtuser'
```

On the users page we get

```
#	Name	Real Name	Email Address	Status
27	lnorgaard	Lise N�rgaard	lnorgaard@keeper.htb	Enabled
14	root	Enoch Root	root@localhost	Enabled
```

In the tickets page we find a ticket about a keepass dump with the following
contents:

```
I have saved the file to my home directory and removed the attachment for security reasons.

Once my investigation of the crash dump is complete, I will let you know.
```

Looking further at the users we can find the following comment for `lnorgaard`

```
New user. Initial password set to Welcome2023!
```

This password also works for SSH

# Keepass dump

It is possible to extract passwords from keepass.dmp files
(https://github.com/vdohney/keepass-password-dumper). Running this tool gives us

```
Password candidates (character positions):
Unknown characters are displayed as "\u25cf"
1.:     \u25cf
2.:     �, �, ,, l, `, -, ', ], �, A, I, :, =, _, c, M,
3.:     d,
4.:     g,
5.:     r,
6.:     �,
7.:     d,
8.:      ,
9.:     m,
10.:    e,
11.:    d,
12.:     ,
13.:    f,
14.:    l,
15.:    �,
16.:    d,
17.:    e,
Combined: \u25cf{�, �, ,, l, `, -, ', ], �, A, I, :, =, _, c, M}dgr�d med fl�de
```

> Or "Porridge with cream" in english
> Cleaned up `r�dgr�d med fl�de`

# Passwords

In the keepass file we find a password for the root user `F4><3K0nd!`. This
password doesn't work for ssh but we also have a `putty` key file in the notes.
Converting this to a pem certificate and passing this to `ssh` gives us root
access.

# Notes to self

- Try default passwords
- More enumeration, the user pass for SSH was easy to find but took me way too long
- Dont give up and read the output, the keepass pass was "corrupted" but the tool said so. Copy pasting the corrupt pass didn't work causing me to look somewhere else
- Installing putty in linux feels wrong
