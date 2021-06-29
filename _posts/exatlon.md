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



