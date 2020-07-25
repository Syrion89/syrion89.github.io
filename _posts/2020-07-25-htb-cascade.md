---
layout: post
title:	"Cascade Writeup"
date:	2020-07-25
categories:
    - blog
tags:
    - hackthebox
---

![Screenshot]({{ site.baseurl }}/images/2020-07-25-htb-cascade/img0.png)

### User Flag

Let’s start by enumerating all the service on the machine with a TCP scan:

~~~
root@kali:~# nmap -sT -sV -T4 10.10.10.182 -p-
Starting Nmap 7.80 ( https://nmap.org ) at 2020-04-07 18:28 EDT
Stats: 0:00:56 elapsed; 0 hosts completed (1 up), 1 undergoing Connect Scan
Nmap scan report for cascade.local (10.10.10.182)                                                                                                                              
Host is up (0.048s latency).                                                                                                                                                   
Not shown: 65520 filtered ports                                                                                                                                                
PORT      STATE SERVICE       VERSION                                                                                                                                          
53/tcp    open  domain        Microsoft DNS 6.1.7601 (1DB15D39) (Windows Server 2008 R2 SP1)                                                                                   
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2020-04-07 22:31:24Z)                                                                                   
135/tcp   open  msrpc         Microsoft Windows RPC                                                                                                                            
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn                                                                                                                    
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: cascade.local, Site: Default-First-Site-Name)                                                   
445/tcp   open  microsoft-ds?                                                                                                                                                  
636/tcp   open  tcpwrapped                                                                                                                                                     
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: cascade.local, Site: Default-First-Site-Name)                                                   
3269/tcp  open  tcpwrapped                                                                                                                                                     
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)                                                                                                          
49154/tcp open  msrpc         Microsoft Windows RPC                                                                                                                            
49155/tcp open  msrpc         Microsoft Windows RPC                                                                                                                            
49157/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0                                                                                                              
49158/tcp open  msrpc         Microsoft Windows RPC                                                                                                                            
49165/tcp open  msrpc         Microsoft Windows RPC                                                                                                                            
Service Info: Host: CASC-DC1; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008:r2:sp1, cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 194.98 seconds

~~~

In LDAP we can find the password for the user r.thompson

![Screenshot]({{ site.baseurl }}/images/2020-07-25-htb-cascade/img1.png)

~~~
root@kali:~# echo "clk0bjVldmE=" | base64 -d
rY4n5eva
~~~

With the password “rY4n5eva” , we can access smb as user r.thompson. There are several shares we can read:

~~~
root@kali:~/Desktop/file# crackmapexec smb 10.10.10.182  -u r.thompson -p rY4n5eva --shares
SMB         10.10.10.182    445    CASC-DC1         [*] Windows 6.1 Build 7601 x64 (name:CASC-DC1) (domailse)
SMB         10.10.10.182    445    CASC-DC1         [+] CASCADE\r.thompson:rY4n5eva 
SMB         10.10.10.182    445    CASC-DC1         [+] Enumerated shares
SMB         10.10.10.182    445    CASC-DC1         Share           Permissions     Remark
SMB         10.10.10.182    445    CASC-DC1         -----           -----------     ------
SMB         10.10.10.182    445    CASC-DC1         ADMIN$                          Remote Admin
SMB         10.10.10.182    445    CASC-DC1         Audit$                          
SMB         10.10.10.182    445    CASC-DC1         C$                              Default share
SMB         10.10.10.182    445    CASC-DC1         Data            READ            
SMB         10.10.10.182    445    CASC-DC1         IPC$                            Remote IPC
SMB         10.10.10.182    445    CASC-DC1         NETLOGON        READ            Logon server share 
SMB         10.10.10.182    445    CASC-DC1         print$          READ            Printer Drivers
SMB         10.10.10.182    445    CASC-DC1         SYSVOL          READ            Logon server share
~~~

Using smbclient, we can access the Data share and find a lot of files:

![Screenshot]({{ site.baseurl }}/images/2020-07-25-htb-cascade/img3.png)

There is an intesting file in \IT\Temp\s.smith:

![Screenshot]({{ site.baseurl }}/images/2020-07-25-htb-cascade/img4.png)

~~~
root@kali:~/Desktop/file# cat VNC\ Install.reg 
��Windows Registry Editor Version 5.00

[HKEY_LOCAL_MACHINE\SOFTWARE\TightVNC]

[HKEY_LOCAL_MACHINE\SOFTWARE\TightVNC\Server]
"ExtraPorts"=""
"QueryTimeout"=dword:0000001e
"QueryAcceptOnTimeout"=dword:00000000
"LocalInputPriorityTimeout"=dword:00000003
"LocalInputPriority"=dword:00000000
"BlockRemoteInput"=dword:00000000
"BlockLocalInput"=dword:00000000
"IpAccessControl"=""
"RfbPort"=dword:0000170c
"HttpPort"=dword:000016a8
"DisconnectAction"=dword:00000000
"AcceptRfbConnections"=dword:00000001
"UseVncAuthentication"=dword:00000001
"UseControlAuthentication"=dword:00000000
"RepeatControlAuthentication"=dword:00000000
"LoopbackOnly"=dword:00000000
"AcceptHttpConnections"=dword:00000001
"LogLevel"=dword:00000000
"EnableFileTransfers"=dword:00000001
"RemoveWallpaper"=dword:00000001
"UseD3D"=dword:00000001
"UseMirrorDriver"=dword:00000001
"EnableUrlParams"=dword:00000001
"Password"=hex:6b,cf,2a,4b,6e,5a,ca,0f
"AlwaysShared"=dword:00000000
"NeverShared"=dword:00000000
"DisconnectClients"=dword:00000001
"PollingInterval"=dword:000003e8
"AllowLoopback"=dword:00000000
"VideoRecognitionInterval"=dword:00000bb8
"GrabTransparentWindows"=dword:00000001
"SaveLogToAllUsersPath"=dword:00000000
"RunControlInterface"=dword:00000001
"IdleTimeout"=dword:00000000
"VideoClasses"=""
"VideoRects"=""
~~~

The file contains a VNC decrypted password, we can use this tool, to decrypt it:

![Screenshot]({{ site.baseurl }}/images/2020-07-25-htb-cascade/img5.png)

With these credentials e can access win-rm and get the user flag:

![Screenshot]({{ site.baseurl }}/images/2020-07-25-htb-cascade/img6.png)

## Root Flag

With the s.smith user, we can access the Audit$ share:

~~~
root@kali:~/Desktop# crackmapexec smb 10.10.10.182 -u s.smith -p sT333ve2 --shares
SMB         10.10.10.182    445    CASC-DC1         [*] Windows 6.1 Build 7601 x64 (name:CASC-DC1) (domailse)
SMB         10.10.10.182    445    CASC-DC1         [+] CASCADE\s.smith:sT333ve2 
SMB         10.10.10.182    445    CASC-DC1         [+] Enumerated shares
SMB         10.10.10.182    445    CASC-DC1         Share           Permissions     Remark
SMB         10.10.10.182    445    CASC-DC1         -----           -----------     ------
SMB         10.10.10.182    445    CASC-DC1         ADMIN$                          Remote Admin
SMB         10.10.10.182    445    CASC-DC1         Audit$          READ            
SMB         10.10.10.182    445    CASC-DC1         C$                              Default share
SMB         10.10.10.182    445    CASC-DC1         Data            READ            
SMB         10.10.10.182    445    CASC-DC1         IPC$                            Remote IPC
SMB         10.10.10.182    445    CASC-DC1         NETLOGON        READ            Logon server share 
SMB         10.10.10.182    445    CASC-DC1         print$          READ            Printer Drivers
SMB         10.10.10.182    445    CASC-DC1         SYSVOL          READ            Logon server share
~~~

![Screenshot]({{ site.baseurl }}/images/2020-07-25-htb-cascade/img7.png)

In the share there is an exe file, some DLLs and a DataBase:

![Screenshot]({{ site.baseurl }}/images/2020-07-25-htb-cascade/img8.png)

From the db, we can find a base64 cypher password for the user Arksvc:

![Screenshot]({{ site.baseurl }}/images/2020-07-25-htb-cascade/img9.png)

By [reversing](https://www.jetbrains.com/decompiler) the file CascAudit.exe, we find a method called get_Password, this method brings us to the CascCrypto.dll. In the DLL, we find a class called Crypto, in this class there are the IV and two methods called EncryptString and DescriptString as shown in the image below:

![Screenshot]({{ site.baseurl }}/images/2020-07-25-htb-cascade/img10.png)

By right click on the function DecryptString we can use the “find usages” to find the call of the method, and yes we find it and the key:

![Screenshot]({{ site.baseurl }}/images/2020-07-25-htb-cascade/img11.png)

At this point we have all we need to decrypt the password for the user Arksvc:

IV: 1tdyjCbY1Ix49842
Key: c4scadek3y654321
Encrypted String: BQO5l5Kj9MdErXx6Q6AGOw==

In the source code, we can see it uses AES in CBC mode.

Using CyberChef we can obtain the password:

![Screenshot]({{ site.baseurl }}/images/2020-07-25-htb-cascade/img12.png)

We can access the machine as user arksvc:

![Screenshot]({{ site.baseurl }}/images/2020-07-25-htb-cascade/img13.png)

The user is member of the CASCADE\AD Recycle Bin group:

![Screenshot]({{ site.baseurl }}/images/2020-07-25-htb-cascade/img14.png)

I didn’t tell you but, at the beginning, as user t.thompson, I found two interesting file in the Data share:

\IT\Email Archives\ Meeting_Notes_June_2018.html

![Screenshot]({{ site.baseurl }}/images/2020-07-25-htb-cascade/img15.png)

\IT\Logs\Ark AD Recycle Bin\ ArkAdRecycleBin.log

![Screenshot]({{ site.baseurl }}/images/2020-07-25-htb-cascade/img16.png)

If you remember, the TempAdmin was also in the Audit.db into the “DeletedUserAudit” table. So, the email tells us there was a user called “TempAdmin” with the same password as the Administrator, we know the this user was deleted and it is in the Recycle Bin and we are member of AD Recycle Bin group.

![Screenshot]({{ site.baseurl }}/images/2020-07-25-htb-cascade/img17.png)

After some enumeration on google we can find this article, using the following command we can find the base64 password for the user TempAdmin:

~~~
Get-ADObject -Filter {SamAccountName -eq 'TempAdmin'} -IncludeDeletedObjects -Properties *
~~~

![Screenshot]({{ site.baseurl }}/images/2020-07-25-htb-cascade/img18.png)

![Screenshot]({{ site.baseurl }}/images/2020-07-25-htb-cascade/img19.png)

The email told us, the password is the same for the Administrator, let’s try to connect as Administrator:

![Screenshot]({{ site.baseurl }}/images/2020-07-25-htb-cascade/img20.png)