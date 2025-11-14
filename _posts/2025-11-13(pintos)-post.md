---
layout: post
title: "2025-11-13(pintos)"
date: 2025-11-14 11:02:50 +0900
categories: [KRAFTON JUNGLE]
tags: [Programming]
---
---

# Today's Log(2025-11-13)
>오늘은 Week10의 금요일이다 이번 주차부터 pintos 프로젝트에 들어간다.

---

# 오늘 한 일
- priority-donate 공부


# TIL
## Priority Donation(우선순위 기부)
### 핵심 개념
Priority Donation은 우선순위 역전(Priority Inversion) 현상을 해결하기 위한 스케줄링 기술이다.

우선순위 역전이란, 높은 우선순위의 스레드가 낮은 우선순위의 스레드가 점유하고 있는 **공유 자원(락)**을 기다리느라 실행되지 못하고, 이 낮은 우선순위 스레드보다 더 낮은 우선순위의 스레드에게 선점당하는(Preempted) 비효율적인 상황을 말한다.

Priority Donation은 공유 자원을 점유한 낮은 우선순위 스레드에게 그 자원을 기다리는 가장 높은 우선순위 스레드의 우선순위를 일시적으로 기부하여, 낮은 우선순위 스레드가 자원 점유 시간을 최소화하고 빠르게 자원을 해제하도록 강제하는 기법이다.

#### 전체 흐름 이해하기
```
[가정 : 높은 우선순위 스레드 H가 낮은 우선순위 스레드 L이 가진 Lock을 기다리는 상황]

[스레드 H (우선순위 높음)]
          ↓
    Lock 획득 시도
          ↓
   Lock이 이미 사용중
   (스레드 L이 보유)
          ↓
  Priority Donation 발생
  (H의 우선순위를 L에게 기부)
          ↓
   스레드 L의 우선순위 상승
   (임시로 H의 우선순위로)
          ↓
  L이 우선적으로 실행됨
          ↓
   L이 작업 완료
          ↓
    L이 Lock 반환
          ↓
  L의 우선순위 원래대로 복구
          ↓
   H가 Lock 획득
          ↓
   H 실행 재개
          ↓
  [Priority Inversion 문제 해결!]
 ```

# 코드 / 실습

```c
```