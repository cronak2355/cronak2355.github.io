---
layout: post
title: "2025-11-11(pintos)"
date: 2025-11-12 10:40:50 +0900
categories: [KRAFTON JUNGLE]
tags: [Programming]
---
---

# Today's Log(2025-11-11)
>오늘은 Week09의 발제날이다 이번 주차부터 pintos 프로젝트에 들어간다.

---

# 오늘 한 일
- timer.c - timer_wakeup() 구현


# TIL
## ## Pintos Alarm 구현하기 - Sleep/Wakeup

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
  ↓
시간이 된 쓰레드 깨운 후
  ↓
Ready list에 추가
```

## timer_sleep()
### 핵심 역할
스레드를 BLOCKED(재우기)하고, 나중에 UNBLOCK(깨우기)할 수 있게 만들기

### 수행해야 할 작업들

#### 깨어날 시간 계산

현재 시간 파악 : 

timer_ticks() 함수로 현재 시간 가져오기
시스템 부팅 이후 경과한 tick 수

sleep list에서 맨 앞 sleep list의 쓰레드를 가져옴
**왜 맨 앞인가?**
timer_sleep()에서 정렬을 해주었고
고로 가장 빨리 깨워줘야 하는 쓰레드가 sleep list 앞에 있기 때문

현재 sleep list 값은 더 이상 필요하지 않으므로 list_pop_front()를 통해서 삭제해준다.

쓰레드 또한 thread_unblock()를 통하여 깨워준다.

이때 ready list에 정렬하여 넣어주어야 하는데 unblock 함수 안에 ready list에 넣어주는 함수가 있으나 정렬하여 넣어주지 않는다.

고로 list_insert_ordered()를 사용하여 ready list에 정렬하여 넣어준다.

ready list에 정렬하여 넣어준다면.

효율적으로 ready list를 사용할 수 있다.

매 tick마다 전체 리스트를 확인하는 건 비효율적이며

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
void thread_wake_up(void){
	int64_t cur_time = timer_ticks ();

	while (!list_empty(&sleep_list))
	{
		struct thread* t = list_entry(list_front(&sleep_list),struct thread,elem);
		if(t->wait_time > cur_time){
			break;
		}
		list_pop_front(&sleep_list);
		thread_unblock(t);
	}	
}
```