---
layout: post
title: "Linuxシェルスクリプトのセキュリティ:sscの構造的限界と脆弱性(ソースコード保護、難読化、リバースエンジニアリング)"
---

[ssc](https://github.com/liberize/ssc)は、shc(シェルスクリプトコンパイラ)を改良したプロジェクトである。  
shcと同じ原理で、シェルスクリプトをCのソースコードでラップした後にバイナリへ変換し、コードの露出を防ぐ。

sscはshcとは異なり、システムシェルに依存せず、独自のシェルインタプリタ(例:BusyBox)を使用する。  
そのため、[shcを攻撃する際に使用した手法(auditd)](https://himitsushell.github.io/ja/shc-security-analysis/)は、sscでは同様に使用することができない。

しかし、sscもまた構造的な限界を抱えている。  
sscによって生成されたバイナリは、内蔵されたシェルインタプリタ(例:BusyBox)を一時的に/tmp/ssc.XXXXXX/busyboxのパスに展開し、そこにシェルスクリプトを渡して実行する。  
**したがって、内蔵されたシェルインタプリタが外部に露出する瞬間だけを監視すれば、シェルスクリプトを窃取することができる。**

![sscの脆弱性の構造図](/assets/images/ssc-security-analysis/1.png)

それでは実際にテストして脆弱性を確認してみよう

## テスト環境
ubuntu 24.04で、以下のシェルスクリプトを使用する。

![テスト用シェルスクリプト](/assets/images/ssc-security-analysis/2.png)

以下のように[シェルインタプリタ(BusyBox)を内蔵](https://github.com/liberize/ssc/tree/master/examples/4_embed_interpreter)させて進める。

```shell
./ssc test ssc_binary -s -r -e busybox -c
```

## テスト方法
```shell
# Install bpftrace
sudo apt install bpftrace

# Start monitoring in terminal 1
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_write /comm == "ssc_binary"/ { printf("PID: %d | FD: %d | Data: %s\n", pid, args->fd, str(args->buf)); }'

# Run ssc_binary in terminal 2
./ssc_binary
```

**以下の写真の赤枠のように、bpftrace(カーネル監視ツール)を使用すれば、容易にシェルスクリプトを窃取することができる。**

![ターミナル2の監視結果](/assets/images/ssc-security-analysis/3.png)

![ターミナル1の監視結果](/assets/images/ssc-security-analysis/4.png)

## 解決方法
シェルインタプリタを外部に抽出せず、内部でのみ処理する保護ツールを使用しなければならない。  
代表的な例として[HimitsuShell](https://github.com/HimitsuShell/HimitsuShell)がある。

次回の記事でHimitsuShellを紹介する。