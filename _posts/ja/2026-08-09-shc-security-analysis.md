---
layout: post
title: "シェルスクリプトのセキュリティ: shcの構造的限界と脆弱性(暗号化、コンパイラ、難読化)"
---

[shc](https://github.com/neurobin/shc)は、最も広く知られているシェルスクリプト保護ツールだ。  
シェルスクリプトをCソースコードでラップしてバイナリに変換する仕組みで、ソースコードの露出を防ぐ。

しかし、shcは構造的にシステムシェルに依存しているため、OSレベルのロギング/フッキング攻撃に弱い。

shcで生成されたバイナリは、シェルスクリプトをシステムシェル(例: /bin/sh)に渡して実行する。  
**したがって、システムシェルに引数を渡す瞬間だけを監視すれば、シェルスクリプトを簡単に窃取できる。**

![shcの脆弱性構造図](/assets/images/shc-security-analysis/1.png)

では、実際にテストして脆弱性を確認してみよう。

## テスト環境
Ubuntu 24.04で以下のシェルスクリプトを使用する。

![テスト用シェルスクリプト](/assets/images/shc-security-analysis/2.png)

以下のように[最高セキュリティレベル](https://github.com/neurobin/shc/blob/master/man.md)で進める。

```shell
shc -Uf launcher.sh -o shc_binary
```

## テスト方法
```shell
# Install auditd
sudo apt install auditd -y

# Register monitoring rule
sudo auditctl -a exit,always -F arch=b64 -S execve

# Run the shc binary
./shc_binary

# Check logs
sudo ausearch -i -sc execve | grep "shc_binary" -A 100
```

下の画像の赤枠のように、**auditd(OSレベルの監視ツール)を使えば、シェルスクリプトを簡単に窃取できる。**

![ターミナルに露出したシェルスクリプト](/assets/images/shc-security-analysis/3.png)


## 解決方法
システムシェルに依存しない保護ツールを使用する必要がある。  
システムシェルに依存する構造では、上記のようなOSレベルのロギング/フッキング攻撃を理論的に防ぐことはできない。

他にも[ssc](https://github.com/liberize/ssc)、[HimitsuShell](https://github.com/HimitsuShell/HimitsuShell)などの保護ツールが存在する。  
**しかし、sscにも致命的な構造的弱点がある。**

次の記事でこの内容を紹介する。