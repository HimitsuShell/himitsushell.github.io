---
layout: post
title: "Comparing Shell Script Protection Tools: shc vs HimitsuShell (Binary Compilation, Encryption, and Obfuscation)"
---

[shc](https://github.com/neurobin/shc/tree/master) (a shell script compiler) is a tool that converts shell scripts into binaries to prevent source code exposure. However, it has the following limitations in real-world use:
- It provides no obfuscation, which leaves it vulnerable to reverse engineering.
- It depends on the system shell (e.g., /bin/sh), which leaves it vulnerable to logging/hooking attacks.

[HimitsuShell](https://github.com/HimitsuShell/HimitsuShell) was built to address these gaps. It applies a variety of LLVM-based obfuscation techniques and doesn't depend on the system shell either.

Now let's walk through the comparison item by item.

## Test Environment
The following shell script was tested on Ubuntu 24.04.

![Test shell script](/assets/images/shc-vs-himitsushell/1.png)

HimitsuShell is run with its default options, while shc is run at [maximum security level](https://github.com/neurobin/shc/blob/master/man.md) as shown below.

```shell
shc -Uf launcher.sh -o shc_binary
```

As shown below, the binaries produced by both HimitsuShell and shc run correctly.

![Binary execution test results](/assets/images/shc-vs-himitsushell/2.png)

## Debug Symbol Stripping

![Debug symbols removed](/assets/images/shc-vs-himitsushell/3.png)

Both HimitsuShell and shc strip debug symbols.

## Protection against Dynamic Library Hooking

![Dynamic library lookup results](/assets/images/shc-vs-himitsushell/4.png)

HimitsuShell doesn't rely on dynamic libraries, but shc does.  
As a result, shc is vulnerable to hooking attacks.

## Debugger Detection

![Debugger detection verification results 1](/assets/images/shc-vs-himitsushell/5.png)

![Debugger detection verification results 2](/assets/images/shc-vs-himitsushell/6.png)

Both HimitsuShell and shc detect and block debuggers (e.g., gdb, strace).

## String and Constant Obfuscation

![String obfuscation verification results 1](/assets/images/shc-vs-himitsushell/7.png)

![String obfuscation verification results 1](/assets/images/shc-vs-himitsushell/8.png)

Looking at the string list in Ghidra makes the difference obvious.  
HimitsuShell's strings are obfuscated, while binaries built with shc expose sensitive strings as plain text.  
HimitsuShell also provides constant obfuscation, while shc does not.

## Advanced Obfuscation (Control Flow Flattening, Dead Code Injection, etc.)

![Control flow graph results 1](/assets/images/shc-vs-himitsushell/9.png)

![Control flow graph results 2](/assets/images/shc-vs-himitsushell/10.png)

Looking at the control flow graph of the main function in Ghidra, the difference is clear.  
HimitsuShell is obfuscated to the point where tracing the control flow is essentially impossible.  
shc, on the other hand, has a simple control flow that's easy to analyze.

## Protection against OS-level Logging and Hooking

![auditd verification results 1](/assets/images/shc-vs-himitsushell/11.png)

If you monitor the system calls (e.g., execve) of a binary built with shc using auditd, the shell script shows up in the logs in plain text.  
Because it relies on the system shell, it's vulnerable to OS-level logging/hooking attacks.

![auditd verification results 2](/assets/images/shc-vs-himitsushell/12.png)

In contrast, a binary built with HimitsuShell shows no shell script logging under the same conditions.  
That's because it doesn't rely on the system shell — it uses a shell embedded in the binary itself.

## Conclusion
As shown above, there's a clear difference in security strength between shc and HimitsuShell.

||shc|HimitsuShell|
|---|---|---|
|Debug Symbol Stripping|✓|✓|
|Protection against Dynamic Library Hooking| |✓|
|Debugger detection|✓|✓|
|String and Constant Obfuscation| |✓|
|Advanced Obfuscation (Control Flow Flattening, Dead Code Injection, etc.)| |✓|
|Protection against OS-level Logging and Hooking| |✓|

## Additional Comparison
There's also [ssc](https://github.com/liberize/ssc), which improves on shc.  
It addresses some of the issues (protection against dynamic library hooking, string obfuscation), but still has limitations around advanced obfuscation and protection against OS-level logging/hooking.

**Notably, while ssc doesn't depend on the system shell, at runtime it extracts the interpreter (e.g., /bin/sh) to the path /tmp/ssc/XXXXXX and passes the shell script to it. This means logging or hooking that path leaves the shell script vulnerable to extraction.**

HimitsuShell, on the other hand, never extracts the interpreter outside the binary, so it's safe from this issue.