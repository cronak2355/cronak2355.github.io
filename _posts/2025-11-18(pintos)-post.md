---
layout: post
title: "2025-11-18(pintos)"
date: 2025-11-18 11:10:33 +0900
categories: [KRAFTON JUNGLE]
tags: [Programming]
---
---

# Today's Log(2025-11-18)
>오늘은 Week10의 화요일이다. Multiple Donation 구현을 완료했다.

---

# 오늘 한 일
- priority-donate-multiple : lock_release 구현 


# TIL
## lock_release() 함수의 원리

Multiple Donation에서 Lock을 해제할 때, 단순히 우선순위를 원래대로 되돌리면 안 된다. 
**남아있는 다른 Lock의 대기자**를 고려해야 한다.

---

### 1. lock_release()가 해결해야 할 문제

#### 문제 상황
```
Thread L (True_priority = 31, priority = 33)
- Lock A 보유: Thread H(32) 대기
- Lock B 보유: Thread M(33) 대기
```

Lock B를 해제한다면
- 단순히 우선순위를 31로 되돌리거나 33으로 유지할 수 없다.
- **`priority = 32`로 재계산** 해야 한다

즉 Lock을 해제할 때마다 **우선순위를 재계산**해야 한다.

---

### 2. lock_release()의 원리

#### 핵심 아이디어
> "Lock을 해제하고, 남아있는 Lock들을 기준으로 우선순위를 다시 계산한다."



#### 구현 방식
```c

void lock_release(struct lock *lock) {
    list_remove(&lock->lock_elem);    // 1. lock_list에서 제거
    lock->holder = NULL;               // 2. 소유자 해제
    max_priority(thread_current());    // 3. 우선순위 재계산
    sema_up(&lock->semaphore);         // 4. 대기자 깨우기
}
```

---

### 3. 동작 과정
```c
void lock_release(struct lock *lock) {
    ASSERT(lock != NULL);
    ASSERT(lock_held_by_current_thread(lock));
    
    // Step 1: lock_list에서 제거
    list_remove(&lock->lock_elem);
    
    // Step 2: 소유자 정보 제거
    lock->holder = NULL;
    
    // Step 3: 우선순위 재계산 
    max_priority(thread_current());
    
    // Step 4: 대기 중인 스레드 깨우기
    sema_up(&lock->semaphore);
}
```

---

### 4. 실행 예시

#### 초기 상태
```
Thread L (True_priority = 31, priority = 33)
lock_list: [Lock A, Lock B]
- Lock A: Thread H(32) 대기
- Lock B: Thread M(33) 대기
```

#### lock_release(Lock B) 실행

**Step 1: list_remove(&lock->lock_elem)**
```
lock_list에서 Lock B 제거
lock_list: [Lock A]  // Lock A만 남음
```

**Step 2: lock->holder = NULL**
```
Lock B의 소유자 정보 제거
Lock B.holder = NULL
```

**Step 3: max_priority(thread_current())**
```
L의 우선순위 재계산:
1. lock_list 확인 → Lock A 존재
2. Lock A의 대기자 확인 → H(32)
3. max(31, 32) = 32
4. L의 priority = 32
```

**Step 4: sema_up(&lock->semaphore)**
```
Lock B를 기다리던 Thread M 깨우기
M이 Lock B 획득하고 실행
```

---

### 5. 핵심 원리

#### 1) 순서가 중요하다
```c
// 올바른 순서
list_remove(&lock->lock_elem);    // 먼저 제거
max_priority(thread_current());    // 그 다음 재계산
sema_up(&lock->semaphore);         // 마지막에 깨우기

// 잘못된 순서
sema_up(&lock->semaphore);         // 먼저 깨우면?
max_priority(thread_current());    // 우선순위가 이상해질 수 있음
```

#### 2) list_remove()의 역할
```c
list_remove(&lock->lock_elem);
```
- `lock_list`에서 해당 Lock 제거
- 이제 `max_priority()`는 **남은 Lock들만** 확인
- 해제된 Lock의 대기자는 더 이상 고려되지 않음

#### 3) max_priority()의 역할
```c
max_priority(thread_current());
```
- 현재 보유 중인 **모든 Lock** 확인
- 각 Lock의 **모든 대기자** 확인
- **가장 높은 우선순위** 선택
- 원래 우선순위와 비교하여 최종 결정

---

```
Lock 해제 요청
    ↓
lock_list에서 제거 (list_remove)
    ↓
소유자 정보 제거 (holder = NULL)
    ↓
우선순위 재계산 (max_priority)
    ↓
대기자 깨우기 (sema_up)
    ↓
완료
```

**핵심:** Lock 상태 변경 → 우선순위 재계산 → 스케줄링

---

# 코드 / 실습
```c
void lock_release(struct lock *lock) {
    ASSERT(lock != NULL);
    ASSERT(lock_held_by_current_thread(lock));
    
    // 1. lock_list에서 해당 Lock 제거
    list_remove(&lock->lock_elem);
    
    // 2. Lock의 소유자 정보 제거
    lock->holder = NULL;
    
    // 3. 남은 Lock들을 기준으로 우선순위 재계산
    max_priority(thread_current());
    
    // 4. 대기 중인 스레드 깨우기
    sema_up(&lock->semaphore);
}
```