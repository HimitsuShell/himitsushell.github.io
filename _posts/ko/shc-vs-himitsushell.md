---
layout: post
title: "쉘 스크립트 보호 도구 비교: shc vs HimitsuShell (바이너리화, 암호화, 난독화)"
date: 2026-08-10 11:00:00 +0900
lang: ko
---
[shc](https://github.com/neurobin/shc/tree/master)(쉘 스크립트 컴파일러)는 쉘 스크립트를 바이너리로 변환하여 코드 유출을 막는 도구다. 하지만 실제 사용에는 다음과 같은 한계가 존재한다.
- 난독화 기능을 제공하지 않아, 리버스 엔지니어링에 취약하다.
- 시스템 쉘(예: /bin/sh)에 의존하고 있어, 로깅/후킹 공격에 취약하다.

[HimitsuShell](https://github.com/HimitsuShell/HimitsuShell)은 이를 보완하기 위해 만들어졌다. llvm 기반의 다양한 난독화 기법이 적용되었고, 시스템 쉘에도 의존하지 않는다.

이제 항목별로 비교해보자.

## 테스트 환경
ubuntu 24.04에서 아래 쉘 스크립트를 사용한다.

![테스트용 쉘 스크립트](/assets/images/shc-vs-himitsushell/1.png)

HimitsuShell은 기본 옵션으로, shc는 아래와 같이 [최대 보안 강도](https://github.com/neurobin/shc/blob/master/man.md)로 진행한다.

```shell
shc -Uf launcher.sh -o shc_binary
```

아래와 같이 HimitsuShell과 shc로 생성한 바이너리 모두 정상 동작한다.

![바이너리 작동 테스트 결과](/assets/images/shc-vs-himitsushell/2.png)

## 디버그 심볼 제거

![제거된 디버그 심볼](/assets/images/shc-vs-himitsushell/3.png)

HimitsuShell과 shc 모두 디버그 심볼이 제거되었다.

## 동적 라이브러리 후킹 방어

![동적 라이브러리 조회 결과](/assets/images/shc-vs-himitsushell/4.png)

HimitsuShell은 동적 라이브러리에 의존하지 않지만, shc는 의존한다.  
따라서 shc는 후킹 공격에 취약하다.

## 디버거 감지

![디버거 감지 검증 결과1](/assets/images/shc-vs-himitsushell/5.png)

![디버거 감지 검증 결과2](/assets/images/shc-vs-himitsushell/6.png)

HimitsuShell과 shc 모두 디버거(예: gdb, strace 등)를 감지해 차단한다.

## 문자열, 상수 난독화

![문자열 난독화 검증 결과1](/assets/images/shc-vs-himitsushell/7.png)

![문자열 난독화 검증 결과1](/assets/images/shc-vs-himitsushell/8.png)

Ghidra에서 문자열 목록을 추출해보면 차이가 명확하다.  
HimitsuShell은 문자열이 난독화되어 있지만, shc로 만든 바이너리는 민감한 문자열이 그대로 노출된다.  
또한 HimitsuShell은 상수 난독화를 제공하지만, shc는 제공하지 않는다.

## 고급 난독화 (제어 흐름 평탄화, 가짜 코드 삽입 등)

![제어 흐름 그래프 조회 결과1](/assets/images/shc-vs-himitsushell/9.png)

![제어 흐름 그래프 조회 결과2](/assets/images/shc-vs-himitsushell/10.png)

Ghidra에서 main 함수의 제어 흐름 그래프를 보면 차이가 명확하다.  
HimitsuShell은 제어 흐름을 파악하는게 사실상 불가능할 정도로 난독화되었다.  
반면 shc는 제어 흐름이 단순하여 분석하기 쉽다.

## 운영체제단 로깅/후킹

![auditd 검증 결과1](/assets/images/shc-vs-himitsushell/11.png)

shc로 생성한 바이너리의 시스템 콜(예: execve)를 auditd로 모니터링하면 쉘 스크립트가 그대로 로깅된다.  
시스템 쉘에 의존하는 구조여서, 운영체제 수준의 로깅/후킹 공격에 취약하다.

![auditd 검증 결과2](/assets/images/shc-vs-himitsushell/12.png)

반면 HimitsuShell로 생성한 바이너리는 동일 조건에서도 쉘 스크립트가 로깅되지 않는다.  
시스템 쉘에 의존하지 않고, 바이너리에 내장된 쉘을 사용하기 때문이다.

## 결론
이처럼 shc와 HimitsuShell은 보안 강도의 차이가 명확하다는걸 알 수 있다.

||shc|HimitsuShell|
|---|---|---|
|디버그 심볼 제거|O|O|
|동적 라이브러리 후킹 방어|X|O|
|디버거 감지|O|O|
|문자열, 상수 난독화|X|O|
|고급 난독화 (제어 흐름 평탄화, 가짜 코드 삽입 등)|X|O|
|운영체제단 로깅/후킹 방어|X|O|

## 추가적인 비교
shc를 개선한 [ssc](https://github.com/liberize/ssc)도 존재한다.  
일부 문제(동적 라이브러리 후킹 방어, 문자열 난독화)는 보강되었지만, 고급 난독화와 운영체제 수준의 로깅/후킹 방어에는 여전히 한계가 있다.

**특히 ssc는 시스템 쉘에는 의존하지 않지만, 실행시 /tmp/ssc/XXXXXX 경로로 인터프리터(예: /bin/sh 등)을 추출해 쉘 스크립트를 전달한다. 이로 인해 해당 경로를 로깅/후킹하면 쉘 스크립트 탈취가 가능하다.**

반면 HimitsuShell은 인터프리터를 바이너리 외부로 추출하지 않아 안전하다.