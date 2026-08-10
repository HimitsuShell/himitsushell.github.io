---
layout: post
title: "도커 이미지·컨테이너 소스코드 보호 방법 (Python, C/C++, 쉘 스크립트, LLVM 난독화, DRM)"
date: 2026-08-10 12:45:00 +0900
lang: ko
categories: [ko]
---

도커 이미지 안에 있는 소스코드와 실행 파일은 기본적으로 누구나 보고 사용할 수 있습니다.  
다른 사람이 도커 이미지 안의 소스코드를 볼 수 없게 하고, 허가된 사람만 사용할 수 있도록 하는 몇 가지 방법이 있습니다.

## 1. 멀티 스테이지 빌드
도커에서 기본적으로 제공하는 이미지 빌드 기능입니다.  
이전 스테이지의 특정 파일만 선택하여, 현재 레이어로 복사할 수 있습니다.  
지정되지 않은 파일은 최종 이미지에 포함되지 않아, 이미지 크기를 줄이고 불필요한 파일이 다른 사람에게 노출되는것을 막을 수 있습니다.  

```shell
FROM ubuntu:24.04 AS builder
WORKDIR /var/work
RUN touch secret.txt public.txt

FROM ubuntu:24.04
WORKDIR /var/work
# Copy only public.txt
COPY --from=builder /var/work/public.txt .
```

## 2. 파일 형태별 난독화, DRM 적용
위와 같이 멀티 스테이지 빌드를 사용하면, 도커 이미지 안에서 노출되는 상당수의 파일을 숨길 수 있습니다.  
하지만 멀티 스테이지 빌드 구조상 최종 스테이지에 포함된 파일은 그대로 남아있습니다.  
최종 스테이지에서는 각 파일(쉘 스크립트, C/C++ 프로그램 등)의 형태에 맞는 별도의 난독화 및 DRM 기술을 적용해야 합니다.

### 2.1 쉘 스크립트 난독화, DRM 적용
쉘 스크립트 보호 도구 중 가장 널리 알려진 것은 shc입니다.  
하지만 shc는 [이전 글](https://himitsushell.github.io/ko/2026/08/10/shc-security-analysis-ko/)에서 분석한 것처럼 치명적인 취약점이 존재합니다.  
shc의 취약점을 보완한 HimitsuShell을 사용해보겠습니다.

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

### 2.2 Python 난독화, DRM 적용
Python 보호 도구 중 가장 널리 알려진 것은 [Pyarmor](https://github.com/dashingsoft/pyarmor)입니다.  
Pyarmor는 난독화, 만료일 설정 등의 기능을 제공합니다.

```shell
# Install Pyarmor
pip install pyarmor

# Obfuscate the Python script
pyarmor gen foo.py

# Run the obfuscated script
python dist/foo.py
```

### 2.3 C/C++, Objective-C/C++ , Swift, Rust, Zig, Kotlin/Native 등
LLVM 기반의 난독화 도구를 사용하면 C/C++, Objective-C 등 다양한 언어를 상용 도구(VMProtect, Enigma Protector 등) 수준으로 난독화할 수 있습니다.  
C/C++ 코드를 예시로 HimitsuObfuscator를 사용해보겠습니다.

|난독화 도구|LLVM 버전|라이선스|상업적 목적|유지보수|주목할 점|
|---|---|---|---|---|---|
|OLLVM|4|UIUC/NCSA|허용|2017년 중단|LLVM 난독화 도구의 시작점|
|Hikari Obfuscator|8|변형된 AGPLv3|제한적 허용|2020년 중단|OLLVM 이후 선진적 난독화 기법 적용|
|Himitsu Obfuscator|17|MIT|허용|유지 중|OLLVM 버그 개선, 난독화 범위 확장|

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

## 결론
위 방법들을 다층적으로 적용하면, 스크립트 키디와 같은 초보자에 의한 소스코드 탈취는 대부분 막아낼 수 있습니다.  
하지만 실력 있는 리버스 엔지니어에게는 공격을 지연시키고 까다롭게 만들 뿐, 불가능하게 만들지는 못합니다.  
따라서 위 방법에 더해 다양한 고강도 기술을 지속적으로 적용해야만 합니다.

다음 글에서 추가적인 방법을 소개하겠습니다.