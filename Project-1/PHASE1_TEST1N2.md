## Threads
**threads 폴더 내의 역할**: 
스레드 생성
우선순위/스케줄링 정책 적용 → 문맥 전환(실행할 스레드 선택 등)
<br>

**alarm의 역할**:
스레드를 일정 시간 동안 잠들고(sleep) 정해신 시간에 다시 runnable 하도록 함
왜? 일이 없을 때 굳이 그 스레드를 안돌릴려고
<br>

**초기 구현 상태**
| multiple | negative | priority | simultaneous | single | zero |
| :--- | :----: | :---: | :---: | :---: | ---: |
| PASS | PASS | FAIL | PASS | PASS | PASS |

priority 시나리오에 대응하여, 함수와 로직을 보완해야 함

<br>

alarm에는 다양한 테스트 시나리오가 존재함.
1. alarm-single: 
    ```
    Executing 'alarm-single':
    (alarm-single) begin
    (alarm-single) Creating 5 threads to sleep 1 times each.
    (alarm-single) Thread 0 sleeps 10 ticks each time,
    (alarm-single) thread 1 sleeps 20 ticks each time, and so on.
    (alarm-single) If successful, product of iteration count and
    (alarm-single) sleep duration will appear in nondescending order.
    (alarm-single) thread 0: duration=10, iteration=1, product=10
    (alarm-single) thread 1: duration=20, iteration=1, product=20
    (alarm-single) thread 2: duration=30, iteration=1, product=30
    (alarm-single) thread 3: duration=40, iteration=1, product=40
    (alarm-single) thread 4: duration=50, iteration=1, product=50
    (alarm-single) end
    Execution of 'alarm-single' complete.
    Thread: 0 idle ticks, 290 kernel ticks, 0 user ticks
    ```
    스레드 5개 생성 → (거의) 동시에 각기 다른 sleep 시간 명령 → ticks 마다 wake스레드 추적
    현재 로그로: 총 290의 `커널` 영역 ticks가 진행됨을 알 수 있음
<br>

2. alarm-multiple:
    ```
    Executing 'alarm-multiple':
    (alarm-multiple) begin
    (alarm-multiple) Creating 5 threads to sleep 7 times each.
    (alarm-multiple) Thread 0 sleeps 10 ticks each time,
    (alarm-multiple) thread 1 sleeps 20 ticks each time, and so on.
    (alarm-multiple) If successful, product of iteration count and
    (alarm-multiple) sleep duration will appear in nondescending order.
    (alarm-multiple) thread 0: duration=10, iteration=1, product=10
    (alarm-multiple) thread 0: duration=10, iteration=2, product=20
    (alarm-multiple) thread 1: duration=20, iteration=1, product=20
    (alarm-multiple) thread 0: duration=10, iteration=3, product=30
    (alarm-multiple) thread 2: duration=30, iteration=1, product=30
    (alarm-multiple) thread 0: duration=10, iteration=4, product=40
    (alarm-multiple) thread 1: duration=20, iteration=2, product=40
    (alarm-multiple) thread 3: duration=40, iteration=1, product=40

    ---------- 생략 ----------

    (alarm-multiple) thread 2: duration=30, iteration=7, product=210
    (alarm-multiple) thread 3: duration=40, iteration=6, product=240
    (alarm-multiple) thread 4: duration=50, iteration=5, product=250
    (alarm-multiple) thread 3: duration=40, iteration=7, product=280
    (alarm-multiple) thread 4: duration=50, iteration=6, product=300
    (alarm-multiple) thread 4: duration=50, iteration=7, product=350
    (alarm-multiple) end
    Execution of 'alarm-multiple' complete.
    Timer: 624 ticks
    Thread: 0 idle ticks, 624 kernel ticks, 0 user ticks
    ```
    왜 멀티플인가? → 하나의 스레드를 여러 번 깨우고 sleep을 함(iteration이 그 횟수)
    각 스레드마다 sleep할 명령이 대기하고 있음
<br>


**함수 호출 순서**:
```
root 호출
└── thread/init.c의 main()
    ├── 각종 init 수행 ... thread_start()로 스레드 준비 완료
    └── run_actions() ... 테스트 시행 함수
        잘은 모르겠으나, struct에 run_task를 넣고, while에서 호출: a->function(argv)
        └── run_test(테스트 명) ... 여기서 test.c 스크립트로 넘어감
            └── test_alarm_single() ... 테스트 시나리오(호출 함수 결합): 개별 테스트 스크립트
                └── test_sleep(인자값) .... 본 함수에서 테스트 수행 동작 명시     
```
**특징**: 
현재 상황: 구체적인 지시 사항이 `테스트 스크립트`에 결합되어 있음.
why? → 실행할 유저 프로그램을 받는 로직이 구현되어 있지 않기 때문.
추후 진행: Project 2에서 ... `상위 명령`이 구현되어, 내부 코드가 지시사항이 만들어짐
<br>

**test_sleep 함수 분석**
```
test_sleep (스레드 수, 반복 횟수) 
{
  구조체 및 변수 선언
  메모리 할당
  초기화 수행: 공통 시간 설정, 락 준비

  1차 반복문: 각 스레드 생성 & 구조체 생성 => 주입
  for (i = 0; i < 스레드 수; i++)
    {
      struct sleep_thread *t = threads + i;
      
      구조체 조건을 설정(duration = 인덱스 기준)
      t->duration = (i + 1) * 10;
      t->iterations = 0;

      스레드 생성 및 구조체 조건 주입
      sleeper 함수 주의!! => 해당 함수 내부 루프에서 스레드가 반복적 수행됨
      thread_create (name, PRI_DEFAULT, sleeper, t);
    }
  
  2차 반복문: 테스트 시나리오 수행 후, 남겨진 기록을 검사
  for (op = output; op < test.output_pos; op++) 
    {
      저장한 기록문(op)를 순차적으로 살펴봄
      
      로그 기록이 예상과 다를 불협화음?이 감지된다면?
      if (new_prod >= product)
        product = new_prod;
      else
        fail (틀린 경우를 출력함);
    }
  
   락 & 메모리 해제
}
```
<br>

**sleeper 함수 분석**
정리: 자기 duration에 맞춰 반복해서 sleep 수행 & 행동을 기록
```
sleeper (해당 스레드의 정보) 
{
  변수 선언

  for (i = 1; i <= 반복 가능 한도; i++) 
    {
      int64_t sleep_until = 목표 시각(깨어날)을 계산
      
      실제로 sleep 하도록 수행
      timer_sleep (정해진 시간);

      lock_acquire & lock_release을 수행
    }
}
```
lock은 여기서 무슨 역할은 하는가: 
놀랍게도 단순 기록? 혼선 방지용이라고 함.
추후에는 동시성 문제 등을 해결하기 위해서 사용이 될 것이라고...
