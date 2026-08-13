---
layout: post
title: "Linux Shell Script Security: Structural Limitations and Vulnerabilities in ssc (Source Code Protection, Obfuscation, Reverse Engineering)"
---

[ssc](https://github.com/liberize/ssc) is a project that improves on shc (shell script compiler).  
Like shc, it wraps a shell script in C source code and compiles it into a binary to keep the code from being exposed.

Unlike shc, ssc doesn't rely on the system shell — it uses a separate shell interpreter (e.g., BusyBox) instead.  
So [the technique used to attack shc (auditd)](https://himitsushell.github.io/en/shc-security-analysis/) can't be applied to ssc in the same way.

That said, ssc has a structural limitation of its own.  
A binary built with ssc briefly drops the embedded shell interpreter (e.g., BusyBox) to disk at /tmp/ssc.XXXXXX/busybox, then hands the shell script off to it to run.  
**So if you just watch for the moment the embedded shell interpreter gets exposed externally, you can capture the shell script.**

![ssc vulnerability diagram](/assets/images/ssc-security-analysis/1.png)

Let's test this directly and confirm the vulnerability.

## Test Environment
The following shell script was tested on Ubuntu 24.04.

![Test shell script](/assets/images/ssc-security-analysis/2.png)

We'll build the binary with the [shell interpreter (BusyBox) embedded](https://github.com/liberize/ssc/tree/master/examples/4_embed_interpreter), as shown below.

```shell
./ssc test ssc_binary -s -r -e busybox -c
```

## Test Method
```shell
# Install bpftrace
sudo apt install bpftrace

# Start monitoring in terminal 1
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_write /comm == "ssc_binary"/ { printf("PID: %d | FD: %d | Data: %s\n", pid, args->fd, str(args->buf)); }'

# Run ssc_binary in terminal 2
./ssc_binary
```

**As shown in the red box in the image below, bpftrace (a kernel tracing tool) makes it easy to capture the shell script.**

![Terminal 2 monitoring result](/assets/images/ssc-security-analysis/3.png)

![Terminal 1 monitoring result](/assets/images/ssc-security-analysis/4.png)

## Solution
What's needed is a protection tool that keeps the shell interpreter entirely internal instead of exposing it externally.  
A representative example is [HimitsuShell](https://github.com/HimitsuShell/HimitsuShell).

The next post will introduce HimitsuShell.