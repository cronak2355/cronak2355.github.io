---
layout: post
title: "2025-11-12(pintos)"
date: 2025-11-13 10:41:50 +0900
categories: [KRAFTON JUNGLE]
tags: [Programming]
---
---

# Today's Log(2025-11-12)
>오늘은 Week09의 수요일이다 이번 주차부터 pintos 프로젝트에 들어간다.

---

# 오늘 한 일
- 테스트 케이스 - priority-change 통과
- priority-donate 공부


# TIL
## Priority-change 통과하기
### 우선 순위에 따른 정렬
Priority-change를 통과하기 위해서는 우선순위(priority)를 기준으로 ready list를 정렬시켜주어야 한다.

#### 전체 흐름 이해하기
```
[스레드 A의 우선순위 변경 or 새로운 스레드가 ready_list에 추가됨]
  ↓
ready_list는 항상 우선순위에 따라 정렬된 상태를 유지
  ↓
현재 실행 중인 스레드 A와 ready_list의 첫 번째(가장 높은 우선순위) 쓰레드 B의 우선순위 비교
  ↓
A의 우선순위가 더 낮다면
  ↓
CPU를 B에게 양보 (thread_yield())
  ↓
아니라면 A는 그대로 실행 유지
```

## thread_priority_change()
### 수행해야 할 작업들
양보해야하는지 판단

#### priority(우선순위)비교

ready list에 값을 추가하는 과정은 thread_unblock()을 통해서 진행된다.

그렇기에 thread_unblock()에 ready list가 추가 된 후에 작업을 수행해 준다.

현재 쓰레드의 priority와 ready list안에 가장 앞에 있는 쓰레드의 priority를 비교하여 ready list의 priority가 더 크다면 양보 해준다.


# 코드 / 실습

```c
tid_t
thread_create (const char *name, int priority, thread_func *function, void *aux) {
	struct thread *t;
	tid_t tid;

	ASSERT (function != NULL);

	/* Allocate thread. */
	t = palloc_get_page (PAL_ZERO);
	if (t == NULL)
		return TID_ERROR;

	/* Initialize thread. */
	init_thread (t, name, priority);
	tid = t->tid = allocate_tid ();

	/* Call the kernel_thread if it scheduled.
	 * Note) rdi is 1st argument, and rsi is 2nd argument. */
	t->tf.rip = (uintptr_t) kernel_thread;
	t->tf.R.rdi = (uint64_t) function;
	t->tf.R.rsi = (uint64_t) aux;
	t->tf.ds = SEL_KDSEG;
	t->tf.es = SEL_KDSEG;
	t->tf.ss = SEL_KDSEG;
	t->tf.cs = SEL_KCSEG;
	t->tf.eflags = FLAG_IF;

	/* Add to run queue. */
	thread_unblock (t);
	thread_priority_change();
	
	return tid;
}

void thread_priority_change(void) {
	struct thread *t = thread_current();
	if(t -> priority < list_entry(list_front(&ready_list), struct thread, elem)) {
		thread_yield();
	}
}

void
thread_yield (void) {
	struct thread *cur = thread_current ();
	enum intr_level old_level;

	ASSERT (!intr_context ());

	old_level = intr_disable ();
	if (cur != idle_thread)
		list_insert_ordered(&ready_list, &cur->elem, c_thread_priority,NULL);
	do_schedule (THREAD_READY);
	intr_set_level (old_level);
}

void thread_set_priority (int new_priority) {
	struct thread *t = thread_current();
	intr_set_level(INTR_OFF);
	t->priority = new_priority;
	int pri = list_entry(list_front(&ready_list), struct thread, elem)->priority;
	if(t->priority < pri) {
		thread_yield();
	}
	intr_set_level(INTR_ON);
}


```