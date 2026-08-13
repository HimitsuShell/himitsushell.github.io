---
layout: post
title: "Shell 脚本保护工具对比：shc 与 HimitsuShell（二进制化、加密、混淆）"
---

[shc](https://github.com/neurobin/shc/tree/master)（Shell 脚本编译器）是一款将 Shell 脚本转换为二进制文件、防止代码泄露的工具。但在实际使用中存在以下局限性。
- 不提供混淆功能，容易被逆向工程破解。
- 依赖系统 Shell（如 /bin/sh），容易受到日志记录/挂钩（logging/hooking）攻击。

[HimitsuShell](https://github.com/HimitsuShell/HimitsuShell) 正是为弥补这些不足而开发的。它应用了基于 LLVM 的多种混淆技术，并且不依赖系统 Shell。

下面从以下几个维度进行对比。

## 测试环境
在 Ubuntu 24.04 环境下使用以下 Shell 脚本。

![测试用 Shell 脚本](/assets/images/shc-vs-himitsushell/1.png)

HimitsuShell 使用默认选项，[shc 则使用以下命令，以最高安全强度进行编译测试](https://github.com/neurobin/shc/blob/master/man.md)：

```shell
shc -Uf launcher.sh -o shc_binary
```

如下所示，HimitsuShell 和 shc 生成的二进制文件均能正常运行。

![二进制文件运行检测](/assets/images/shc-vs-himitsushell/2.png)


## 移除调试符号
![调试符号检测](/assets/images/shc-vs-himitsushell/3.png)

HimitsuShell 和 shc 都移除了调试符号。

## 动态库挂钩防护
![动态库检测](/assets/images/shc-vs-himitsushell/4.png)

HimitsuShell 不依赖动态库，而 shc 则依赖动态库。  
因此 shc 容易受到挂钩攻击。

## 调试器检测
![调试器检测 1](/assets/images/shc-vs-himitsushell/5.png)

![调试器检测 2](/assets/images/shc-vs-himitsushell/6.png)

HimitsuShell 和 shc 都能检测并阻止调试器（如 gdb、strace 等）。

## 字符串、常量混淆
![字符串、常量混淆检测 1](/assets/images/shc-vs-himitsushell/7.png)

![字符串、常量混淆检测 2](/assets/images/shc-vs-himitsushell/8.png)

在 Ghidra 中提取字符串列表可以清楚地看出两者的差异。  
HimitsuShell 对字符串进行了混淆处理，而 shc 生成的二进制文件中，敏感字符串仍以明文形式暴露。  
此外，HimitsuShell 提供常量混淆功能，而 shc 并不提供。

## 高级混淆（控制流平坦化、虚假代码插入等）
![高级混淆检测 1](/assets/images/shc-vs-himitsushell/9.png)

![高级混淆检测 2](/assets/images/shc-vs-himitsushell/10.png)

在 Ghidra 中查看 main 函数的控制流图，差异一目了然。  
HimitsuShell 经过混淆处理后，控制流几乎无法被识别。  
相反，shc 的控制流较为简单，容易被分析。

## 操作系统层面的日志记录/挂钩防护
![日志记录/挂钩检测 1](/assets/images/shc-vs-himitsushell/11.png)

使用 auditd 监控 shc 生成的二进制文件的系统调用（如 execve）时，可以看到 Shell 脚本内容被完整记录下来。  
由于其结构依赖系统 Shell，因此容易受到操作系统层面的日志记录/挂钩攻击。  

![日志记录/挂钩检测 2](/assets/images/shc-vs-himitsushell/12.png)

相反，在相同条件下，HimitsuShell 生成的二进制文件不会记录 Shell 脚本内容。  
这是因为它不依赖系统 Shell，而是使用内置在二进制文件中的 Shell。

## 结论
由此可见，shc 和 HimitsuShell 在安全强度上存在明显差异。

| |shc|HimitsuShell|
|---|---|---|
|移除调试符号|✓|✓|
|动态库挂钩防护| |✓|
|调试器检测|✓|✓|
|字符串、常量混淆| |✓|
|高级混淆（控制流平坦化、虚假代码插入等）| |✓|
|操作系统层面的日志记录/挂钩防护| |✓|

## 补充对比
还有一个对 shc 进行改进的项目，叫 [ssc](https://github.com/liberize/ssc)。  
它在部分方面有所改善（如动态库挂钩防护、字符串混淆），但在高级混淆和操作系统层面的日志记录/挂钩防护方面仍存在局限性。

**尤其是 ssc 虽然不依赖系统 Shell，但运行时会将解释器（如 /bin/sh 等）提取到 /tmp/ssc/XXXXXX 路径下，并通过它传递 Shell 脚本。因此，只要监控或挂钩该路径下解释器的执行过程，就有可能窃取 Shell 脚本内容。**

相比之下，HimitsuShell 不会将解释器提取到二进制文件之外，因此更为安全。