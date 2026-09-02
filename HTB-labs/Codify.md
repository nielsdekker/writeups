# Codify

Initial look at the website shows a tool for running nodeJS code in _"A secure
environment"_. The about page gives us the tool for this: `vm2 at version
3.9.16`

Quick google search shows there is a sandbox bypass vulnerability for every
version below `3.9.18`. We should be able to use this to run our own code.

> https://security.snyk.io/vuln/SNYK-JS-VM2-5537100

# 2023-10-05 12:39

Using the poc here shows we can execute arbitrary code. Running a basic bash
revshell errors but we can use python to get a shell going. See
`Payloads/Foothold`

# 2023-10-05-13:01

Using the payload we get shell access for a limited `svc` user. We do find
another user with the name `joshua` in the `/home` folder. 

# 2023-10-05 13:44

Looking in the `/var/www/` folder I came across a `/ticket` subdomain which
contains login stuff and a database.

This database also contains the `joshua` user and a password. This hash was
crackable with `hashcat`

# 2023-10-05 13:53

The ticket database resulted in a login for `joshua` with shell/ssh permissions.
Running `sudo -l` shows we can run the backup script for the database as root. 

# 2023-10-05 16:51

The database backup script can be run with an authentication bypass by passing
`*` as the password. This results in the backup script from running which
executes a command similar to `mysqldump -u root -p "the password"`. Using
`pspy` to monitor started commands we get the password of the database.

The retrieved password is not just for `mysql` but also for the root account
