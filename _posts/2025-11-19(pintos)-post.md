---
layout: post
title: "2025-11-19(pintos)"
date: 2025-11-20 00:10:33 +0900
categories: [KRAFTON JUNGLE]
tags: [Programming]
---
---

# Today's Log(2025-11-19)
>오늘은 Week10의 수요일이다. Multiple Donation의 lock_acquire 구현을 완료했다.

---

# 오늘 한 일
- priority-donate-multiple : lock_acquire 구현 


# TIL
## lock_acquire() 함수의 원리

Multiple Donation에서 Lock을 획득할 때, Lock이 이미 보유 중이라면 **현재 스레드의 우선순위를 소유자에게 빌려줘야** 한다.
`lock_acquire()`는 이 donation이 실제로 발생하는 지점이다.

---

### 1. lock_acquire()가 해결해야 할 문제

#### 문제 상황: 우선순위 역전
```
Thread L (우선순위 31): Lock A를 보유 중
Thread H (우선순위 32): Lock A를 획득하려고 대기

현상: H가 L을 기다려야 하므로, 우선순위가 의미 없어짐
```

H가 자신의 우선순위(32)를 L에게 donation
→ L이 더 높은 우선순위로 빨리 실행
→ Lock을 빨리 해제
→ H가 빨리 Lock을 획득

---

### 2. lock_acquire()의 원리

#### 핵심 아이디어
Lock이 이미 보유 중이면, 소유자에게 내 우선순위를 직접 donation한다.

#### 필요한 정보
```c
struct thread {
    int priority;           // 현재 우선순위 (donation 반영)
    int True_priority;      // 원래 우선순위 (복구 기준점)
    struct list lock_list;  // 보유 중인 Lock 목록
};

struct lock {
    struct thread *holder;         // Lock 소유자
    struct semaphore semaphore;    // 대기자 관리
    struct list_elem lock_elem;    // lock_list 연결용
};
```

---

### 3. 동작 과정
```c
void lock_acquire(struct lock *lock) {
    ASSERT(lock != NULL);
    ASSERT(!intr_context());
    ASSERT(!lock_held_by_current_thread(lock));
    
    struct thread *t = thread_current();
    
    // Step 1: Lock이 이미 보유 중인지 확인
    if (lock->holder != NULL) {
        
        // Step 2: 소유자에게 우선순위 donation
        if (t->priority > lock->holder->priority) {
            lock->holder->priority = t->priority;
        }
    }
    
    // Step 3: Lock 획득 대기 (blocking)
    sema_down(&lock->semaphore);
    
    // Step 4: Lock 획득 성공 - 소유자 설정
    lock->holder = t;
    
    // Step 5: lock_list에 추가
    list_push_back(&t->lock_list, &lock->lock_elem);
}
```

---

### 4. 실행 예시

#### 초기 상태
```
Thread L (True_priority = 31, priority = 31): Lock A 보유
Thread H (True_priority = 32, priority = 32): Lock A 획득 시도
```

#### lock_acquire(Lock A) 실행 (Thread H)

**Step 1: if (lock->holder != NULL)**
```
Lock A의 holder 확인
→ L이 보유 중 (holder = L)
→ donation 수행 필요
```

**Step 2: 우선순위 비교 및 donation**
```
H의 priority(32) > L의 priority(31)?
→ Yes
→ L의 priority = 32로 갱신
```

**Step 3: sema_down(&lock->semaphore)**
```
Lock 획득 대기
→ Thread H는 BLOCKED 상태
→ semaphore.waiters에 H 추가
→ L이 Lock을 해제할 때까지 대기
```

**Step 4: lock->holder = t**
```
(나중에 L이 Lock을 해제하면)
→ H가 깨어남
→ Lock A.holder = H
```

**Step 5: list_push_back(&t->lock_list, &lock->lock_elem)**
```
H의 lock_list에 Lock A 추가
→ H->lock_list: [Lock A]
→ 나중에 max_priority() 계산 시 사용
```

---

### 5. 핵심 원리

#### 1) 직접 donation
```c
if (t->priority > lock->holder->priority) {
    lock->holder->priority = t->priority;
}
```
- **단순하고 직접적인 방식**
- 내 우선순위가 더 높으면 → 즉시 donation
- Multiple Donation은 `max_priority()`가 처리

#### 2) lock_list에 추가하는 이유
```c
list_push_back(&t->lock_list, &lock->lock_elem);
```
- Lock을 획득한 후 자신의 목록에 기록
- `lock_release()` 시 이 목록에서 제거
- `max_priority()` 계산 시 이 목록 순회

#### 3) 순서의 중요성
```c
// 올바른 순서
if (lock->holder) { donation }  // 1. Donation
sema_down(&lock->semaphore);     // 2. 대기
lock->holder = t;                // 3. 소유자 설정
list_push_back(...);             // 4. 목록 추가
```

---

### 6. lock_list의 역할

#### 왜 필요한가?
```c
struct thread {
    struct list lock_list;  // 보유 중인 Lock 목록
};
```

**목적:**
1. **현재 보유 Lock 추적**: 어떤 Lock을 가지고 있는지
2. **lock_release() 시 제거**: 해제된 Lock을 목록에서 제거
3. **max_priority() 계산**: 모든 보유 Lock의 대기자 확인

#### 언제 사용되는가?
```c
// lock_acquire() - 추가
list_push_back(&t->lock_list, &lock->lock_elem);

// lock_release() - 제거
list_remove(&lock->lock_elem);

// max_priority() - 순회
for (각 lock in t->lock_list) {
    // 각 lock의 waiters 확인
}
```

---

### 7. 전체 흐름 요약
```
Lock 획득 시도
    ↓
Lock 보유 중?
    ↓ (Yes)
우선순위 비교
    ↓ (더 높으면)
소유자에게 donation
    ↓
Lock 획득 대기 (sema_down)
    ↓
획득 성공
    ↓
소유자 설정 (holder)
    ↓
lock_list에 추가
    ↓
완료
```

**핵심:** 비교 → Donation → 대기 → 획득 → 기록

---

# 코드 / 실습
```c
void lock_acquire(struct lock *lock) {
    ASSERT(lock != NULL);
    ASSERT(!intr_context());
    ASSERT(!lock_held_by_current_thread(lock));
    
    struct thread *t = thread_current();
    
    // 1. Lock이 보유 중이면 donation 수행
    if (lock->holder != NULL) {
        // 2. 내 우선순위가 더 높으면 donation
        if (t->priority > lock->holder->priority) {
            lock->holder->priority = t->priority;
        }
    }
    
    // 3. Lock 획득 대기 (blocking)
    sema_down(&lock->semaphore);
    
    // 4. Lock 획득 완료 - 소유자 설정
    lock->holder = t;
    
    // 5. 내 lock_list에 추가 (나중에 max_priority 계산용)
    list_push_back(&t->lock_list, &lock->lock_elem);
}
```


