---
layout: post
title: "2025-11-22(pintos)"
date: 2025-11-23 00:39:47 +0900
categories: [KRAFTON JUNGLE]
tags: [Programming]
---
---

# Today's Log(2025-11-21)
>오늘은 Week11의 토요일이다. 이번 주차부터 pintos 프로젝트에 들어간다.

---

# 오늘 한 일
- argument passing 개념 공부


# TIL
## File Descriptor


| 항목              | 값 / 규칙                                    |
|-------------------|---------------------------------------------|
| 자료형            | `int` (0번부터 시작)                         |
| 0번               | STDIN_FILENO → 무조건 콘솔 입력               |
| 1번               | STDOUT_FILENO → 무조건 콘솔 출력              |
| 2번부터           | open() 한 파일 순서대로 할당                  |
| 최대 개수         | 128개 (thread->fd_table[128])                |
| 저장 위치         | `thread_current()->fd_table[idx] = struct file *` |
| exec() 시         | 자식 프로세스는 부모 fd **물려받지 않음** |

## Cache

### 1. 왜 Cache가 필요한가?

| 저장소         | 접근 시간          | 비고                              |
|----------------|-------------------|-----------------------------------|
| Register       | 1 사이클 이하      | CPU 내부에 존재                   |
| L1 Cache       | 4~5 사이클        | 코어별 전용                        |
| L2 Cache       | 10~20 사이클      |                                   |
| L3 Cache       | 40~70 사이클      | 코어들 공유                        |
| DRAM           | 200~300 사이클    |  우리가 사용하는 RAM              |
| SSD            | 수십만 사이클     |                                   |

→ CPU에게 DRAM은 너무 느리기에 Cache가 필요하다.

### 2. Cache 기본 구조

- **Cache Line**: 64바이트 고정 (한 번에 이 크기씩 메모리에서 끌고 온다)
- **Set-Associative**: 대부분 8-way (한 Set에 8개 Line)
- **Tag + Set Index + Offset** → 이 3개로 Cache Hit/Miss 판단

### 3. Cache Miss 3종 세트 (면접 100% 나옴)

| 종류            | 원인                            | 해결법                     |
|-----------------|----------------------------------|----------------------------|
| Cold Miss       | 처음 접근                        | 대처 하기 쉽지 않음        |
| Capacity Miss   | Cache 크기 부족                  | Cache 키우거나 코드 최적화 |
| Conflict Miss   | 같은 Set에 너무 많이 몰림        | Fully Associative면 없어진다 |

### 4. Write Policy 2가지

| 정책            | 특징                                      | Pintos에서 쓰는  |
|-----------------|-------------------------------------------|-------------------|
| Write-through   | Cache 쓰자마자 바로 DRAM에도 씀           | 느리기에 쓰지 않는다           |
| **Write-back**  | Cache에만 쓰고 Dirty bit 켜고 나중에 한 번에 반영 | **사용한다**      |

→ Pintos Buffer Cache는 **Write-back + Clock Algorithm** 조합

### 5. Pintos Project 4 Buffer Cache 핵심 스펙

| 항목                | 값 / 특징                                         |
|---------------------|---------------------------------------------------|
| 총 Entry 수         | 512개 (`buffer_head cache[512]`)                  |
| 한 블록 크기        | 512바이트 (sector 크기)                           |
| 매핑 방식           | `sector_idx % 512` → 같은 sector 중복 불가         |
| Eviction 알고리즘   | **Clock Algorithm** (accessed_bit 활용)           |
| Dirty 블록 처리     | 주기적으로 write-behind (주기적으로 flush)        |
| 보너스 점수         | Sequential ahead-read (다음 sector 미리 읽기)    |

### 6. 요약

Cache는 64바이트 단위로 CPU한테 매우 빠르게 정보를 준다,  
Pintos Buffer Cache는 512개 슬롯 + Write-back + Clock Algorithm


# 코드 / 실습
```c
```
