---
layout: post
title: "Docker 镜像与容器源代码保护方法（Python、C/C++、Shell 脚本、LLVM 混淆、DRM）"
date: 2026-08-08 10:30:00 +0900
lang: zh
---
Docker 镜像中的源代码和可执行文件，默认情况下任何人都可以查看和使用。  
不过有几种方法可以防止他人查看 Docker 镜像中的源代码，让只有获得授权的人才能使用。

## 1. 多阶段构建(Multi-stage build)
这是 Docker 默认提供的镜像构建功能。  
可以只选取前一构建阶段中的特定文件，复制到当前阶段中。  
未被指定的文件不会包含在最终镜像里，这样既能减小镜像体积，又能防止不必要的文件暴露给他人。

```shell
FROM ubuntu:24.04 AS builder
WORKDIR /var/work
RUN touch secret.txt public.txt

FROM ubuntu:24.04
WORKDIR /var/work
# Copy only public.txt
COPY --from=builder /var/work/public.txt .
```

## 2. 按文件类型进行混淆、应用 DRM
如上所述，使用多阶段构建可以隐藏 Docker 镜像中大部分会暴露出来的文件。  
但由于多阶段构建的结构特性，最终层中包含的文件仍会原样保留下来。  
因此需要针对最终层中每种文件(Shell 脚本、C/C++ 程序等)各自的形式，分别应用相应的混淆和 DRM 技术。

### 2.1 Shell 脚本混淆与 DRM 应用

在 Shell 脚本保护工具中，最广为人知的是 shc。  
但正如[上一篇文章](https://himitsushell.github.io/zh/2026/08/07/shc-security-analysis-zh/)中分析的那样，shc 存在致命的安全漏洞。  
接下来我们使用弥补了 shc 漏洞的 HimitsuShell 来试一下。

```shell
# 1. download and load docker image
curl -LO https://github.com/HimitsuShell/Himitsu/releases/download/v1.2.0/himitsu_core_v1.2.0.tar.gz
docker load -i himitsu_core_v1.2.0.tar.gz

# 2. start container
docker run --name himitsu_core -d -it himitsu_core:v1.2.0

# 3. upload your shell script (must be named launcher.sh)
docker cp launcher.sh himitsu_core:/var/work/

# 4. build and download binary (10–20 seconds)
docker exec himitsu_core /var/work/compile.sh
docker cp himitsu_core:/var/work/safeLauncher .

# 5. Run the obfuscated binary
chmod +x ./safeLauncher
./safeLauncher
```

### 2.2 Python 混淆与 DRM 应用

在 Python 保护工具中，最广为人知的是 Pyarmor。  
Pyarmor 提供代码混淆、设置过期日期等功能。

```shell
# Install Pyarmor
pip install pyarmor

# Obfuscate the Python script
pyarmor gen foo.py

# Run the obfuscated script
python dist/foo.py
```

### 2.3 C/C++、Objective-C/C++、Swift、Rust、Zig、Kotlin/Native 等

使用基于 LLVM 的混淆工具，可以让 C/C++、Objective-C 等多种语言的混淆强度达到商业级保护工具(VMProtect、Enigma Protector 等)的水平。  
下面以 C/C++ 代码为例，使用 HimitsuObfuscator 来演示一下。

|混淆工具|LLVM 版本|许可证|商业用途|维护状态|值得关注的点|
|---|---|---|---|---|---|
|OLLVM|4|UIUC/NCSA|允许|2017 年停止维护|LLVM 混淆工具的起点|
|Hikari Obfuscator|8|修改版 AGPLv3|有限制地允许|2020 年停止维护|在 OLLVM 之后采用了更先进的混淆技术|
|Himitsu Obfuscator|17|MIT|允许|持续维护中|修复了 OLLVM 的缺陷，扩大了混淆范围|

```shell
# Required: Ubuntu 24.04

# download and extract obfuscator
curl -LO https://github.com/HimitsuShell/HimitsuObfuscator/releases/download/v1.2.0_0/himitsu_obfuscator_v1.2.0_0.tar
tar -xvf himitsu_obfuscator_v1.2.0_0.tar

vim main.c
-----------------------------
#include <stdio.h>
int main() {
  printf("Hello World!\n");
  return 0;
}
-----------------------------

# builds a binary that runs on any linux (static musl)
sudo apt-get install -y build-essential
./compiler/bin/x86_64-unknown-linux-musl-clang -flto -fuse-ld=lld -mllvm -sobf -mllvm -sub -static main.c -o main
./main
```

## 总结
综合运用上述方法，基本可以防范像脚本小子(script kiddie)这类新手对源代码的窃取。  
但对于技术娴熟的逆向工程师来说，这些方法只能增加逆向难度、延缓破解进度，并不能使其变得不可能。  
因此，除了上述方法之外，还必须持续应用各种更高强度的技术。

我们将在下一篇文章中介绍更多方法。