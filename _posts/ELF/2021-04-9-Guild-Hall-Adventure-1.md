---
layout: post
title: Guild Hall Adventure Ch 1
gh-repo: bendkaye/bendkaye.github.io
gh-badge: [star, fork, follow]
tags: [ELF]
---

This crack me is part of a "text based adventure" series. Not that difficult to solve, but I did get stuck on the final answer. It's pretty common with Reverse Engineering, the answer is right in front of you and is super simple in hindsight. I guess that's part of the fun :)

| Author | Difficulty | Language |
| :------ |:--- | :--- |
| [ker2x](http://crackmes.one/user/ker2x) | 1.0 | C/C++ |

## Let's get started

**Step One:** 
I opened the terminal and ran the program, which closed immediately because it required a command line argument. I actually guessed the first part of the crackme, although any disassembler would have shown you the first answer in plain text.

<img width="733" alt="Screen Shot 2021-06-09 at 10 20 53" src="https://user-images.githubusercontent.com/46263689/121474782-3cc5b280-c9cd-11eb-936c-cd5a0e10a632.png">


I plug the program into Ghidra. Because we know the program expects a command line argument, we can easily edit the function signature in the awesome decompiler that ghidra offers, making the program much more readable. _Right click on param1, edit function signature, change **param1** to argc and **param2** to argv.

<img width="649" alt="Screen Shot 2021-06-09 at 10 23 20" src="https://user-images.githubusercontent.com/46263689/121314783-6240b680-c910-11eb-9bd2-cff609396f03.png">


Register EDI contains argc, RSI contains argv and they are both written to variables on the stack.


    MOV        qword ptr [RBP + local_10],RAX                              
    XOR        EAX,EAX
    LEA        RAX,[s_hello_00100a08]                           = "hello"

    MOV        qword ptr [RBP + local_a8],RAX=>s_hello_00100a08 = "hello"

    CMP        dword ptr [RBP + local_ac],0x2

    JZ         LAB_0010084e                               <-----#if the argument count (argc) is not 2, program puts message and closes **
    LEA        argc,[s_Woaaa_!_what_about_being_polite_?_0010   = "Woaaa ! what about being poli  

    CALL       puts                                             int puts(char * __s)

    MOV        EAX,0xffffffff

    JMP        LAB_00100967




Following the jump, it compares argv[1] with the string "hello". It does this by pointing RAX to the variable on the stack that holds argv, b8,and adding 0x8.
                                                              
    MOV        RAX,qword ptr [RBP + local_b8]
    ADD        RAX,0x8
    MOV        RAX,qword ptr [RAX]
    MOV        argc,RAX
    CALL       strlen         <--------------- it calls strlen to check the length of our argument, jumping if its not 5 letters long                                         
    CMP        RAX,0x5
    JNZ        LAB_00100922
    
 
 Everyting was pretty simple for me, but this is where I got confused:
 
    MOV        RDX,RAX
    MOV        RAX,qword ptr [RBP + local_b8]

    MOV        RAX,qword ptr [RAX]
    LEA        RCX=>local_98,[RBP + -0x90]        <------------- user input

    MOV        argv,RCX
    MOV        argc,RAX          
    CALL       strncmp                               <--------- argv is compared with my user input

I went back to the program, and entered "hello" again, but it didn't work. It took me longer than I'd like to admit until I figured out the solution, but I eventually did. 

It's easier to understand when it's not in assembly, so I've decompiled the above into C.

{% highlight c linenos %}
length = strlen(user_input)
i = strncmp(*argv, user_input, length)

if (i == 0) {
  puts("good, good, welcome to the guild hall!");
  }
{% endhighlight %}

Do you see it now? It's comparing **argv** not argv[1]!

argv **points to the name of the program**

<img width="644" alt="Screen Shot 2021-06-09 at 13 38 25" src="https://user-images.githubusercontent.com/46263689/121340175-fa966580-c927-11eb-8919-6a4bd61a2909.png">

I type in the name of the program, and the game has been won!


### something cool I learnt

because _length_ is determined by _user_input_, you don't need to write out "./adventure". 
Even just a "." will suffice.
<img width="644" alt="Screen Shot 2021-06-09 at 13 42 54" src="https://user-images.githubusercontent.com/46263689/121340731-9a53f380-c928-11eb-8752-a610e5972ed7.png">




