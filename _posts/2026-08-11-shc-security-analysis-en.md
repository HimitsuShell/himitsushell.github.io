---
layout: post
title: "Shell Script Security: Structural Limitations and Vulnerabilities of shc (Encryption, Compiler, Obfuscation)"
date: 2026-08-11 05:00:00 +0900
lang: en
categories: [en]
---

[shc](https://github.com/neurobin/shc) is the most widely known shell script protection tool.  
It works by wrapping a shell script in C source code and compiling it into a binary, which keeps the source code from being exposed.

However, because shc is structurally dependent on the system shell, it's vulnerable to OS-level logging/hooking attacks.

A binary built with shc runs the shell script by passing it to the system shell (e.g., /bin/sh) for execution.  
**So all you have to do is monitor the moment the arguments are passed to the system shell, and you can easily capture the shell script.**

![shc vulnerability structure diagram](/assets/images/shc-security-analysis/1.png)

Let's actually test this and confirm the vulnerability.

## Test Environment
The following shell script is used on ubuntu 24.04.

![Shell script used for testing](/assets/images/shc-security-analysis/2.png)

Proceed with [maximum security strength](https://github.com/neurobin/shc/blob/master/man.md) as shown below.

```shell
shc -Uf launcher.sh -o shc_binary
```

## Test Method
```shell
# Install auditd
sudo apt install auditd -y

# Register monitoring rule
sudo auditctl -a exit,always -F arch=b64 -S execve

# Run the shc binary
./shc_binary

# Check logs
sudo ausearch -i -sc execve | grep "shc_binary" -A 100
```

As shown in the red box in the image below, **using auditd (an OS-level monitoring tool) makes it trivially easy to capture the shell script**.

![Shell script exposed in the terminal](/assets/images/shc-security-analysis/3.png)

## Solution
You need to use a protection tool that doesn't depend on the system shell.  
Any tool built on a structure that relies on the system shell is fundamentally unable to defend against OS-level logging/hooking attacks like the one shown above.

There are protection tools out there such as [ssc](https://github.com/liberize/ssc) and [HimitsuShell](https://github.com/HimitsuShell/HimitsuShell).  
**But ssc also has a critical structural weakness.**

I'll cover that in the next post.