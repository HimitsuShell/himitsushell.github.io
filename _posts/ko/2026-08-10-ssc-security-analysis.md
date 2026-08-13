---
layout: post
title: "리눅스 쉘 스크립트 보안: ssc의 구조적 한계와 취약점 (소스코드 보호, 난독화, 역공학)"
---

[ssc](https://github.com/liberize/ssc)는 shc(쉘 스크립트 컴파일러)를 개선한 프로젝트다.  
shc와 같은 원리로, 쉘 스크립트를 C 소스코드로 감싼 뒤 바이너리로 변환하여 코드 노출을 막는다.

ssc는 shc와 달리 시스템 쉘에 의존하지 않고, 별도의 쉘 인터프리터(예: BusyBox)를 사용한다.  
따라서 [shc를 공격할때 사용했던 기법(auditd)](https://himitsushell.github.io/ko/shc-security-analysis/)은 ssc에서는 동일하게 사용할 수 없다.

그러나 ssc 역시 구조적인 한계를 가지고 있다.  
ssc로 생성된 바이너리는 내장된 쉘 인터프리터(예: BusyBox)를 잠시 /tmp/ssc.XXXXXX/busybox 경로에 내보내고, 이곳에 쉘 스크립트를 전달하여 실행한다.  
**따라서 내장된 쉘 인터프리터가 외부로 노출되는 순간만 모니터링하면, 쉘 스크립트를 탈취 할 수 있다.**

![ssc 취약점 구조도](/assets/images/ssc-security-analysis/1.png)

이제 직접 테스트해서 취약점을 확인해보자

## 테스트 환경
ubuntu 24.04에서 아래 쉘 스크립트를 사용한다.

![테스트용 쉘 스크립트](/assets/images/ssc-security-analysis/2.png)

아래와 같이 [쉘 인터프리터(BusyBox)를 내장](https://github.com/liberize/ssc/tree/master/examples/4_embed_interpreter)해서 진행한다.

```shell
./ssc test ssc_binary -s -r -e busybox -c
```

## 테스트 방법
```shell
# Install bpftrace
sudo apt install bpftrace

# Start monitoring in terminal 1
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_write /comm == "ssc_binary"/ { printf("PID: %d | FD: %d | Data: %s\n", pid, args->fd, str(args->buf)); }'

# Run ssc_binary in terminal 2
./ssc_binary
```

**아래 사진의 빨간 박스와 같이, bpftrace(커널 감시 도구)를 사용하면 손쉽게 쉘 스크립트를 탈취 할 수 있다.**

![터미널2 모니터링 결과](/assets/images/ssc-security-analysis/3.png)

![터미널1 모니터링 결과](/assets/images/ssc-security-analysis/4.png)

## 해결 방법
쉘 인터프리터를 외부로 추출하지 않고, 내부적으로만 처리하는 보호 도구를 사용해야한다.  
대표적인 예로 [HimitsuShell](https://github.com/HimitsuShell/HimitsuShell)이 있다.

다음 글에서 HimitsuShell을 소개한다.