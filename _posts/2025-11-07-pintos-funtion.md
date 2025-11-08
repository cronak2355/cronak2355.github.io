---
layout: post
title: "2025-11-07"
date: 2025-11-07 22:29:31 +0900
categories: [KRAFTON JUNGLE]
tags: [Programming]
---
---
# Pintos 주요 함수 레퍼런스

> Pintos 프로젝트에서 자주 사용되는 핵심 함수들을 카테고리별로 정리한 가이드입니다.

---

## 목차
1. [Thread 관련 함수](#thread-관련-함수)
2. [동기화 함수](#동기화-함수)
3. [Timer 관련 함수](#timer-관련-함수)
4. [Interrupt 관련 함수](#interrupt-관련-함수)
5. [List 자료구조 함수](#list-자료구조-함수)

---

## Thread 관련 함수

### `thread_create()`
```c
tid_t thread_create (const char *name, int priority,
                     thread_func *function, void *aux)
```

**설명:**
- 새로운 커널 스레드를 생성하고 ready queue에 추가
- 생성된 스레드는 즉시 스케줄링될 수 있음

**매개변수:**
- `name`: 스레드 이름 (디버깅용)
- `priority`: 초기 우선순위
- `function`: 실행할 함수 포인터
- `aux`: 함수에 전달할 인자

**반환값:**
- 성공 시: 새 스레드의 tid
- 실패 시: `TID_ERROR`

**주의사항:**
- `thread_start()` 호출 후에는 새 스레드가 `thread_create()` 반환 전에 실행될 수 있음
- 순서 보장이 필요하면 세마포어 등 동기화 사용

---

### `thread_block()`
```c
void thread_block (void)
```

**설명:**
- 현재 스레드를 blocked 상태로 전환
- `thread_unblock()`이 호출될 때까지 스케줄링되지 않음

**사용 조건:**
- 인터럽트가 꺼진 상태에서 호출해야 함
- 인터럽트 핸들러 내에서 호출 불가

**주의사항:**
- 직접 사용하기보다는 `synch.h`의 동기화 primitives 사용 권장
- 반드시 `intr_disable()` 후 호출

---

### `thread_unblock()`
```c
void thread_unblock (struct thread *t)
```

**설명:**
- blocked 상태의 스레드를 ready 상태로 전환
- ready queue에 추가하지만 즉시 선점하지는 않음

**매개변수:**
- `t`: unblock할 스레드 포인터

**주의사항:**
- 스레드가 BLOCKED 상태여야 함
- running 스레드를 ready로 만들 때는 `thread_yield()` 사용

---

### `thread_yield()`
```c
void thread_yield (void)
```

**설명:**
- 현재 실행 중인 스레드가 CPU를 자발적으로 양보
- 스레드는 ready queue로 이동하고 스케줄러가 다음 스레드 선택

**사용 시기:**
- Busy waiting 대신 사용
- 우선순위 변경 후 재스케줄링 필요할 때

**주의사항:**
- 즉시 다시 실행될 수 있음 (스케줄러 재량)
- 인터럽트 핸들러에서 호출 불가

---

### `thread_current()`
```c
struct thread *thread_current (void)
```

**설명:**
- 현재 실행 중인 스레드의 포인터 반환

**반환값:**
- 현재 running 스레드의 `struct thread *`

**사용 예시:**
```c
struct thread *cur = thread_current();
printf("Current thread: %s\n", cur->name);
```

---

### `thread_tid()`
```c
tid_t thread_tid (void)
```

**설명:**
- 현재 실행 중인 스레드의 thread ID 반환

**반환값:**
- 현재 스레드의 tid

---

### `thread_exit()`
```c
void thread_exit (void) NO_RETURN
```

**설명:**
- 현재 스레드를 종료하고 스케줄링에서 제거
- 이 함수는 절대 반환하지 않음

**사용 시기:**
- 스레드 함수가 완료되었을 때
- 명시적으로 스레드를 종료해야 할 때

---

### `thread_set_priority()` / `thread_get_priority()`
```c
void thread_set_priority (int new_priority)
int thread_get_priority (void)
```

**설명:**
- 현재 스레드의 우선순위 설정/조회

**매개변수:**
- `new_priority`: 새로운 우선순위 값 (0-63)

**주의사항:**
- Priority donation 구현 시 수정 필요
- MLFQS 모드에서는 사용 불가

---

## 동기화 함수

### Semaphore

#### `sema_init()`
```c
void sema_init (struct semaphore *sema, unsigned value)
```

**설명:**
- 세마포어를 주어진 값으로 초기화

**매개변수:**
- `sema`: 초기화할 세마포어 포인터
- `value`: 초기 값 (보통 0 또는 1)

---

#### `sema_down()`
```c
void sema_down (struct semaphore *sema)
```

**설명:**
- P 연산 (Wait)
- 세마포어 값이 0이면 대기, 양수면 1 감소 후 진행

**동작:**
1. 값이 0이면 waiters 리스트에 추가되고 블록
2. 값이 양수면 1 감소하고 계속 실행

**주의사항:**
- 인터럽트 핸들러에서 호출 불가 (sleep 가능)

---

#### `sema_up()`
```c
void sema_up (struct semaphore *sema)
```

**설명:**
- V 연산 (Signal)
- 세마포어 값을 1 증가하고, 대기 중인 스레드가 있으면 하나를 깨움

**주의사항:**
- 인터럽트 핸들러에서도 호출 가능

---

#### `sema_try_down()`
```c
bool sema_try_down (struct semaphore *sema)
```

**설명:**
- 블록하지 않는 P 연산
- 값이 0이면 실패, 양수면 감소

**반환값:**
- 성공 시 `true`, 실패 시 `false`

---

### Lock

#### `lock_init()`
```c
void lock_init (struct lock *lock)
```

**설명:**
- Lock 초기화
- Lock은 value=1인 세마포어의 특수한 형태

---

#### `lock_acquire()`
```c
void lock_acquire (struct lock *lock)
```

**설명:**
- Lock을 획득, 사용 중이면 대기

**주의사항:**
- 같은 스레드가 이미 획득한 lock을 다시 acquire하면 안 됨 (non-recursive)
- 인터럽트 핸들러에서 호출 불가

---

#### `lock_release()`
```c
void lock_release (struct lock *lock)
```

**설명:**
- Lock을 해제하고 대기 중인 스레드 깨움

**주의사항:**
- 현재 스레드가 lock의 holder여야 함

---

#### `lock_held_by_current_thread()`
```c
bool lock_held_by_current_thread (const struct lock *lock)
```

**설명:**
- 현재 스레드가 lock을 보유하고 있는지 확인

**반환값:**
- 보유 중이면 `true`, 아니면 `false`

---

### Condition Variable

#### `cond_init()`
```c
void cond_init (struct condition *cond)
```

**설명:**
- Condition variable 초기화

---

#### `cond_wait()`
```c
void cond_wait (struct condition *cond, struct lock *lock)
```

**설명:**
- Condition을 기다리며 대기
- Lock을 원자적으로 해제하고 signal을 기다림
- Signal 받으면 lock을 다시 획득하고 반환

**Mesa-style:**
- signal 받아도 조건을 다시 확인해야 함

**사용 패턴:**
```c
lock_acquire(&lock);
while (!condition)
    cond_wait(&cond, &lock);
// critical section
lock_release(&lock);
```

---

#### `cond_signal()`
```c
void cond_signal (struct condition *cond, struct lock *lock)
```

**설명:**
- 대기 중인 스레드 하나를 깨움

**주의사항:**
- Lock을 보유한 상태에서 호출해야 함

---

#### `cond_broadcast()`
```c
void cond_broadcast (struct condition *cond, struct lock *lock)
```

**설명:**
- 대기 중인 모든 스레드를 깨움

---

## Timer 관련 함수

### `timer_ticks()`
```c
int64_t timer_ticks (void)
```

**설명:**
- OS 부팅 이후 경과한 timer tick 수 반환

**반환값:**
- 경과한 tick 수 (int64_t)

**활용:**
- 시간 측정의 기준점

---

### `timer_elapsed()`
```c
int64_t timer_elapsed (int64_t then)
```

**설명:**
- 주어진 시점 이후 경과한 tick 수 계산

**매개변수:**
- `then`: 과거의 tick 값

**반환값:**
- `timer_ticks() - then`

**사용 예시:**
```c
int64_t start = timer_ticks();
// ... 작업 수행 ...
int64_t elapsed = timer_elapsed(start);
```

---

### `timer_sleep()`
```c
void timer_sleep (int64_t ticks)
```

**설명:**
- 지정된 tick 동안 실행을 중단

**현재 구현 (기본):**
- Busy waiting 방식 (비효율적)
- `thread_yield()` 반복 호출

**프로젝트 목표:**
- Busy waiting을 제거하고 효율적인 sleep 구현
- Blocked 상태로 전환 후 wake-up time에 unblock

**주의사항:**
- `ASSERT (intr_get_level () == INTR_ON)` - 인터럽트가 켜진 상태여야 함

---

### `timer_msleep()` / `timer_usleep()` / `timer_nsleep()`
```c
void timer_msleep (int64_t ms)
void timer_usleep (int64_t us)
void timer_nsleep (int64_t ns)
```

**설명:**
- 밀리초/마이크로초/나노초 단위로 sleep
- 내부적으로 tick 단위로 변환하여 `timer_sleep()` 호출

---

## Interrupt 관련 함수

### `intr_disable()`
```c
enum intr_level intr_disable (void)
```

**설명:**
- 인터럽트를 끄고 이전 인터럽트 레벨 반환

**반환값:**
- 이전 인터럽트 상태 (`INTR_ON` 또는 `INTR_OFF`)

**사용 패턴:**
```c
enum intr_level old_level = intr_disable();
// critical section
intr_set_level(old_level);
```

---

### `intr_enable()`
```c
enum intr_level intr_enable (void)
```

**설명:**
- 인터럽트를 켜고 이전 인터럽트 레벨 반환

---

### `intr_set_level()`
```c
enum intr_level intr_set_level (enum intr_level level)
```

**설명:**
- 인터럽트를 지정된 레벨로 설정

**매개변수:**
- `level`: `INTR_ON` 또는 `INTR_OFF`

---

### `intr_get_level()`
```c
enum intr_level intr_get_level (void)
```

**설명:**
- 현재 인터럽트 레벨 반환

**반환값:**
- `INTR_ON` 또는 `INTR_OFF`

---

### `intr_context()`
```c
bool intr_context (void)
```

**설명:**
- 현재 인터럽트 핸들러 내에서 실행 중인지 확인

**반환값:**
- 인터럽트 핸들러 내부면 `true`, 아니면 `false`

**사용 시기:**
- 특정 함수가 인터럽트 핸들러에서 호출되면 안 될 때 검사

---

## List 자료구조 함수

### 기본 개념

Pintos의 List는 **침입형(intrusive) 이중 연결 리스트**입니다.
- `struct list_elem`을 구조체 멤버로 포함
- `list_entry()` 매크로로 원래 구조체 포인터 획득

**구조:**
```c
struct list_elem {
    struct list_elem *prev;
    struct list_elem *next;
};

struct list {
    struct list_elem head;  // sentinel
    struct list_elem tail;  // sentinel
};
```

---

### `list_init()`
```c
void list_init (struct list *list)
```

**설명:**
- 빈 리스트로 초기화

---

### `list_push_back()` / `list_push_front()`
```c
void list_push_back (struct list *list, struct list_elem *elem)
void list_push_front (struct list *list, struct list_elem *elem)
```

**설명:**
- 리스트의 뒤/앞에 요소 추가

**사용 예시:**
```c
struct thread *t = ...;
list_push_back(&ready_list, &t->elem);
```

---

### `list_pop_back()` / `list_pop_front()`
```c
struct list_elem *list_pop_back (struct list *list)
struct list_elem *list_pop_front (struct list *list)
```

**설명:**
- 리스트의 뒤/앞 요소 제거 및 반환

**주의사항:**
- 리스트가 비어있으면 undefined behavior

---

### `list_remove()`
```c
struct list_elem *list_remove (struct list_elem *elem)
```

**설명:**
- 리스트에서 특정 요소를 제거하고 다음 요소 반환

**반환값:**
- 제거된 요소의 다음 요소

**올바른 사용법:**
```c
// 올바른 방법
for (e = list_begin(&list); e != list_end(&list); e = list_remove(e)) {
    // process e
}
```

**잘못된 사용법:**
```c
// 잘못된 방법 - 제거 후 list_next() 사용하면 안 됨
for (e = list_begin(&list); e != list_end(&list); e = list_next(e)) {
    list_remove(e);  // 위험!
}
```

---

### `list_begin()` / `list_end()`
```c
struct list_elem *list_begin (struct list *list)
struct list_elem *list_end (struct list *list)
```

**설명:**
- 리스트의 시작/끝 위치 반환
- `list_end()`는 마지막 요소가 아닌 tail sentinel

**사용 패턴:**
```c
for (struct list_elem *e = list_begin(&list); 
     e != list_end(&list); 
     e = list_next(e)) {
    // process e
}
```

---

### `list_next()` / `list_prev()`
```c
struct list_elem *list_next (struct list_elem *elem)
struct list_elem *list_prev (struct list_elem *elem)
```

**설명:**
- 다음/이전 요소 반환

---

### `list_empty()`
```c
bool list_empty (struct list *list)
```

**설명:**
- 리스트가 비어있는지 확인

**반환값:**
- 비어있으면 `true`, 아니면 `false`

---

### `list_size()`
```c
size_t list_size (struct list *list)
```

**설명:**
- 리스트의 요소 개수 반환

**복잡도:**
- O(n) - 전체 순회 필요

---

### `list_entry()` 매크로
```c
#define list_entry(LIST_ELEM, STRUCT, MEMBER)
```

**설명:**
- `list_elem` 포인터에서 해당 구조체 포인터를 얻음

**사용 예시:**
```c
struct thread {
    tid_t tid;
    struct list_elem elem;
    // ...
};

struct list_elem *e = list_begin(&ready_list);
struct thread *t = list_entry(e, struct thread, elem);
```

**동작 원리:**
- `offsetof` 매크로로 멤버의 오프셋 계산
- 포인터 산술로 구조체 시작 주소 계산

---

### `list_insert_ordered()`
```c
void list_insert_ordered (struct list *list, struct list_elem *elem,
                         list_less_func *less, void *aux)
```

**설명:**
- 정렬된 순서를 유지하며 요소 삽입

**매개변수:**
- `less`: 비교 함수 포인터
- `aux`: 비교 함수에 전달할 추가 데이터

**비교 함수 타입:**
```c
typedef bool list_less_func (const struct list_elem *a,
                            const struct list_elem *b,
                            void *aux);
```

**사용 예시:**
```c
bool
priority_less (const struct list_elem *a, 
               const struct list_elem *b, 
               void *aux UNUSED) {
    struct thread *ta = list_entry(a, struct thread, elem);
    struct thread *tb = list_entry(b, struct thread, elem);
    return ta->priority < tb->priority;
}

list_insert_ordered(&ready_list, &t->elem, priority_less, NULL);
```

---

### `list_sort()`
```c
void list_sort (struct list *list, list_less_func *less, void *aux)
```

**설명:**
- 리스트를 정렬 (stable merge sort 사용)

---

## 중요 매크로 및 상수

### Thread Priority
```c
#define PRI_MIN 0          // 최저 우선순위
#define PRI_DEFAULT 31     // 기본 우선순위
#define PRI_MAX 63         // 최고 우선순위
```

### Thread Status
```c
enum thread_status {
    THREAD_RUNNING,     // 현재 실행 중
    THREAD_READY,       // 실행 준비 완료
    THREAD_BLOCKED,     // 대기 중 (이벤트 대기)
    THREAD_DYING        // 종료 중
};
```

### Timer Frequency
```c
#define TIMER_FREQ 100  // 초당 100 ticks
```

---

## 자주 하는 실수

### 1. 인터럽트 제어 실수
```c
// ❌ 나쁜 예
thread_block();  // 인터럽트를 끄지 않고 호출

// ✅ 좋은 예
enum intr_level old_level = intr_disable();
thread_block();
intr_set_level(old_level);
```

### 2. List 순회 중 제거 실수
```c
// ❌ 나쁜 예
for (e = list_begin(&list); e != list_end(&list); e = list_next(e)) {
    list_remove(e);  // 이미 제거된 노드의 next 참조
}

// ✅ 좋은 예
for (e = list_begin(&list); e != list_end(&list); e = list_remove(e)) {
    // 제거하면서 다음 노드 반환
}
```

### 3. list_entry() 잘못 사용
```c
// ❌ 잘못된 멤버 이름
struct thread *t = list_entry(e, struct thread, tid);  // tid는 list_elem이 아님

// ✅ 올바른 멤버 이름
struct thread *t = list_entry(e, struct thread, elem);
```

### 4. Lock 없이 공유 자원 접근
```c
// ❌ Race condition 발생
shared_variable++;

// ✅ Lock으로 보호
lock_acquire(&lock);
shared_variable++;
lock_release(&lock);
```

---