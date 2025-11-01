---
layout: post
title: "Implicit Free LIST 매크로에 관하여"
date: 2025-10-29 23:53:15 +0900
categories: [KRAFTON JUNGLE]
tags: [Programming]
---
# [Malloc Lab] Implicit Free List 매크로

## 들어가며

Malloc Lab을 구현하면서  가장 먼저 마주하게 되는 것이 바로 **매크로(Macro)**입니다. 

처음에는 어렵고 복잡하다는 생각이 들 수 있지만, 매크로를 이해하고 나면 정말 멋있고 편리한 기능이라는 것을 알 수 있습니다.

이 글에서는 Implicit Free List 구현에 필요한 모든 매크로를 하나씩 뜯어보겠습니다.

---

## 목차
1. 기본 상수 매크로
2. 메모리 조작 매크로
3. 블록 정보 읽기 매크로
4. 블록 포인터 계산 매크로
5. 실제 사용 예시

---

## 기본 상수 매크로

### WSIZE, DSIZE, CHUNKSIZE
```c
#define WSIZE 4        // 워드(4바이트)
#define DSIZE 8        // 더블 워드(8바이트)
#define CHUNKSIZE (1<<12)  // 힙 확장(4096바이트)
```

**WSIZE (Word Size)**
- 1 워드 = 4바이트
- Header와 Footer의 크기
- 32비트 시스템 기준

**DSIZE (Double Word Size)**
- 2 워드 = 8바이트
- **정렬(Alignment) 요구사항**
- 모든 블록은 8바이트의 배수여야 함

**CHUNKSIZE**
- 힙 확장 시 요청할 크기
- `1 << 12` = 2^12 = 4096바이트 = 4KB

---

### MAX 매크로
```c
#define MAX(x,y) ((x)>(y)?(x):(y))
```

**역할:** 두 값 중 큰 값 반환

---

## 메모리 조작 매크로

### PACK - 크기와 할당 비트 합치기
```c
#define PACK(size, alloc) ((size)|(alloc))
```

**동작 원리:**
- `size`: 블록 크기 (8의 배수, 하위 3비트는 항상 0)
- `alloc`: 할당 여부 (0 또는 1)
- OR 연산으로 합침

**예시:**
```c
PACK(16, 1)
= 16 | 1
= 0b10000 | 0b00001 //0b는 이진수라는 뜻임(0은 8진수 0x는 16진수)
= 0b10001
= 17

// 17에서:
// 크기 = 17 & ~0x7 = 16
// 할당 = 17 & 0x1 = 1
```

---

### GET / PUT - 메모리 읽기/쓰기
```c
#define GET(p) (*(unsigned int*)(p))
#define PUT(p, val) (*(unsigned int*)(p) = (val))
```

**GET(p)**
- 주소 `p`를 읽음
- `unsigned int`로 캐스팅(unsigned int는 음수 없는 0이상의 정수)

**PUT(p, val)**
- 주소 `p`에 값 `val`을 씀

**사용 예:**
```c
PUT(heap_listp, 0);  // heap_listp 위치에 0 쓰기
int value = GET(heap_listp);  // heap_listp에서 값 읽기
```

---

## 블록 정보 읽기 매크로

### GET_SIZE - 블록 크기 읽기
```c
#define GET_SIZE(p) (GET(p) & ~0x7)
```

**동작:**
1. `GET(p)`: 주소 p에서 값 읽기
2. `& ~0x7`: 하위 3비트를 0으로 만듦

**왜 하위 3비트를 제거?**
- 블록 크기는 8의 배수 (DSIZE 정렬)
- 하위 3비트는 할당 비트용
- 크기만 추출하려면 하위 3비트 제거

**예시:**
```c
// Header에 17(0b10001)이 저장되어 있다면
GET_SIZE(header_addr)
= 17 & ~0x7
= 0b10001 & 0b11000
= 0b10000
= 16  // 순수 크기
```

---

### GET_ALLOC - 할당 여부 읽기
```c
#define GET_ALLOC(p) (GET(p) & 0x1)
```

**동작:**
1. `GET(p)`: 주소 p에서 값 읽기
2. `& 0x1`: 최하위 1비트만 추출

**결과:**
- `0`: Free 블록
- `1`: Allocated 블록

**예시:**
```c
// Header에 17(0b10001)이 저장되어 있다면
GET_ALLOC(header_addr)
= 17 & 0x1
= 0b10001 & 0b00001
= 1  // 할당됨
```

---

## 블록 포인터 계산 매크로

### 블록 구조
```
[Header: 4B][Payload: N bytes][Footer: 4B]
            ↑
            bp (블록 포인터)
```

**bp는 Payload의 시작 주소**

---

### HDRP - Header 주소 계산
```c
#define HDRP(bp) ((char*)(bp) - WSIZE)
```

**동작:**
- bp에서 4바이트(WSIZE) 앞으로 이동
- Header는 Payload 바로 앞에 위치

**시각화:**
```
[Header]    [Payload]
   ↑           ↑
 HDRP(bp)     bp

bp - 4 = Header 주소
```

---

### FTRP - Footer 주소 계산
```c
#define FTRP(bp) ((char*)(bp) + GET_SIZE(HDRP(bp)) - DSIZE)
```

**동작 단계:**
1. `HDRP(bp)`: Header 주소 구하기
2. `GET_SIZE(HDRP(bp))`: 블록 전체 크기 읽기
3. `bp + 전체크기`: 다음 블록의 Header 위치
4. `- DSIZE`: 8바이트(Header 4B + Footer 4B) 뒤로 = Footer 위치

**시각화:**
```
[H:4B][Payload: N][F:4B]
  ↑               ↑     ↑
Header           bp   다음 Header

Footer = (다음 Header) - 8
       = bp + 전체크기 - DSIZE
```

**예시:**
```c
// 블록 크기가 16바이트라면
// [Header:4][Payload:8][Footer:4]

FTRP(bp) = bp + 16 - 8
         = bp + 8
         = Footer 위치
```

---

### NEXT_BLKP - 다음 블록 찾기
```c
#define NEXT_BLKP(bp) ((char*)(bp) + GET_SIZE(((char*)(bp) - WSIZE)))
```

**동작 단계:**
1. `bp - WSIZE`: 현재 블록의 Header
2. `GET_SIZE(...)`: 현재 블록 크기
3. `bp + 크기`: 다음 블록의 bp

**시각화:**
```
[현재 블록............][다음 블록............]
        ↑                      ↑
       bp                NEXT_BLKP(bp)

다음 bp = 현재 bp + 현재 블록 크기
```

---

### PREV_BLKP - 이전 블록 찾기
```c
#define PREV_BLKP(bp) ((char*)(bp) - GET_SIZE(((char*)(bp) - DSIZE)))
```

**동작 단계:**
1. `bp - DSIZE`: 이전 블록의 Footer
2. `GET_SIZE(...)`: 이전 블록 크기
3. `bp - 크기`: 이전 블록의 bp

**시각화:**
```
[이전 블록............][현재 블록............]
        ↑                      ↑
  PREV_BLKP(bp)              bp

이전 bp = 현재 bp - 이전 블록 크기
```

---

## 매크로 사용의 핵심 포인트

### 1. 타입 캐스팅의 중요성
```c
#define HDRP(bp) ((char*)(bp) - WSIZE)
                  ↑
              char*로 캐스팅!
```

**왜 char*?**
- 1바이트 단위로 포인터 연산
- `bp - WSIZE` = 정확히 4바이트 앞으로

---

### 2. 괄호의 중요성
```c
// 나쁜 예
#define MAX(x,y) x>y?x:y

MAX(a+1, b+2)
= a+1>b+2?a+1:b+2  // 연산 순서 문제!

// 좋은 예
#define MAX(x,y) ((x)>(y)?(x):(y))

MAX(a+1, b+2)
= ((a+1)>(b+2)?(a+1):(b+2))  // 안전!
```

---

### 3. 비트 연산의 이해
```c
// 크기 추출
GET_SIZE(p) = value & ~0x7
            = value & 0b11111000

// 할당 비트 추출
GET_ALLOC(p) = value & 0x1
             = value & 0b00000001
```

**핵심:** 
- 크기는 8의 배수 (하위 3비트 항상 0)
- 마지막 1비트를 할당 여부로 사용

---

## 자주 하는 실수

### 실수 1: GET_SIZE에 bp 직접 사용
```c
// 틀림
GET_SIZE(bp)

// 맞음
GET_SIZE(HDRP(bp))
```

**이유:** GET_SIZE는 Header/Footer 주소를 받아야 함!

---

### 실수 2: PACK 없이 PUT 사용
```c
// 틀림
PUT(HDRP(bp), size);

// 맞음
PUT(HDRP(bp), PACK(size, 0));
```

**이유:** 크기와 할당 비트를 합쳐야 함!

---

### 실수 3: 포인터 연산 실수
```c
// 틀림
bp + WSIZE  // void* + int는 동작이 불명확!

// 맞음
(char*)bp + WSIZE  // char*로 캐스팅 후 연산
```

---

## 매크로 전체 정리표

| 매크로 | 역할 | 입력 | 출력 |
|--------|------|------|------|
| WSIZE | 워드 크기 | - | 4 |
| DSIZE | 더블 워드 크기 | - | 8 |
| CHUNKSIZE | 힙 확장 크기 | - | 4096 |
| MAX(x,y) | 최댓값 | 두 값 | 큰 값 |
| PACK(size,alloc) | 크기+할당 합치기 | 크기, 할당비트 | 합친 값 |
| GET(p) | 메모리 읽기 | 주소 | 4바이트 값 |
| PUT(p,val) | 메모리 쓰기 | 주소, 값 | - |
| GET_SIZE(p) | 크기 읽기 | Header/Footer | 블록 크기 |
| GET_ALLOC(p) | 할당 여부 읽기 | Header/Footer | 0 or 1 |
| HDRP(bp) | Header 주소 | bp | Header 주소 |
| FTRP(bp) | Footer 주소 | bp | Footer 주소 |
| NEXT_BLKP(bp) | 다음 블록 | bp | 다음 bp |
| PREV_BLKP(bp) | 이전 블록 | bp | 이전 bp |

---

