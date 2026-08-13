---
layout: post
title: "쉘 스크립트 보안: shc의 구조적 한계와 취약점 (암호화, 컴파일러, 난독화)"
---

[shc](https://github.com/neurobin/shc)는 가장 널리 알려진 쉘 스크립트 보호 도구다.  
쉘 스크립트를 C 소스코드로 감싸서 바이너리로 변환하는 원리로, 소스코드 노출을 막는다.

그러나 shc는 구조적으로 시스템 쉘에 의존하기 때문에, 운영체제 수준의 로깅/후킹 공격에 취약하다.  

shc로 생성된 바이너리는 쉘 스크립트를 시스템 쉘(예: /bin/sh)에 전달하여 실행한다.  
**따라서 시스템 쉘로 인자를 넘기는 순간만 모니터링하면 쉽게 쉘 스크립트를 탈취 할 수 있다.**

![shc 취약점 구조도](/assets/images/shc-security-analysis/1.png)

이제 직접 테스트해서 취약점을 확인해보자.

## 테스트 환경
ubuntu 24.04에서 아래 쉘 스크립트를 사용한다.

![테스트용 쉘 스크립트](/assets/images/shc-security-analysis/2.png)

아래와 같이 [최대 보안 강도](https://github.com/neurobin/shc/blob/master/man.md)로 진행한다.

```shell
shc -Uf launcher.sh -o shc_binary
```

## 테스트 방법
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

아래 사진의 빨간 박스와 같이, **auditd(운영체제단 감시 도구)를 사용하면 손쉽게 쉘 스크립트를 탈취**할 수 있다.

![터미널에 노출된 쉘 스크립트](/assets/images/shc-security-analysis/3.png)


## 해결 방법
시스템 쉘에 의존하지 않는 보호 도구를 사용해야한다.  
시스템 쉘에 의존하는 구조에서는 위와 같은 운영체제 수준의 로깅/후킹 공격을 이론적으로 막아낼 수 없다.

시중에는 [ssc](https://github.com/liberize/ssc), [HimitsuShell](https://github.com/HimitsuShell/HimitsuShell) 등의 보호 도구가 존재한다.  
**하지만 ssc도 치명적인 구조적 약점을 가지고 있다.**

다음 글에서 이 내용을 소개한다.