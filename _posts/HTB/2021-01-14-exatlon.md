---
layout: post
title: Exatlon
gh-repo: bendkaye/bendkaye.github.io
gh-badge: [star, fork, follow]
tags: [HTB]
---

This is my first ever walkthrough of a reverse challenge, and the second RE challenge on Hack The Box that I’ve completed so far. I’m really excited to share how 
I solved this challenge, and I hope to help others who get stuck (as I did, many, many times).

| Author | Difficulty | Challenge |
| :------ |:--- | :--- |
| [OctopusTR](https://app.hackthebox.eu/users/158119)| Easy | [Exatlon](https://app.hackthebox.eu/challenges/121) |


## let's rock!

Try to run the program in the terminal, just to see what it does. The first things I noticed about the program is that it prints “EXATLON_V1” as a large 
banner that takes, for what feels like forever, to print itself before prompting the user to enter a password.

I enter various combinations of numbers, letters and symbols of different lengths to see if the program reacts differently. It appears to accept any input,
except for the letter “q,” which exits the program.


![1_hxSl72bnewyCyasCnPkkIg](https://user-images.githubusercontent.com/46263689/123754941-17d5a880-d8c4-11eb-92fe-e9cdcfb57573.png)

Next, I open the program in IDA64(free version), but everything appears to be unreadable. Open exatlon_v1 in [DIE](https://github.com/horsicq/Detect-It-Easy) to see what’s going on in there.

![1_ip6UtMMgxPmOoJAsK2rdFA](https://user-images.githubusercontent.com/46263689/123755113-3fc50c00-d8c4-11eb-8885-6a317ea45d81.png)


I can see that it’s been packed with UPX, and I proceed to unpack it with the following command in the terminal:

    upx -d exatlon_v1 -o exatlon_v1_unpacked

Now that the program is unpacked, I plug it into IDA64 to see what’s going on inside. Because I’ve already played around with the program, I have a general idea about what I will see inside: A call to multiple print functions, a function to check my input with a saved value, an exit call for when ‘q’ is entered, and a loop.

The first thing I look for in IDA is for the print function “[+] Enter Exatlon Password :”. I know that user input should come after that, as well as the call function for comparing the input.

![1_5zGIWrxyVf6Me8-6ac5CFg](https://user-images.githubusercontent.com/46263689/123755312-6f741400-d8c4-11eb-8b49-eafd99c411ea.png)


We can clearly see where Exatlon prompts the user to enter the password, as well as a function at memory location 0x404D29 which seems to convert the input.

Additionally, I can see another variable at 0x54B4F0 holding what seems to be the encrypted correct password.

    1152 1344 1056 1968 1728 816 1648 784 1584 816 1728 1520 1840 1664 784 1632 1856 1520 1728 816 1632 1856 1520 784 1760 1840 1824 816 1584 1856 784 1776 1760     528 528 2000


I am going to assume that each number corresponds to a letter, so the next move is to run Exatlon in GDB, and place a breakpoint after the call function of the program that converts the input (at 0x404D29). I run the program, and enter a single letter, “A”.

![1_DGTufaI-Y0DLs8Nc75W1Gg](https://user-images.githubusercontent.com/46263689/123755524-a64a2a00-d8c4-11eb-8dc2-2a5edc464a16.png)

There are two registers that change according to my input: R8 and R10. At first, I spent time trying to see how the value stored in R8 corresponded to “1152” in our code, but I couldn’t find anything, and moved on to the next register, R10.

    R10 holds the memory address of 0x7fffffffdbb4
    
![1_eHuN4iONOKU3e_Fpb46WFA](https://user-images.githubusercontent.com/46263689/123755587-be21ae00-d8c4-11eb-948c-f10154827294.png)

0x30343031 is hex for the ASCII value of 1040 and holds four digits, just like our encrypted answer up top! I rerun the program, but this time entering “B”, which holds the value of 1056. It seems to be going up by intervals of 16. I repeat this process for lowercase letters and numbers, and symbols.

![1_D3tmJDNbFAS06G2yupbfow](https://user-images.githubusercontent.com/46263689/123755681-d396d800-d8c4-11eb-8265-caa623f74c8c.png)

Instead of placing every possible letter, number, and symbol into Exatlon and examining the memory location value of the R10 register, I wrote a quick Python script to return the values to me in a text file.

{% highlight python linenos %}

alphabet = 'ABCDEFGHIJKLMNOPQRSTUVWXYZ'
alphabetList = [char for char in alphabet]
lowerCase = []
numbers = [1,2,3,4,5,6,7,8,9]

for letter in alphabetList:
    lowerCase.append(letter.lower())

counter = 1024
output = "{} = {} \n"
result = open("result.txt", "w+")

for letter in alphabetList:
    counter += 16
    result.write(output.format(letter, counter))

counter = 1536
for letter in lowerCase:
    counter += 16
    result.write(output.format(letter, counter))

counter = 768
for number in numbers:
    counter += 16
    result.write(output.format(number, counter))

{% endhighlight %}

Adding 16 does not work for symbols, so those have to be done manually. You don’t need to do all of them, remember, the password will be in this format:

    HTB{flag_goes_here}
    
I was able to fill out the vast majority of the password after running my script, but there was still some blank spots where the symbols go. I went back to Exatlon in GDB and make a couple educated guesses to find/confirm the most symbols needed to find the correct password!


