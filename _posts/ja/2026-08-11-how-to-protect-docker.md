---
layout: post
title: "Docker イメージ・コンテナのソースコード保護方法(Python、C/C++、シェルスクリプト、LLVM難読化、DRM)"
---

Dockerイメージの中にあるソースコードや実行ファイルは、基本的に誰でも見て使うことができます。  
他人がDockerイメージ内のソースコードを見られないようにし、許可されたユーザーだけが使用できるようにするいくつかの方法があります。

## 1. マルチステージビルド
Dockerが標準で提供しているイメージビルド機能です。  
前のステージから特定のファイルだけを選んで、現在のステージにコピーすることができます。  
指定していないファイルは最終イメージに含まれないため、イメージサイズを削減できるだけでなく、不要なファイルが他人に露出するのを防ぐこともできます。  

```shell
FROM ubuntu:24.04 AS builder
WORKDIR /var/work
RUN touch secret.txt public.txt

FROM ubuntu:24.04
WORKDIR /var/work
# Copy only public.txt
COPY --from=builder /var/work/public.txt .
```

## 2. ファイル形式ごとの難読化・DRM適用
上記のようにマルチステージビルドを使えば、Dockerイメージ内で露出するファイルの多くを隠すことができます。  
しかし、マルチステージビルドの構造上、最終ステージに含まれるファイルはそのまま残ってしまいます。  
最終ステージでは、各ファイル(シェルスクリプト、C/C++プログラムなど)の形式に合わせて、個別の難読化やDRM技術を適用する必要があります。

### 2.1 シェルスクリプトの難読化・DRM適用
シェルスクリプト保護ツールの中で最も広く知られているのはshcです。  
しかしshcには、[前回の記事](https://himitsushell.github.io/ja/shc-security-analysis/)で分析したように致命的な脆弱性が存在します。  
shcの脆弱性を補ったHimitsuShellを使ってみましょう。

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

### 2.2 Pythonの難読化・DRM適用
Python保護ツールの中で最も広く知られているのは[Pyarmor](https://github.com/dashingsoft/pyarmor)です。  
Pyarmorは難読化や有効期限の設定などの機能を提供します。

```shell
# Install Pyarmor
pip install pyarmor

# Obfuscate the Python script
pyarmor gen foo.py

# Run the obfuscated script
python dist/foo.py
```

### 2.3 C/C++、Objective-C/C++、Swift、Rust、Zig、Kotlin/Nativeなど
LLVMベースの難読化ツールを使えば、C/C++やObjective-Cなど多様な言語を、商用ツール(VMProtect、Enigma Protectorなど)と同等のレベルまで難読化することができます。  
C/C++コードを例に、HimitsuObfuscatorを使ってみましょう。

|難読化ツール|LLVMバージョン|ライセンス|商用利用|メンテナンス|特徴|
|---|---|---|---|---|---|
|OLLVM|4|UIUC/NCSA|許可|2017年に開発停止|LLVM難読化ツールの出発点|
|Hikari Obfuscator|8|改変されたAGPLv3|制限付きで許可|2020年に開発停止|OLLVM以降の先進的な難読化技術を適用|
|Himitsu Obfuscator|17|MIT|許可|継続中|OLLVMのバグ修正、難読化範囲の拡張|

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

## まとめ
上記の方法を多層的に適用すれば、スクリプトキディのような初心者によるソースコードの窃取はほとんど防ぐことができます。  
しかし、実力のあるリバースエンジニアに対しては、攻撃を遅らせ、困難にするだけであり、不可能にすることはできません。  
したがって、上記の方法に加えて、様々な高強度の技術を継続的に適用していく必要があります。

次回の記事で、さらに他の方法をご紹介します。