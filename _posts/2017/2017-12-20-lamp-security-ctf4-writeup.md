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