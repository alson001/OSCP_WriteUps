Initial Access

Login ftp anonymously

Get trytofind.jpeg

Explore target website, get to his /blog directory and then
**/S3cret-T3xt** directory.

used **steghide** and the secret key **3xtr4ctd4t4** (obtained from
/S3cret-T3xt directory with view-source) for extraction from the image
file and found a **data.txt** file.

using the following command given below

**steghide extract -sf trytofind.jpg**

data file showed user renu

crack user password with Hydra and rockyou.txt

**hydra -l renu -P /usr/share/wordlists/rockyou.txt ssh://machine Ip**

Successfully ssh into renu

**Privilege Escalation**

Run linpeas, realised there is another user lily. Go into her from
/home/lily

And go into the hidden /.ssh folder. **ssh lily@127.0.0.1**

- **ssh**: This is the tool used to securely connect to another
  computer\'s command line over a network.

- **lily**: This is the username they want to log in as.

- **@**: This separates the username from the address.

- **127.0.0.1**: This is a special IP address known as
  **\"localhost.\"** It always points to the computer you are currently
  sitting at.

**What this means:** The user (remu) is on a machine called MoneyBox.
They just used SSH to connect to their *own* machine, but logged in as a
*different user* named lily. (This is a very common move in CTFs after
finding a user\'s password---you test if the password works by logging
in as them locally).

When you type ssh lily@127.0.0.1, the SSH program automatically checks
your \~/.ssh/ folder for a file called id_rsa. If it finds it, and it
matches the authorized_keys file for lily, it logs you in automatically
without asking for a password. The content (in authorized_keys file)
mentions of user **renu** who can connect to **ssh** with **lily**
without having to have a password.

Next I checked for the sudo rights with sudo -l and lily can run
**perl** as root without password.

Browsed our friendly neighborhood
[https://gtfobins.github.io/](https://gtfobins.github.io/#perl) and ran
the **perl** command and got a **root** shell immediately.

**sudo perl -e \'exec \"/bin/sh\"\'**
