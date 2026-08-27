---
layout: post
title:  "Tomb Raider Reboot"
date:   2026-08-27 14:49:04 +0200
categories: jekyll update
---

# Running TR2_x64_release.exe

Fetch the executable from [here](https://debugging.games/_files/Windows/[WIN]%20Rise%20of%20the%20Tomb%20Raider%20[2021-10-22]%20(PDB).7z)

First patch the executable. Edit the HasValidMainUser function (offset = 0xe9bc40) to always return true. 

![Image](/assets/TombRaider/images/HasValidMainUserAsm.png)

Then run the executable with the following arguments: `-archive -norootchange -mainmenu -noassert -launcher`