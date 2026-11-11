### Initial Foothold
Run nmap. Found port 80 and ssh open.\
Run ssh brute script, run feroxbuster on port 80.\
check out /robots.txt, obtain new directory /Enum_this_Box which leads to nowhere.\
Feroxbuster further with raft-medium-directories.txt. \
Found /mini.php, which allows us to upload php files.\
Run `penelope -p 4444` as listening port. Upload php reverse shell and add additional permissions on website. Go towards /php-reverse-shell.php. Caught a session with penelope.

### Privesc
Check for suid and capabilities. psexec seems suspicious as could be pwnkit.\
Run linpeas. Linpeas suggest pwnkit as well. wget, chmod +x and ran pwnkit. Obtain root.
