# Analytics

Manual look up of the website gives the following info

- `analytical.htb` Fairly simple company page, no real entrypoints
- `data.analytical.htb` Login page for metabase. Metabase is a tool to graphically display info from a database.

# 10-08 20:39

Referencing https://blog.assetnote.io/2023/07/22/pre-auth-rce-metabase/ we know
there is some data that is still visible in metabase even after authentication.
Fetching `data.analytical.htb/api/session/properties` we get the setup token of:

```
setup-token: "249fa03d-fd94-4d5b-b94f-b4ebf3df681f"
```

This also looks like a good way to get an RCE. Time to craft a payload.

# 10-11 20:46

I'm in :sunglasses:

In short we could use the token with a carefully crafted payload to gain remote
code execution. An example PoC to gain RCE is in `payloads/rce.go`

# 10-11 20:54

The following user was found in the environment variables
```
META_USER=metalytics
META_PASS=An4lytics_ds20223#
```

We can use this data to get SSH access
```
Last login: Wed Oct 11 18:40:20 2023 from 10.10.14.121
metalytics@analytics:~$
```

# 10-11 20:10

There is a docker container or multiple running. Just looking from HTB
perspective this doesn't happen often so this might be the thing to target.

# 10-11 22:10

From the user we can't really do much, docker runs as root but we are not part
of the docker group. Maybe we can break out of the container instead?

# 10-13 21:01

The docker container was a false flag. It seems the ubuntu version of the target
machine has a local privilege escalation exploit we could exploit, more info:
https://www.wiz.io/blog/ubuntu-overlayfs-vulnerability

Using a small exploit script gave us root access in no time. Flag captured, left
no traces. :sunglasses:
