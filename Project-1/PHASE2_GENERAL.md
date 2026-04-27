## Threads: phase 2
**6개의 테스트 목록들**
현재 alarm-priority 초점으로 둔 현 로직에서 합불 상태 정리
| 테스트 명 | 합불여부 | 시나리오 특징 |
| :--- | :--- |  :--- | 
| priority-change | PASS | 실행 중인 스레드가 priority 낮췄을 때|
| priority-preempt | PASS | priority가 높은 새 스레드 출현 시|
| priority-fifo | PASS | priority가 동일한 스레드 여러 개 → 순차적 실행 보장 |
| priority-sema | FAIL | semaphore에서 기다리는 스레드 중 priority 높은 순 |
| priority-condvar | FAIL | conditional variable에서 대기 중 스레드 중 priority 높은 순 |
| alarm-priority | PASS | sleep 중인 스레드가 wake되었을 때, priority 순으로 선점 하는지 |

따라서 ... 현재 기본적인 조건은 만족, but 내부 대기열(sema, cond)에서는 깨짐.
<br>

### semaphore 정리
semaphore value = cpu 입장권(코어 수에 종속되지 않음 = 자원 허용량)
block은 thread 객체 내 상태 설정하는 변수, semaphore value는 전역(?)상태에서 스레드 허용량 체킹 변수

그럼 현재 로직의 문제는?
먼저 semaphore의 구조체를 보자:
```
struct semaphore {
	unsigned value;        
	struct list waiters;       
}
```
semaphore 구조체에는 list 형태의 스레드의 elem 형식으로 기다리고 있을 것으로 추정이 가능하다.
<br>

**문제의 코드 보기**
```
sema_up (struct semaphore *sema) {
	enum intr_level old_level = intr_disable ();
	
    if (!list_empty (&sema->waiters))
		thread_unblock (list_entry (list_pop_front (&sema->waiters),
					struct thread, elem));
	sema->value++;
	intr_set_level (old_level);
}
```
**해당 함수의 흐름**: interrupt 정지 → 리스트 앞 스레드 unblock(ready_list로 이양) → interrupt 활성화
**문제**: 본 테스트에서는 즉시 cpu에 우선순위 스레드가 선점하기를 기대하고 있음 (**현재는 delay 상태**)
**방안**: interrupt가 정지된 term 내에서 cpu내 yeild가 일어나야 함(현재는 timer_interrupt에서만 진행)
<br> 

**구현 및 수정**
1. delay 없이 즉시 yeild하는 함수 구현
	```
	yeid_without_interrupt(void){
		struct thread *t_fst = ready_list의 front 스레드 가져오기
		
		만약 cpu 점유 스레드보다 우선순위가 높다면 => 선점
		if(thread_current()->priority < t_fst->priority){
			thread_yield();
		}
	}
	```
<br>

2. sema_up에 즉시 양보 함수 호출
3. sema_down에서 waiter 오름차순(priority 기준)으로 반영하기
<br>

**결과 output**
```
perl -I../.. ../../tests/threads/priority-sema.ck tests/threads/priority-sema tests/threads/priority-sema.result
pass tests/threads/priority-sema
```
재 빌드 후 테스트가 통과되었음을 알 수 있다.
<br>

### condvar 정리
semaphore의 waits 리스트는 → prioirty 기준으로 정렬되어 있는 상태.
condvar는 이 waits 리스트를 정렬하는 더 높은 계층의 정렬(2차원 배열처럼 이해를 하면 쉽다).
condvar도 priority 순으로 정렬되어 있다... 그럼 굳이 왜 waits 상위로 배열을 하나 더 만들어야 하나?

**해답**: 2차원 배열 같지만, 각각 waiters는 하나만 가지고 있는 상황...
```
condvar.waiters
    │
    ├── semaphore_elem (스레드 A가 만든 것)
    │       └── semaphore → waiters: [스레드 A]
    │
    ├── semaphore_elem (스레드 B가 만든 것)
    │       └── semaphore → waiters: [스레드 B]
    │
    └── semaphore_elem (스레드 C가 만든 것)
            └── semaphore → waiters: [스레드 C]
``` 
왜 semaphore로 감싸야 할까? → 아직 이유를 모르겠음!!


**주요 함수**
cond_wait: 코어에 있는 스레드가 Lock 반납 후, Condvar waiters 큐로 이동
cond_signal: 특정 조건에서 호출, Condvar waiters에 있던 스레드가 Ready 큐로 이동 → Lock 획득 후 코어로
**여기서 주의할 점:** 스레드에 따라, Lock을 획득하거나 ... 획득 안하고 코어에 접근하는 경우가 있음.
<br>

**LOCK에 대한 이해**
왜 특정 스레드만 lock을 획득하는가: 공유 자원(ex. 힙 영역)에 접근(write)하는 스레드들은 Lock이 필요함.
why? → 일단 lock 유무 시나리오를 살펴보자.
```
LOCK이 걸려 있지 않았을 때:               
[ Core ]                  [ Ready Q ]          
    |                        |
 thread A (10)  <--run--  thread B (50)
    |                        |
 (expelled) ------------> thread A (10)
    |                        |
 thread B (50)  <--run--  (ready)
```
```
LOCK이 걸려 있을 때:
[ Core ]                  [ Ready Q ]          [ Lock Waiters ]
    |                        |                        |
 thread A (10) <--run--  thread B (50)                |
[has LOCK]                   |                        |
    |                        |                        |
 (yield) ------------->  thread A (10)                |
    |                        |                        |
 thread B (50) <--run--  (ready)                      |
    |                        |                        |
 (B: need something!)        |                        |
    | ------------------------------------------>  thread B (50)
    |                        |                      (sleep)
    |                        |                        |
 (LOCK owner back) <------- thread A (10)             |
```
스레드 B가 `뭔가 요청`을 할 경우, 우선순위가 낮은 스레드 B를 불러오고 자신의 우선순위를 임시로 준다.
전문 용어로 `우선순위 역적(Priority Inversion)`의 발생을 방지하기 위해서.
아... 근데 이건 phase 3에서 수행해야 하는 문제라고 함. 다음에 하겠음!