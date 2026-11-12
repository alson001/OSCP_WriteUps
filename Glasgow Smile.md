### Initial Foothold
Run nmap scan to reveal port 22, 80 open. Run feroxbuster to find out site is built on Joomla.
Run joomscan 
Joomla 3.7.3rc1 and a bunch of plugins that does not have any available RCE exploits online.


Access `http://192.168.145.79/joomla/administrator/index.php` 
Realise it is a login page. 
cewl the site for credentials, `cewl -d 4 -m 5 --with-numbers -w wordlist.txt http://192.168.249.79/joomla/administrator/` 


Run joomla-brute.py script
`python3 joomla-brute.py -u http://192.168.145.79/joomla/administrator/index.php -w cewl2.txt -U cewl2.txt -v `
```
Joomla:whole
Joomla:freak
**Joomla:Gotham**
software:Joker
```

Enter credentials and gained assess to admin rights.
From Hacktricks:
If you managed to get **admin credentials** you can **RCE inside of it** by adding a snippet of **PHP code** to gain **RCE**. We can do this by **customizing** a **template**.

1. **Click** on **`Templates`** on the bottom left under `Configuration` to pull up the templates menu.
2. **Click** on a **template** name. Let’s choose **`protostar`** under the `Template` column header. This will bring us to the **`Templates: Customise`** page.
3. Finally, you can click on a page to pull up the **page source**.
We realise that we need a page source that we can access so we cannot use stuff like error.php. Use index.php and insert the pentestmonkey php reverse shell script at the top of the code. Start penelope at 4444 to catch.
Obtain initial access and found flag in /home/rob.

### Privesc
PwnKit works because pkexec has sudo bits.

Intended method:
Run `netstat -tulpn` to see that port 3306(MySQL) is listening
Dig around to find config file in /var/www/html/joomla. Obtain joomla and babyjoker as credentials for MySQL database.

In MySQL, go into batjoke database and list out taskforce. rob which is a user has his credentials listed.
`Pz8/QWxsSUhhdmVBcmVOZWdhdGl2ZVRob3VnaHRzPz8/`
https://www.dcode.fr/cipher-identifier says its base64
`???AllIHaveAreNegativeThoughts???` obtained using cyberchef

`su rob` to switch to rob user. Go to /home/rob to find a note encoded in ROT cipher by using the cipher identifier again. Decode it with cyberchef with rotation 1.

`STMzaG9wZTk5bXkwZGVhdGgwMDBtYWtlczQ0bW9yZThjZW50czAwdGhhbjBteTBsaWZlMA==`
`I33hope99my0death000makes44more8cents00than0my0life0`
This password turns out to be abner. su abner.

Run linpeas on abner to find out there is a hidden file.
```
Interesting writable files owned by me or writable by everyone (not in Home) (max 200)                               
...
/var/lib/php/sessions
/var/tmp
/var/www/joomla2/administrator/manifests/files/.dear_penguins.zip
```
Copy to tmp folder as we cannot unzip there. Unzip and obtain penguin password.
`scf4W7q4B4caTMRhSFYmktMsn87F35UkmKttM5Bz`

Under `/home/penguin/SomeoneWhoHidesBehindAMask`, root executes `.trash_old` every minute.
Overwrite the script in .trash_old to give us a reverse shell on root.





