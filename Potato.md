Run nmap to find port 22, 80 and 2112 open
2112 is ftp and allows for anonymous login. Get two file welcome.msg and index.php.back

The latter has a php juggling vulnerability as it uses strcmp and a weak equivalent == sign.
![[Pasted image 20261109190303.png]]

Using password[] = '' with the burpsuite post request, we successfully login into the dashboard. Go the get logs and interrupt getting one of the logs with burpsuite. Send to repeater and replace file with ../../../../../../../../../etc/passwd
![[Pasted image 20261109190632.png]]

Get the webadmin hash.
![[Pasted image 20261109191127.png]]
Cracked with md5crypt hashcat mode 500. ssh into webadmin with password dragon

In webadmin, run sudo -l. Found out /bin/nice and /notes/* on gtfo bin. However not allowed to run `sudo nice /bin/sh`. Abuse notes wildcard for sudo and ran
`sudo nice /notes/../../../../bin/sh`

Got root.


