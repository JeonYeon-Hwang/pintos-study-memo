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
      int priority = PRI_DEFAULT - (i + 5) % 10 - 1;
      char name[16];
      snprintf (name, sizeof name, "priority %d", priority);
      thread_create (name, priority, alarm_priority_thread, NULL);
    }

  thread_set_priority (PRI_MIN);

  for (i = 0; i < 10; i++)
    sema_down (&wait_sema);
}

```
추가로 궁금한 점:
세마포어란? ... sema_init과 sema_down은 무엇을 위해서 있는 걸까?