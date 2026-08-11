---
layout: post
title: "How to Protect Source Code in Docker Images and Containers (Python, C/C++, Shell Scripts, LLVM Obfuscation, DRM)"
---

By default, anyone can view and use the source code and executables inside a Docker image.  
There are a few ways to keep others from viewing the source code inside a Docker image and to restrict its use to authorized users only.

## 1. Multi-Stage Builds
This is a built-in image-building feature provided by Docker.  
You can select specific files from a previous stage and copy only those into the current stage.  
Files you don't specify aren't included in the final image, which reduces image size and keeps unnecessary files from being exposed to others.

```shell
FROM ubuntu:24.04 AS builder
WORKDIR /var/work
RUN touch secret.txt public.txt

FROM ubuntu:24.04
WORKDIR /var/work
# Copy only public.txt
COPY --from=builder /var/work/public.txt .
```

## 2. Obfuscation and DRM by File Type
As shown above, multi-stage builds let you hide a large portion of the files that would otherwise be exposed inside a Docker image.  
However, because of how multi-stage builds work, any files included in the final stage remain intact.  
For the final stage, you need to apply separate obfuscation and DRM techniques suited to each type of file (shell scripts, C/C++ programs, etc.).

### 2.1 Shell Script Obfuscation and DRM
The most widely known tool for protecting shell scripts is shc.  
However, as analyzed in [a previous post](https://himitsushell.github.io/en/shc-security-analysis/), shc has a critical vulnerability.  
Let's use HimitsuShell, which fixes shc's vulnerability.

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

### 2.2 Python Obfuscation and DRM
The most widely known tool for protecting Python code is [Pyarmor](https://github.com/dashingsoft/pyarmor).  
Pyarmor provides features such as obfuscation and expiration-date settings.

```shell
# Install Pyarmor
pip install pyarmor

# Obfuscate the Python script
pyarmor gen foo.py

# Run the obfuscated script
python dist/foo.py
```

### 2.3 C/C++, Objective-C/C++, Swift, Rust, Zig, Kotlin/Native, etc.
LLVM-based obfuscation tools let you obfuscate a wide range of languages — C/C++, Objective-C, and more — to a level comparable to commercial tools like VMProtect and Enigma Protector.  
Let's try HimitsuObfuscator using C/C++ code as an example.

|Obfuscation Tool|LLVM Version|License|Commercial Use|Maintenance|Notable Points|
|---|---|---|---|---|---|
|OLLVM|4|UIUC/NCSA|Allowed|Discontinued in 2017|The starting point of LLVM-based obfuscation tools|
|Hikari Obfuscator|8|Modified AGPLv3|Allowed with restrictions|Discontinued in 2020|Applied more advanced obfuscation techniques after OLLVM|
|Himitsu Obfuscator|17|MIT|Allowed|Actively maintained|Fixed OLLVM bugs, expanded obfuscation coverage|

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

## Conclusion
Applying the methods above in layers can block most source code theft attempts by beginners, like script kiddies.  
For a skilled reverse engineer, though, these methods only slow down and complicate the attack — they don't make it impossible.  
So on top of the methods above, you'll need to keep applying various stronger techniques as well.

I'll cover additional methods in the next post.