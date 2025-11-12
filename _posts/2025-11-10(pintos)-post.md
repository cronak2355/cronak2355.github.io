---
layout: post
title: "2025-11-10(pintos)"
date: 2025-11-12 10:39:20 +0900
categories: [KRAFTON JUNGLE]
tags: [Programming]
---
---

# Today's Log(2025-11-10)
>오늘은 Week09의 발제날이다 이번 주차부터 pintos 프로젝트에 들어간다.

---

# 오늘 한 일
- timer.c - timer_sleep() 구현


# TIL
## Pintos Alarm 구현하기 - Sleep/Wakeup

#### Busy waiting을 제거하고 효율적인 sleep 기능을 구현하기

Pintos의 Alarm 구현은 크게 두 부분으로 나뉜다

- Sleep: 스레드를 재우는 부분 (timer_sleep)
- Wakeup: 스레드를 깨우는 부분 (timer_interrupt 또는 별도 함수)
- 이 두 부분을 sleep list라는 공유 자료구조를 통해 제어한다.

전체 흐름 이해하기
큰 그림
```
[스레드 A]
timer_sleep(100) 호출
  ↓
깨어날 시간 계산
  ↓
재운 후
  ↓
Sleep list에 추가
```
## timer_sleep()
### 핵심 역할
스레드를 BLOCKED(재우기)하고, 나중에 UNBLOCK(깨우기)할 수 있게 만들기

### 수행해야 할 작업들

#### 깨어날 시간 계산

현재 시간 파악 : 

timer_ticks() 함수로 현재 시간 가져오기
시스템 부팅 이후 경과한 tick 수

깨어날 시간 = 현재 시간 + 자야 할 시간
이 값을 thread 구조체에 저장

Sleep List에 추가
**왜 정렬이 필요한가?**

매 tick마다 전체 리스트를 확인하는 건 비효율적이며

깨어날 시간 순으로 정렬하게 된다면, 앞에서부터만 확인하면 됨으로 효율적으로 깨울 리스트를 알 수 있다. 

Pintos의 정렬 삽입:

list_insert_ordered() 함수 사용
비교 함수(comparator) wakeup_less() 필요
두 스레드의 wakeup_time을 비교

스레드 블락
thread_block()의 역할:

현재 스레드를 BLOCKED 상태로 변경
Ready_list에서 제거
스케줄러가 다른 스레드를 선택하게 함
이 함수는 리턴하지 않음 UNBLOCK(깨우기) 전까지







# 코드 / 실습

```c
/* Returns true if value A is less than value B, false
   otherwise. */
/* 값 A가 값 B보다 작으면 true를 반환하고,
   그렇지 않으면 false를 반환합니다. */
static bool
wakeup_less (const struct list_elem *a_, const struct list_elem *b_, void *aux UNUSED) 
{
  const struct thread *a = list_entry (a_, struct thread, elem);
  const struct thread *b = list_entry (b_, struct thread, elem);

  return a->wakeup_time < b->wakeup_time;
}

/* Suspends execution for approximately TICKS timer ticks. */
/* 현재 스레드의 실행을 약 TICKS 타이머 틱 동안 일시 중지. */
void
timer_sleep (int64_t ticks) { //멈춰야 할 시간 
	int64_t start = timer_ticks (); //start에 현재 ticks 시간을 담음 
	struct thread *t = thread_current();
	struct list the_sleep_list = sleep_list; 
	//block을 통해 스레드를 멈춘다음
	//현재 timer_ticks가 start + ticks와 같은 지 확인 후
	//같다면 unblock 후 thread_ready

	t->wakeup_time = start + ticks; // 깨어나야할 시간
	list_insert_ordered(&the_sleep_list, &t->elem, wakeup_less, 0); //sleep_list 정렬하기
	thread_block();

	// ASSERT (intr_get_level () == INTR_ON);
	// while (timer_elapsed (start) < ticks) {
	// 	thread_yield ();
	// }
	
}
```