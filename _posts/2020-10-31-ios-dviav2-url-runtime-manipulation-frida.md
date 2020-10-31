---
layout: post
title:  "DVIA v2 iOS URL Runtime Manipulation with Frida"
date:   2020-10-31
categories:
    - blog
tags:
    - iOS
---


![Screenshot]({{ site.baseurl }}/images/2020-10-31-ios-dviav2-url-runtime-manipulation-frida/img0.png)


After my previous blog posts about [DVIA](https://syrion.me/blog/ios-dvia-antidebugging-bypass) and [Frida with Swift](https://syrion.me/blog/ios-swift-antijailbreak-bypass-frida) some guys asked me about the URL Runtime Manipulation challenge in [DVIA v2](https://github.com/prateek147/DVIA-v2). 
I will not show my solution for **“Login Method 1”**, **“Login Method 2”** and **“Validate Code”** challenges because they were already described [here](https://philkeeble.com/ios/reverse-engineering/iOS-Runtime-Manipulation), I will focus only on the **“Read Tutorial”** challenge. 

### URL Runtime Manipulation Challenge

The image below shows the **“Runtime Manipulation”** section in DVIA v2.

![Screenshot]({{ site.baseurl }}/images/2020-10-31-ios-dviav2-url-runtime-manipulation-frida/img1.png)

![Screenshot]({{ site.baseurl }}/images/2020-10-31-ios-dviav2-url-runtime-manipulation-frida/img2.png)

When pressing the **“Read tutorial”** button, the url “http://highaltitudehacks.com/2013/11/08/ios-application-security-part-21-arm-and-gdb-basics” is opened in Safari as showed in the image below.

![Screenshot]({{ site.baseurl }}/images/2020-10-31-ios-dviav2-url-runtime-manipulation-frida/img3.png)

### Reversing

First of all, let's find this URL in the application binary, for this purpose I used the Ghidra function **“Search for Strings”**.

![Screenshot]({{ site.baseurl }}/images/2020-10-31-ios-dviav2-url-runtime-manipulation-frida/img4.png)

By clicking on the string, we can see it at address **0x1003476e0**.

![Screenshot]({{ site.baseurl }}/images/2020-10-31-ios-dviav2-url-runtime-manipulation-frida/img5.png)

By looking at the memory map, we can see that our string is in the **__cstring** section, this section is part of the **__TEXT** segment, it’s a read-only area, containing the **“Literal string constants (quoted strings in source code)”** as reported in the [Apple documentation](https://developer.apple.com/library/archive/documentation/Performance/Conceptual/CodeFootprint/Articles/MachOOverview.html#//apple_ref/doc/uid/20001860-BAJGJEJC).

![Screenshot]({{ site.baseurl }}/images/2020-10-31-ios-dviav2-url-runtime-manipulation-frida/img6.png)

There are three references to our string.

![Screenshot]({{ site.baseurl }}/images/2020-10-31-ios-dviav2-url-runtime-manipulation-frida/img7.png)

After a bit of debugging using LLDB, we can see that when pressing the **“START CHALLENGE”** button the third reference is reached (I didn’t find any way to reach the other two references).

![Screenshot]({{ site.baseurl }}/images/2020-10-31-ios-dviav2-url-runtime-manipulation-frida/img8.png)

By disassembling the address **0x10019d980** we can find our string address and its length.

![Screenshot]({{ site.baseurl }}/images/2020-10-31-ios-dviav2-url-runtime-manipulation-frida/img9.png)

## Frida Script

Let’s create a simple Frida script in order to read the registers **x0** and **x9** containing respectively the string length and the string address.

~~~
var addr = ptr(0x19d988) 
var t_module = 'DVIA-v2';
var nw = Module.getBaseAddress(t_module);
var toAtt = nw.add(addr);

Interceptor.attach(toAtt, {
    onEnter: function(args) {
    console.log("")
    var stringAddress = this.context.x0
    var stringLength = this.context.x9
    console.log("[!] String: " + stringAddress.readUtf8String() + "at address " + stringAddress)
    console.log("[!] Length: " + stringLength)
    },
});
~~~

![Screenshot]({{ site.baseurl }}/images/2020-10-31-ios-dviav2-url-runtime-manipulation-frida/img10.png)

As we expected, we can dump our url and its length. I want to overwrite the url “http://highaltitudehacks.com/2013/11/08/ios-application-security-part-21-arm-and-gdb-basics”  with “https://syrion.me”, we have to change the **x9** register value with my url length **0x11** and overwrite the URL with the new one, to do this we can use the Frida **Memory.protect** function in order to make the first **17**  bytes of the string at address contained in the **x0** register, readable and writable (ASLR) and then, use the the **Memory.copy** to copy "https://syrion.me" into the address contained in the **x0** register.

~~~
var addr = ptr(0x19d990) 
var t_module = 'DVIA-v2';
var nw = Module.getBaseAddress(t_module);
var toAtt = nw.add(addr);

Interceptor.attach(toAtt, {
    onEnter: function(args) {
        var stringAddress = this.context.x0
        var stringLength = this.context.x1
        console.log("[!] String: " + stringAddress.readUtf8String() + "at address " + stringAddress)
        console.log("[!] Length: " + stringLength)
        var myString = Memory.allocUtf8String("https://syrion.me")
        this.context.x1=0x11
        Memory.protect(this.context.x0,17,'rw-')
        Memory.copy(this.context.x0, myString, 17)
    },
});
~~~

By running it, we  successfully change the opened URL.

![Screenshot]({{ site.baseurl }}/images/2020-10-31-ios-dviav2-url-runtime-manipulation-frida/img11.png)