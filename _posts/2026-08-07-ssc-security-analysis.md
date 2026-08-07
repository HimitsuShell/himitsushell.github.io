---
layout: post
title: "ssc 安全漏洞分析：Shell 脚本编译器的结构性局限"
date: 2026-08-07 22:30:00 +0900
---

[ssc](https://github.com/liberize/ssc) 是在 shc(Shell 脚本编译器)基础上改进而来的项目。  
其原理与 shc 相同，都是先将 Shell 脚本包装成 C 源代码，再编译为二进制文件，从而防止源码暴露。

与 shc 不同的是，ssc 不依赖系统 Shell，而是使用独立的 Shell 解释器(例如 BusyBox)。  
因此，[攻击 shc 时所使用的技术(auditd)](https://himitsushell.github.io/2026/08/07/shc-security-analysis.html)在 ssc 上无法同样奏效。

但 ssc 同样存在结构性的局限。  
由 ssc 生成的二进制文件，会将内置的 Shell 解释器(例如 BusyBox)临时导出到 /tmp/ssc.XXXXXX/busybox 路径下，再将 Shell 脚本传递到该路径执行。  
**因此，只要监控内置 Shell 解释器暴露到外部的那一瞬间，就能窃取到 Shell 脚本。**

![漏洞展开图](/assets/images/ssc-security-analysis/1.png)

现在让我们直接测试一下，确认这个漏洞。

## 测试环境
在 ubuntu 24.04 环境下，使用如下 Shell 脚本。

![测试用 Shell 脚本](/assets/images/ssc-security-analysis/2.png)

下面我们通过[内置 Shell 解释器(BusyBox)](https://github.com/liberize/ssc/tree/master/examples/4_embed_interpreter)的方式进行测试。

```shell
./ssc test ssc_binary -s -r -e busybox -c
```

## 测试方法

```shell
# 安装 bpftrace
sudo apt install bpftrace

# 在终端1中开始监控
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_write /comm == "ssc_binary"/ { printf("PID: %d | FD: %d | Data: %s\n", pid, args->fd, str(args->buf)); }'

# 在终端2中执行 ssc_binary
./ssc_binary
```

**如下图红框所示，使用 bpftrace(内核监控工具)即可轻松窃取 Shell 脚本。**

![终端2](/assets/images/ssc-security-analysis/3.png)

![终端1](/assets/images/ssc-security-analysis/4.png)

## 解决方法
应当使用不将 Shell 解释器导出到外部、仅在内部进行处理的防护工具。  
具有代表性的例子是 [HimitsuShell](https://github.com/HimitsuShell/HimitsuShell)。

下一篇文章将介绍 HimitsuShell。