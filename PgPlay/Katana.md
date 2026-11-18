### Initial Foothold
autorecon on 192.168.180.83
```
21/tcp   open  ftp          syn-ack ttl 61 vsftpd 3.0.3
22/tcp   open  ssh          syn-ack ttl 61 OpenSSH 7.9p1 Debian 10+deb10u2 
80/tcp   open  http         syn-ack ttl 61 Apache httpd 2.4.38 ((Debian))
139/tcp  open  netbios-ssn? syn-ack ttl 61
445/tcp  open  netbios-ssn  syn-ack ttl 61 Samba smbd 4.9.5-Debian (workgroup: 
7080/tcp open  empowerid    syn-ack ttl 61 LiteSpeed
8088/tcp open  http         syn-ack ttl 61 LiteSpeed httpd
8715/tcp open  unknown      syn-ack ttl 61
```

Anonymous ftp and smb disabled. 
Ran enum4linux-ng on smb to reveal users debian and katana.

Go to website as 8715 and tried admin:admin as credentials. It worked.
Directory bust port 80, 7080, 8088 and 8715.
Nothing interesting on 80, 7080, 8715.
```
http://192.168.180.83:8088/upload.html
File : file1  
Name : php-reverse-shell.php  
Type : application/x-php  
Path : /tmp/phptwSbEw  
Size : 5490  
Please wait for 1 minute!. Please relax!.

Moved to other web server: /tmp/phptwSbEw ====> /opt/manager/html/katana_php-reverse-shell.php
```
There is somewhere to upload on port 8088. Upload a php reverse shell.
Started a penelope listener and went to port 8715 site as it is the most suspicious
```
http://192.168.180.83:8715/katana_php-reverse-shell.php
```

Attain initial access

### Privesc
check capabilities to show python. Go to gtfobins. copy command and get root.