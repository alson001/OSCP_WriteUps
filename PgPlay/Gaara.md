Run nmap with port 80 and port 22 open.\
Run feroxbuster with medium list on victim ip.\
There was a directory called **Cryoserver,** which mentioned another three sub-directories:\
Temari, Kazekage, iamGaara\
In the **iamGaara** sub-directory I finally found a weird string: **f1MgN9mTf9SNbzRygcU**\
Using an online cipher identifier https://www.dcode.fr/cipher-identifier, it says its a base58 encoding. In cyberchef, crack this message and got password: ismyname

SSH with this password but failed. Try to hydra ssh with gaara user using rockyou.txt. Obtained password iloveyou2 and gain initial foothold.\
Run `find / -perm -u=s -type f 2>/dev/null` to list SUID. Realise gdb has SUID permissions.\
Use GTFO bins 
```
gdb -nx -ex '!/bin/sh' -ex quit
```
Did not work. Try the Capabilities.
```
gdb -nx -ex 'python import os; os.setuid(0)' -ex '!/bin/sh' -ex quit
```
Obtain root.
