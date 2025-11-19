---
layout: post
title: "2025-11-14(pintos)"
date: 2025-11-18 11:02:50 +0900
categories: [KRAFTON JUNGLE]
tags: [Programming]
---
---

# Today's Log(2025-11-14)
>오늘은 Week10의 금요일이다 이번 주차부터 pintos 프로젝트에 들어간다.

---

# 오늘 한 일
- priority-donate-one 구현


# TIL
## priority-donate-one
### 핵심 역할
가장 기본적인 Priority Donation 즉 우선순위를 빌려주고 다시 돌려 받아야 한다.
```
- Main 스레드 (우선순위: 낮음)
- Medium 스레드 (우선순위: 중간) 
- High 스레드 (우선순위: 높음)
- Lock (공유 자원)
```

실행 과정

1단계: Main이 Lock을 잡음
```
Main (낮음) Lock 획득
```
2단계: Medium 생성 및 대기
```
Medium (중간) 생성
     ↓
Medium이 Lock 필요
     ↓
Lock은 Main이 보유 중
     ↓
Priority Donation 발생!
Main (낮음 → 중간)
```
3단계: High 생성 및 대기
```
High (높음) 생성
     ↓
High도 Lock 필요
     ↓
Lock은 여전히 Main이 보유 중
     ↓
또 다시 Priority Donation
Main (중간 → 높음)
핵심: Main이 High의 우선순위를 받아서 빠르게 실행되어야함
```
4단계: Lock 해제
```
Main이 Lock 해제
     ↓
Main 우선순위 복구 (높음 → 낮음)
     ↓
대기 중인 스레드 중 가장 높은 High가 Lock 획득
```
5단계: 실행 순서
```
1. High (높음) 실행 완료
2. Medium (중간) 실행 완료  
3. Main (낮음) 나머지 작업 수행
``` 

왜 필요한가?
```
Priority Donation이 없을 경우
Main (낮음) Lock 보유
High (높음) Lock 대기
Medium (중간) Main 선점하고 실행
     ↓
High는 계속 기다된다 (Priority Inversion)
```
```
Priority Donation이 있을 경우
해결:
Main (낮음 → 높음) 🔒 우선순위 상승!
High (높음) ⏰ 대기
Medium (중간) 😴 실행 불가
     ↓
Main이 빠르게 Lock 해제
     ↓
High 즉시 실행!
```

# 코드 / 실습
lock_acquire()
```c
void lock_acquire (struct lock *lock) {
	ASSERT (lock != NULL);
	ASSERT (!intr_context ());
	ASSERT (!lock_held_by_current_thread (lock));
	struct thread *t = thread_current();

	if(lock->holder != NULL) {
    	if(t->priority > lock->holder->priority) {
        	lock->holder->priority = t->priority;
    	}
	}

	sema_down (&lock->semaphore);
	lock->holder = thread_current ();
}
```

lock_release()
```c
void
lock_release (struct lock *lock) {
	ASSERT (lock != NULL);
	ASSERT (lock_held_by_current_thread (lock));
	struct thread *t = thread_current();
	
	t->priority = t->True_priority;
	lock->holder = NULL;
	sema_up (&lock->semaphore);
	thread_yield();
}

```

sema_up()
```c
void sema_up (struct semaphore *sema) {
	enum intr_level old_level;

	ASSERT (sema != NULL);

	old_level = intr_disable ();
	if (!list_empty (&sema->waiters)) {
		list_sort(&sema->waiters, c_thread_priority, NULL);
		thread_unblock (list_entry (list_pop_front (&sema->waiters), struct thread, elem));
	}
	sema->value++;
	intr_set_level (old_level);
}
```