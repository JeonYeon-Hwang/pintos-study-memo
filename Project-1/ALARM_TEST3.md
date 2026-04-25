## Threads
**alarm-priority**
alarm-priority란? → priority가 우위인 스레드가 CPU를 점유하도록 하는 로직.
따라서... 특정 복수의 스레드가 `동시에` wake 되었을 때만 로직이 수행.
어느 스레드가 `선점`하느냐가 테스트 통과의 관건.
<br>

**테스트 로그 분석하기**
```
Executing 'alarm-priority':
(alarm-priority) begin
(alarm-priority) Thread priority 23 woke up.
(alarm-priority) Thread priority 22 woke up.
(alarm-priority) Thread priority 21 woke up.
(alarm-priority) Thread priority 30 woke up.
(alarm-priority) Thread priority 29 woke up.
(alarm-priority) Thread priority 28 woke up.
(alarm-priority) Thread priority 27 woke up.
(alarm-priority) Thread priority 26 woke up.
(alarm-priority) Thread priority 25 woke up.
(alarm-priority) Thread priority 24 woke up.
(alarm-priority) end
```
해당 출력문은 ... cpu를 점유하였을 때 출력되는 문구임.
이들 중 몇몇은 시나리오 상, 동일한 시점에 wake 되도록 설계.
동시에 wake라면 → priority가 높은 것이 cpu 점유 ... 즉 내림차순으로 출력이 되어야 함.
<br>


**test_alarm_priority 함수 분석**
개요: 생성 시점은 차이가 나더라도 `wake_time은 동일`하도록 설계되었음
```
test_alarm_priority () 
{
  모든 스레드의 공통 wake 시각을 일괄 설정: 지금부터 5초 뒤  
  wake_time = timer_ticks () + 5 * TIMER_FREQ;
  sema_init (&wait_sema, 0);
  
  for (i = 0; i < 10; i++) 
    {
      int priority = 우선순위를 각 스레드 별로 설정(i 기준)

      스레드 생성 로직: 우선순위 주입 & 각 스레드는 alarm_priority_thread로 바톤터치
      thread_create (name, priority, alarm_priority_thread, NULL);
    }

  thread_set_priority (PRI_MIN);

  여기서 메인 스레드에 접근
  for (i = 0; i < 10; i++)
    sema_down (&wait_sema);
}

```
추가로 궁금한 점:
세마포어란? → 공유 자원을 사용할 수 있도록 허가하는 `열쇠`와 같은 역할. 열쇠의 갯수(S)를 설정.
P Operation: 공유자원 접근함 → S -= 1
S Operation: 공유자원 사용후 → S += 1

현 함수에서 사용하는 세마포어: 각 스레드 실행 이후 접근하는 메인스레드(wait_sema)의 접근권 관리
처음엔 ... sema_init()를 0으로 설정하여 '메인 스레드'가 타 요소의 접근 자체를 봉쇄
다음엔 ... sema_up()을 하여 '메인 스레드'가 접근 가능권을 +1 씩 증가
최종은 ... sema_down()로, '메인 스레드'에 접근하고 열쇠를 -1 씩 감소
<br>

**alarm_priority_thread 함수 분석**
개요: 요 함수는 각 스레드에서 개별로 실행이 됨.
```
alarm_priority_thread (void *aux UNUSED) 
{
  현재 시각을 시작 시간으로 설정
  int64_t start_time = timer_ticks ();
  틱 단위로 경계를 맞추기 위한 방어코드
  while (timer_elapsed (start_time) == 0)
    continue;

  위의 공통 wake_time에서 현 시간을 뺀 만큼 sleep함: wake 시간을 동일하게 설정하기 위해
  timer_sleep (wake_time - timer_ticks ());

  메인 스레드(wait_sema)에서 접근 가능권 +1 증가
  여기서 실행된 스레드가 메인 스레드에 접근권을 획득함
  sema_up (&wait_sema);
}

```
새롭게 안 사실: sema_ 요 함수들은 CPU를 먼저 점유한 순서에 종속된다.
<br><br>


### Priority 수정 전략
**스레드를 위한 queue는 두 가지가 있다**
Work 배분용 큐: 큐에 쌓인 `일`이 순차적으로 스레드들에 배분하는 역할
CPU 배분용 큐: 생성된 스레드들 중 어느 것을 cpu(코어의 연산 자원)에 배분하는 역할 → 좀 더 low 한 계층
<br>
... 따라서 ...
ready queue(cpu 타기 대기줄)
scheduler(다음 차례 스레드 고르기)
context switch(해당 스레드를 cpu에 올리기)
가 진행된다.
<br>

**수정해야할 함수들**
| 함수명 | 기능 | ready queue와 상호작용 | 
| :--- | :---- | :--- | 
| thread_create() | 새 스레드를 만듦 | 새롭게 실행할 후보를 q에 추가 | 
| thread_unblock() | wake한 스레드를 runnable하게 | 해당 후보 스레드를 q에 추가 | 
| thread_yield() | 현재 실행 중인 스레드가 스스로 양보 | 다른 후보 선택 후 q에 해당 스레드 넣기 | 
|thread_set_priority()| 현재 실행 중인 스레드 우선순위 변경| |

어떻게 수정해야 하나?
thread_create() → priority가 높은 스레드를 해당 위치의 q에 중간 삽입하기(현재는 FIFO 상황)
thread_unblock() → 마찬가지로 다시 q에 넣어야 하는 상황이니 중간 삽입하기
thread_yield() → 동일함. 다시 q에 넣어야 하는 만큼 중간 삽입
<br>

**thread 구조체**
```
struct thread {
    ...
    struct list_elem elem;
    ...
}

struct list_elem {
    struct list_elem *prev;
    struct list_elem *next;
}
```
연결 방식은 대략 다음과 같다:
thread A의 elem <-> thread B의 elem <-> thread C의 elem
<br>

**수정 및 구현**
1. 다음과 같은 함수를 추가로 구현:
    ```
    /* ready_list 중간 삽입을 위한 helper 함수 선언부 */
    /* 순회하면서 어디에 순차적으로 삽입할 지 결정 */
    void 
    list_push_ordered (struct list *list, struct list_elem *elem){
      /* head 부터 구하기 */
      struct list_elem *e;

      /* 순회를 진행한다 */
      for(e = list_begin(list); e != list_end(list); e = list_next(e)){
        if(cmp_priority(e, elem)){
          list_insert(e, elem);
          return;
        }
      }

      /* 만약 삽입할 곳을 못 찾았다면 => 마지막에 넣기 */
      list_push_back(list, elem);
    }

    /* 현재와 이후의 thread의 priority 비교 */
    bool 
    cmp_priority(struct list_elem *cur, struct list_elem *new){
      /* list_elem을 가지고 상위 객체인 thread를 찾기 */
      struct thread *tcur = list_entry(cur, struct thread, elem);
      struct thread *tnew = list_entry(new, struct thread, elem);

      return tcur->priority < tnew->priority;
    }
    ```
    위의 list_push_ordered 함수를 기존 각각 list_push_back 함수 호출부를 대체함.
    <br>
    
2. 이외에 수정할 영역: 즉시 양보 로직 구현
    ```
    if (priority > thread_current()->priority)
    thread_yield();
    ```
    새롭게 생성된 스레드가 현재 돌아가는 스레드 보다 priority가 높다면 즉시 양보할 것
    <br>

3. 현 시나리오는 단일 코어를 사용하는 것을 가정
   thread_current ()->priority 이 메서드에서, 현재 스레드가 하나로만 특정이 가능함.
    ```    
    thread_set_priority (int new_priority) {
      thread_current ()->priority = new_priority;
      /* 현재의 스레드를 구한다 */
      struct thread *tcur = thread_current();

      if(!list_empty(&ready_list)){
        struct list_elem *e = list_front(&ready_list);
        struct thread *front = list_entry(e, struct thread, elem);

        if(front->priority > tcur->priority){
          thread_yield();
        }
      }
    }
    ```
    설명: 현 스레드의 priority를 new_priority로 갱신 
    → list가 비어있지 않다면? → front의 스레드를 가져와서 priority를 비교
    → new_priority가 작다면? → thread_yeid하여 양보 진행

<br>

**추가구현 & 수정 후 결과**
tests/threads/alarm-priority.result:PASS