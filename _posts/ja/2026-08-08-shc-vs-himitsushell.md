---
layout: post
title: "シェルスクリプト保護ツール比較:shc vs HimitsuShell(バイナリ化、暗号化、難読化)"
---

[shc](https://github.com/neurobin/shc/tree/master)(シェルスクリプトコンパイラ)は、シェルスクリプトをバイナリに変換してコード漏洩を防ぐツールだ。しかし、実際の運用には次のような限界がある。
- 難読化機能を提供しないため、リバースエンジニアリングに対して脆弱だ。
- システムのシェル(例: /bin/sh)に依存しているため、ロギング/フッキング攻撃に対して脆弱だ。

[HimitsuShell](https://github.com/HimitsuShell/HimitsuShell)は、これを補うために作られた。LLVMベースの様々な難読化技法が適用されており、システムのシェルにも依存しない。

それでは項目別に比較してみよう。

## テスト環境
Ubuntu 24.04で以下のシェルスクリプトを使用する。

![テスト用シェルスクリプト](/assets/images/shc-vs-himitsushell/1.png)

HimitsuShellはデフォルトオプションで、shcは以下のように[最大セキュリティ強度](https://github.com/neurobin/shc/blob/master/man.md)でコンパイルする。

```shell
shc -Uf launcher.sh -o shc_binary
```

以下のように、HimitsuShellとshcで生成したバイナリはどちらも正常に動作する。

![バイナリ動作テスト結果](/assets/images/shc-vs-himitsushell/2.png)

## デバッグシンボルの削除

![削除されたデバッグシンボル](/assets/images/shc-vs-himitsushell/3.png)

HimitsuShellとshcはどちらもデバッグシンボルが削除されている。

## 動的ライブラリフッキング防御

![動的ライブラリ確認結果](/assets/images/shc-vs-himitsushell/4.png)

HimitsuShellは動的ライブラリに依存しないが、shcは依存する。  
そのため、shcはフッキング攻撃に対して脆弱だ。

## デバッガ検知

![デバッガ検知検証結果1](/assets/images/shc-vs-himitsushell/5.png)

![デバッガ検知検証結果2](/assets/images/shc-vs-himitsushell/6.png)

HimitsuShellとshcはどちらもデバッガ(例: gdb、strace等)を検知して遮断する。

## 文字列・定数の難読化

![文字列難読化検証結果1](/assets/images/shc-vs-himitsushell/7.png)

![文字列難読化検証結果1](/assets/images/shc-vs-himitsushell/8.png)

Ghidraで文字列一覧を抽出してみると、その差は明確だ。  
HimitsuShellは文字列が難読化されているが、shcで作成したバイナリは機密性の高い文字列がそのまま露出する。  
また、HimitsuShellは定数難読化を提供するが、shcは提供しない。

## 高度な難読化(制御フロー平坦化、ダミーコード挿入等)

![制御フローグラフ確認結果1](/assets/images/shc-vs-himitsushell/9.png)

![制御フローグラフ確認結果2](/assets/images/shc-vs-himitsushell/10.png)

Ghidraでmain関数の制御フローグラフを見ると、その差は明確だ。  
HimitsuShellは制御フローの把握が事実上不可能なほど難読化されている。  
一方、shcは制御フローが単純で解析しやすい。

## OSレベルのロギング/フッキング防御

![auditd検証結果1](/assets/images/shc-vs-himitsushell/11.png)

shcで生成したバイナリのシステムコール(例: execve)をauditdで監視すると、シェルスクリプトがそのままログに記録される。  
システムのシェルに依存する構造のため、OSレベルのロギング/フッキング攻撃に対して脆弱だ。

![auditd検証結果2](/assets/images/shc-vs-himitsushell/12.png)

一方、HimitsuShellで生成したバイナリは、同一条件下でもシェルスクリプトがログに記録されない。  
システムのシェルに依存せず、バイナリに内蔵されたシェルを使用しているためだ。

## 結論
このように、shcとHimitsuShellはセキュリティ強度の差が明確であることが分かる。

||shc|HimitsuShell|
|---|---|---|
|デバッグシンボルの削除|✓|✓|
|動的ライブラリフッキング防御| |✓|
|デバッガ検知|✓|✓|
|文字列・定数の難読化| |✓|
|高度な難読化(制御フロー平坦化、ダミーコード挿入等)| |✓|
|OSレベルのロギング/フッキング防御| |✓|

## 追加の比較
shcを改良した[ssc](https://github.com/liberize/ssc)も存在する。  
一部の弱点(動的ライブラリフッキング防御、文字列難読化)は補強されているが、高度な難読化やOSレベルのロギング/フッキング防御には依然として限界がある。

**特に、sscはシステムのシェルには依存しないが、実行時に/tmp/ssc/XXXXXXというパスにインタプリタ(例: /bin/sh等)を展開し、そこにシェルスクリプトを渡す。そのため、このパスをロギング/フッキングすればシェルスクリプトの奪取が可能だ。**

一方、HimitsuShellはインタプリタをバイナリの外部に展開しないため安全だ。