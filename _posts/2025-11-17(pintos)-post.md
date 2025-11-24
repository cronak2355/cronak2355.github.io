---
layout: post
title: "2025-11-17(pintos)"
date: 2025-11-18 11:05:33 +0900
categories: [KRAFTON JUNGLE]
tags: [Programming]
---
---

# Today's Log(2025-11-17)
>오늘은 Week10의 월요일이다 이번 주차부터 pintos 프로젝트에 들어간다. 

---

# 오늘 한 일
- priority-donate-multiple : max_priority 구현


# TIL
## max_priority() 함수의 원리

Multiple Donation에서 가장 중요한 함수인 `max_priority()`를 통해 핵심 원리를 이해해보자.

---

### 1. max_priority()가 필요한 이유

#### 문제 상황
```
Thread L (우선순위 31): Lock A와 Lock B를 보유
Thread H (우선순위 32): Lock A를 기다림
Thread M (우선순위 33): Lock B를 기다림
```

Lock B를 해제하면 L의 우선순위는
- 31로 복구는 H가 여전히 기다리므로 할 수 없음
- 33 유지는 M은 떠났으므로 할 수 없음 
- 32로 변경은 남은 대기자 중 가장 높은 우선순위

Lock 상태가 변할 때마다 **동적으로 재계산**해야 한다.

---

### 2. max_priority()의 원리

#### 핵심 아이디어
> "보유한 모든 Lock의 대기자를 확인하고, 가장 높은 우선순위를 찾는다."

#### 필요한 정보
```c
struct thread {
    int True_priority;      // 원래 우선순위 (복구 기준점)
    struct list lock_list;  // 보유 중인 Lock 목록
};

struct lock {
	struct thread *holder; 
    struct semaphore semaphore;    // semaphore.waiters에 대기자 저장
    struct list_elem lock_elem;    // lock_list 연결용
};
```

---

### 3. 동작 과정
```c
void max_priority(struct thread *t) {
    int temp = 0;
    
    if(!list_empty(&t->lock_list)) {
        // Step 1: 보유한 모든 Lock 순회
        for (각 Lock in t->lock_list) {
            
            // Step 2: 각 Lock의 모든 대기자 순회
            for (각 waiter in lock->semaphore.waiters) {
                
                // Step 3: 가장 높은 우선순위 추적
                if (waiter->priority > temp) {
                    temp = waiter->priority;
                }
            }
        }
        
        // Step 4: 원래 우선순위와 비교
        if (temp > t->True_priority) {
            t->priority = temp;           // Donation 받음
        } else {
            t->priority = t->True_priority;  // 복구
        }
    } else {
        // Step 5: 보유 Lock이 없으면 복구
        t->priority = t->True_priority;
    }
}
```

---

### 4. 실행 예시

#### 시나리오
```
Thread L (True_priority = 31)
Lock A 보유: Thread H(32) 대기
Lock B 보유: Thread M(33) 대기
```

#### max_priority(L) 실행
```
1. lock_list 확인 → Lock A, Lock B 존재

2. Lock A의 대기자 확인
   - waiter: H(32)
   - temp = 32

3. Lock B의 대기자 확인
   - waiter: M(33)
   - temp = 33 (갱신)

4. 최종 비교
   - temp(33) > True_priority(31)
   - L의 priority = 33
```

#### Lock B 해제 후 재실행
```
1. lock_list 확인 → Lock A만 존재

2. Lock A의 대기자 확인
   - waiter: H(32)
   - temp = 32

3. 최종 비교
   - temp(32) > True_priority(31)
   - L의 priority = 32
```

---

### 5. 핵심 원리

#### 1) 저장이 아닌 계산
```c
// 저장 방식
int donation_from_lockA = 32;
int donation_from_lockB = 33;
// Lock이 몇 개가 될지 모르므로 좋지 않음

// 계산 방식
// 필요할 때마다 모든 Lock 확인
```

#### 2) 최댓값 선택
여러 donation 중 선택 → **가장 높은 것**

이유: 우선순위가 가장 높은 스레드를 빨리 처리해야 함

#### 3) 조건부 적용
```c
if (temp > True_priority) {
    priority = temp;           // Donation
} else {
    priority = True_priority;  // 복구
}
```

대기자가 없거나, 모두 나보다 우선순위가 낮으면 원래대로 복구

```c
max_priority() {
    보유한 모든 Lock의 모든 대기자를 확인하고,
    가장 높은 우선순위를 선택한다.
}
```


# 코드 / 실습
```c
void max_priority(struct thread *t) {
    int temp = 0;

    if(!list_empty(&t->lock_list)) {
        struct list_elem *lock_e;
        // lock_list 순회
        for (lock_e = list_begin(&t->lock_list); lock_e != list_end(&t->lock_list); lock_e = list_next(lock_e)) {
            struct lock *lock = list_entry(lock_e, struct lock, lock_elem);
            
            // 각 lock의 waiters 순회
            struct list_elem *e;
            for (e = list_begin(&lock->semaphore.waiters); e != list_end(&lock->semaphore.waiters); e = list_next(e)) {
                struct thread *waiter = list_entry(e, struct thread, elem);
                if(waiter->priority > temp) {
                    temp = waiter->priority;
                }
            }
        }
        if(temp > t->True_priority) {
            t->priority = temp;
        }
		else {
    		t->priority = t->True_priority;  // 복구!
		}
    }
	else {
    	t->priority = t->True_priority;
	}
}
```