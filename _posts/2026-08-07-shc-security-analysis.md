---
layout: post
title: "shc 安全漏洞分析：Shell 脚本编译器的结构性局限"
---
[shc](https://github.com/neurobin/shc) 是目前最广为人知的 Shell 脚本保护工具。  
它的原理是把 Shell 脚本包装成 C 源代码后编译成二进制文件，以此防止源代码泄露。

但 shc 在结构上依赖系统 Shell，因此容易受到操作系统层面的日志记录/hook 攻击。

shc 生成的二进制文件会把 Shell 脚本传给系统 Shell（例如 /bin/sh）来执行。  
**所以，只要在参数传递给系统 Shell 的那一瞬间进行监控，就能轻松窃取 Shell 脚本内容。**

![漏洞展开图](/assets/images/shc-security-analysis/1.png)

现在我们直接动手测试，复现这个漏洞。

## 测试环境
在 Ubuntu 24.04 上使用下面的 Shell 脚本。

![测试用 Shell 脚本](/assets/images/shc-security-analysis/2.png)

以[最高安全强度](https://github.com/neurobin/shc/blob/master/man.md)进行测试，如下所示：

```shell
shc -Uf launcher.sh -o shc_binary
```

## 测试方法
```shell
# 安装 auditd
sudo apt install auditd -y

# 注册监控规则
sudo auditctl -a exit,always -F arch=b64 -S execve

# 运行 shc_binary
./shc_binary

# 查看日志
sudo ausearch -i -sc execve | grep "shc_binary" -A 100
```

如下图红框所示，**使用 auditd（操作系统层面的监控工具）就能轻松窃取 Shell 脚本。**

![暴露的 Shell 脚本](/assets/images/shc-security-analysis/3.png)

## 解决方法
应该使用不依赖系统 Shell 的保护工具。  
只要架构上依赖系统 Shell，理论上就无法防御上述操作系统层面的日志记录/hook 攻击。

市面上存在 [ssc](https://github.com/liberize/ssc)、[HimitsuShell](https://github.com/HimitsuShell/HimitsuShell) 等保护工具。  
**但 ssc 同样存在致命的结构性缺陷。**

这部分内容将在下一篇文章中介绍。