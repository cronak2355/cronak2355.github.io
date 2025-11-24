---
layout: post
title: "2025-11-20(pintos)"
date: 2025-11-20 00:50:33 +0900
categories: [KRAFTON JUNGLE]
tags: [Programming]
---
---
# 스레드(Thread)

**프로그램 실행의 최소 단위**

여러 스레드가 하나의 CPU를 번갈아 사용하며 
마치 동시에 실행되는 것처럼 보인다.

---

## 스레드의 상태

![](/assets/img/images/About-pintos-base.png)


**4가지 상태:**
- **Ready** - CPU를 기다리는 중
- **Running** - CPU에서 실행 중
- **Blocked** - 이벤트를 기다리는 중  
- **Dying** - 종료됨

---

## 스레드 전환 흐름

1. **생성(New)** → Ready
2. **스케줄러 선택** → Running
3. **실행 중 3가지 경로:**
   - 인터럽트 → Ready (CPU 양보)
   - I/O 대기 → Blocked → Ready
   - 완료 → Terminated

---

# 인터럽트(Interrupt)

**하드웨어나 소프트웨어가 CPU에 보내는 신호**

CPU가 현재 작업을 중단하고 특정 작업을 처리하게 만든다.

---

## 인터럽트 동작 과정
```
인터럽트 신호 발생
    ↓
현재 실행 중단 & 상태 저장
    ↓
운영체제 개입 (Interrupt Handler)
    ↓
인터럽트 처리
    ↓
원래 작업 재개 또는 다른 스레드로 전환
```

**역할:** 운영체제가 시스템을 제어할 수 있는 **시점**을 제공

---

## 인터럽트의 종류

- **하드웨어 인터럽트**: 키보드 입력, 타이머, 디스크 I/O 완료 등
- **소프트웨어 인터럽트**: 시스템 콜, 예외 처리 등

→ Pintos에서는 주로 **타이머 인터럽트**를 활용


