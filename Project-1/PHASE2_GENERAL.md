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

**semaphore 정리**
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