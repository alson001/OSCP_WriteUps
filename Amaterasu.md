**Initial Access**

Find there is a POST function under /file-upload

GET /file-list?dir=/tmp: List of the files

Change tmp to /home then /alfredo then /.ssh but enumerating them one by
one.

ssh-keygen your own ssh keys and upload the public key into the /.ssh
fold

your ssh private key should be able to get access after your chmod 600
it

**Privilege Escalation**

Run linpeas by using

scp -P 25022 -i id_rsa \~/Documents/linpeas/linpeas.sh
alfredo@192.168.x.249:/tmp/

chmod +x linpeas.sh

./linpeas.sh

**The Key: The PATH Variable**

Look at line 2 of the script:\
export PATH=\"/home/alfredo/restapi:\$PATH\"

The **PATH** is a list of folders the computer looks in when you type a
command. When the script says tar czf\..., the computer doesn\'t know
where tar lives yet. It reads that PATH line and says, *\"First, I will
look inside /home/alfredo/restapi/ to see if there is a program called
tar in there.\"*

Because you put your malicious file named tar in that exact folder, the
computer finds **your** tar before it ever finds the real Linux tar
program. This is called **PATH Hijacking**.

**The Cronjob**

Look at the /etc/crontab file:\
\*/1 \* \* \* \* root /usr/local/bin/backup-flask.sh

- \*/1 \* \* \* \* means **every single minute**.

- root means it runs as the super-admin (the highest level of access).

- The script backup-flask.sh is run by root.

**How the Attack Happens (Step-by-Step)**

1.  **The Timer:** Every minute, the system wakes up and says, \"Time to
    run the backup script as root!\"

2.  **The Script Starts:** It runs backup-flask.sh.

3.  **The PATH Change:** The script changes the PATH to put your folder
    first.

4.  **The Trick:** The script gets to the line tar czf /tmp/flask.tar.gz
    \*. It wants to run the tar command. Because of step 3, it looks in
    /home/alfredo/restapi/ first. It finds *your* file named tar and
    runs it **as root**.

5.  **The Exploit:** Your file runs the command chmod u+s /bin/bash.
    This tells the system, \"Make the Bash shell run with Root
    permissions for anyone who uses it.\"

6.  **The Reward:** Because this happened as root, the u+s (setuid) bit
    gets set. Now, when you (as Alfredo) type /bin/bash -p in your
    terminal, you will instantly become root.
