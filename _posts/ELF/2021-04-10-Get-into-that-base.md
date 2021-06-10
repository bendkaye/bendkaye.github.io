---
layout: post
title: Get Into That Base
gh-repo: bendkaye/bendkaye.github.io
gh-badge: [star, fork, follow]
tags: [ELF]
---

Solving the guess challenge wasn’t that difficult, but it was an incredible learning experience. This challenge includes a lot, 
and it’s also impossible to guess the answer, or discover the answer with a simple trace  or  strings. 

| Author | Difficulty | Language |
| :------ |:--- | :--- |
| [js-on](http://crackmes.one/user/js-on)| 1.0 | C/C++ |

## just a head's up:

There’s an unusual register (for beginners) in the disassembly that might make solving this challenge daunting. To eliminate this, I’m going to explain it at the beginning of this walkthrough.

At _main+73h_ lists the following instruction:

    00101392 66 0f ef c9     PXOR       XMM1,XMM1
    
**XMM0** through **XMM7** are registers with a unique instruction set known as [SSE](https://en.wikibooks.org/wiki/X86_Assembly/SSE) (Streaming SIMD Extensions). They are 128bit registers and are capable of storing and performing double-precision floating point calculations.

They are necessary when performing calculations that demand more precision than a register containing a _float_.

## let's go!

Powering up the program, it demands an input. My first attempt fails, so I run ltrace.

<img width="921" alt="Screen Shot 2021-06-10 at 10 29 01" src="https://user-images.githubusercontent.com/46263689/121509524-67c0fe00-c9ef-11eb-866a-c6f0cc942848.png">

ltrace gives me my first hint as to what to look out for when I disassemble the program in ghidra.
Curious to lean more about the _gettimeofday_ function, I look it up in the BSD system calls manual. (man _gettimeofday_)

The function takes two arguments - _timeval_, which contains the time and _tzp_, which contains the timezone.

_timeval_ and _tzp_ are both structs:

{% highlight c linenos %}

  struct timeval {
           time_t       tv_sec;   /* seconds since Jan. 1, 1970 */
           suseconds_t  tv_usec;  /* and microseconds */
   };
   struct timezone {
           int     tz_minuteswest; /* of Greenwich */
           int     tz_dsttime;     /* type of dst correction to apply */
   };
     
{% endhighlight %}


Now I have an idea of what to look for in the disassembler. Main+53h calls the _gettimeofday_ function followed by instructions to store the seconds(tv_sec) and microseconds(tv_usec) into local variables.

    00101377 48 8b 45 b0     MOV        RAX,qword ptr [RBP + local_58]
    0010137b 48 89 45 98     MOV        qword ptr [RBP + local_70],RAX
    0010137f 48 8b 45 b8     MOV        RAX,qword ptr [RBP + local_50]
    00101383 48 89 45 a0     MOV        qword ptr [RBP + local_68],RAX
    00101387 48 8b 45 98     MOV        RAX,qword ptr [RBP + local_70]

In order to better understand what’s going on in the program, I manually reversed the assembly into a simple C program to show the seconds past since January 1, 1970.

{% highlight c linenos %}

#include <stdio.h>
#include <sys/time.h>

int main() {
  struct timeval current_time;
  gettimeofday(&current_time, NULL);
  printf("seconds since 1970: %ld\nmicro seconds since 1970 : %ld",
    current_time.tv_sec, current_time.tv_usec);

  return 0;
}

{% endhighlight %}

Then, arithmetic is performed on the values returned from gettimeofday(). Here lies the answer to our challenge.

    0010138b 48 69 c0        IMUL       RAX,RAX,0x3e8
             e8 03 00 00
    00101392 66 0f ef c9     PXOR       XMM1,XMM1
    00101396 f2 48 0f        CVTSI2SD   XMM1,RAX
             2a c8
    0010139b 66 0f ef c0     PXOR       XMM0,XMM0
    0010139f f2 48 0f        CVTSI2SD   XMM0,qword ptr [RBP + local_68]
             2a 45 a0
    001013a5 f2 0f 10        MOVSD      XMM2,qword ptr [DAT_00102038]                    = 408F400000000000h
             15 8b 0c 
             00 00
    001013ad f2 0f 5e c2     DIVSD      XMM0,XMM2
    001013b1 f2 0f 58 c1     ADDSD      XMM0,XMM1
    001013b5 f2 48 0f        CVTTSD2SI  RAX,XMM0
             2c c0
    001013ba 48 89 45 a8     MOV        qword ptr [RBP + local_60],RAX
    001013be 48 8b 45 a0     MOV        RAX,qword ptr [RBP + local_68]
    001013c2 48 89 c7        MOV        RDI,RAX
    001013c5 e8 83 fe        CALL       cross_sum                                        undefined cross_sum(long param_1)
             ff ff
    001013ca 48 98           CDQE
    001013cc 48 0f af        IMUL       RAX,qword ptr [RBP + local_68]
             45 a0
    001013d1 48 89 c1        MOV        RCX,RAX
    001013d4 48 8b 45 a8     MOV        RAX,qword ptr [RBP + local_60]
    001013d8 48 99           CQO
    001013da 48 f7 f9        IDIV       RCX
    001013dd 48 89 d0        MOV        RAX,RDX
    001013e0 89 45 90        MOV        dword ptr [RBP + local_78],EAX
    001013e3 c7 45 94        MOV        dword ptr [RBP + local_74],0x42
             42 00 00 00

It’s truly tedious to manually translate this, so I used the Ghidra decompiler to get a better sense of what’s going on.

~~~
local_70 = local_58.tv_sec;
  local_68 = local_58.tv_usec;
  local_60 = (long)((double)local_58.tv_usec / 1000.0 + (double)(local_58.tv_sec * 1000));
  iVar1 = cross_sum(local_58.tv_usec);
  local_78 = (int)(local_60 % (iVar1 * local_68));
  local_74 = 0x42;

~~~

Now I could solve this manually, but it would be annoying and the correct answer would be different by the time I reran the program. 

Time to fire up gdb, and gather the info while the program is running.

From ghidra, we know that the final value is stored in RBP-0x70 :
    
    00101413 39 45 90        CMP        dword ptr [RBP + local_78],EAX

We also know that at main+210, there is a prompt for user input. I set the breakpoint at *main+210 and run the program in gdb.

<img width="640" alt="Screen Shot 2021-06-10 at 13 38 26" src="https://user-images.githubusercontent.com/46263689/121511285-24678f00-c9f1-11eb-8ef0-89fca1ffded5.png">

Now for the next step, I got stuck, but it was a good reminder to take into account syntax when working with gdb. I found [this](https://visualgdb.com/gdbreference/commands/x) guide online which explains the x command quite nicely. In order to examine a memory address, I used the command: **x/dg *memory_location.**

‘g’ returns a 64-bit word at memory location, when the input accepted by the program was a 32-bit word. The correct syntax is: x/dw $rbp-0x70 .

<img width="640" alt="Screen Shot 2021-06-10 at 13 43 43" src="https://user-images.githubusercontent.com/46263689/121511962-e323af00-c9f1-11eb-9809-244331300e23.png">

I copied the password, pressed ‘c’ for continue, and pasted the password.

<img width="640" alt="Screen Shot 2021-06-10 at 13 45 11" src="https://user-images.githubusercontent.com/46263689/121512143-15cda780-c9f2-11eb-88ba-3e6b1009b887.png">



