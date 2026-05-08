## System call: Process control

### Exec
**exec이란?**
현 프로세스의 이미지(?)를 교체하는 것. → 아니, 뭔말이야?

두 syscall 비교를 통해 이해해보자 ... 
**Fork**: 부모 프로세스를 통째로 복사하여 자식 프로세스를 만듦.
**Exec**: 현재의 <u>context</u>를 버리고 새로운 <u>파일</u>을 실행함.

context란? → 다음과 같은 것이 context:
프로그램 카운터: 여기까지 읽었다고 표시한 변수
레지스터 값: CPU에 임시로 저장된 데이터들
메모리 공간: 현 프로그램 용 변수, 작업 기록(Stack) 등.

피일이란? → 프로그램의 원본은 파일이기 때문:
프로그램은 저장장치(SSD/HDD)에 잠자고 있는 `실행 파일`이다.
잠자고 있다가, 실행 상태로 변경 시 프로세스에 이 데이터가 적재.

따라서 context 교체란? 
→ 프로세서를 그대로 유지를 함.
→ 그러나 기존 프로그램용 각종 데이터를 싹다 버림.
→ 새 프로그램을 로드하고 대상 데이터로 다시 채운다고 생각하면 좋음.
<br>

**어떻게 구현되어야 할까?**
먼저 유저 프로그램 main을 보자:
```
void test_main (void) 
{
  msg ("I'm your father");
  exec ("child-simple");
}
```
msg: <u>write</u>라는 syscall 호출. 유저 스택에 인자를 쌓고 출력.
exec: <u>exec</u>라는 syscall 호출. argument passing 진행.

두 call에서 유저 스택에 인자를 넣지만, 각각 용도나 의도가 다르다:
| 구분 | exec(argument passing) | write |
| :--- | :--- | :--- |
| 주체 | 커널에서 주도적으로 시행 | 유저 프로세스 |
| 위치 | 유저 스택 맨 top | 특정 스택 프레임 |
| 목적 | main(argc, argv)에 전달 | syscall함수 매개변수 |
| 기간 | 프로그램 종료까지 가능 | syscall 호출 직후 소멸 |

<br>

**exec 호출 시 실행 순서**
`syscall(EXEC)` → `cmd_l을 palloc` → `process_exec(cmd_l)` → `status 반환`
<br>


### Fork
**Fork이란?**
앞서 말했다시피, 부모 프로세스를 통째로 복제하여 생성하는 것
*보통 Fork 시행 후 새 자식에게 → Exec하여 새 환경을 셋팅*

호출 시 실행 순서를 살펴보자:
`syscall(FORK)` → `process_fork함수 분기` → `tid_t 결과값 반환`

**함수 살펴보기**
예외처리 분기는 생략되어 있다:
```
process_fork (이름, intr_frame *if_) {
  struct thread *curr = 현 스레드
  struct child_status *cs = 구조체1
  struct fork_args *args = 구조체2

  새 cs 구조체 값 설정 => {
    tid = TID_ERROR
    exit_status = -1
    waited, exited, fork_sucess = false
    fork_sema, wait_sema => sema_init하기
  }

  현 스레드의 child_status_list에 elem에 설정한 cs구조체 넣기

  args 구조체 값 설정 => {
    parent = 현 스레드
    if_ = 현 if_ 값들
    cs = 새 cs
  }

  이제 자식 스레드를 생성한다
  tid = thread_create (이름, PRI_DEFAULT, __do_fork, args 구조체);

  .. 잘은 모르겠지만, 추후 배선 작업 같음 ..
  args = NULL;
  cs->tid = tid;
  sema_down (&cs->fork_sema);

  return tid;
}
```
<br>

여기서 스레드가 생성되는 동시에 __do_fork 함수로 과정을 밟게 됢:
→ __do_fork 함수 살펴보기:
```
__do_fork (void *aux) {
  struct fork_args *args = 부모의 args 구조체
  struct thread *curr = 현 자식 스레드
  struct thread *parent => 부모가 될 스레드?
  struct intr_frame if_ => 레지스터 상태

  parent = args에서 부모 스레드를 할당한다.
  curr->self_status = args에서 cs를 할당한다.

  부모가 멈췄던 순간의 포즈(레지스터) 값을 그대로 if_에 복사
  memcpy (&if_, &args->if_, sizeof if_);
  curr->pml4 = 독자 메모리 공간 생성

  process_activate (curr) => 현 프로세스를 활성화한다.

  .. 잘 이해는 못하겠지만 페이지 테이블을 복사하는 무언가라고 함 ..
  if (!pml4_for_each (parent->pml4, duplicate_pte, parent))

  자식 프로세스에서 fork()의 반환값?은 0이다.
  if_.R.rax = 0;

  process_init () => 아마 활성화 → 개시 순 일 듯?
  현 스레드의 상태값 중 fork_success = "성공"으로
  부모에게 자신이 생성되었음을 알림.
  sema_up (&curr->self_status->fork_sema);

  do_iret (&if_);
}
```
두 함수의 흐름과 구조를 정리하자!
... 1차 process_fork: 부모 스레드의 상태값 → 구조체 args{ cs{ } }에 저장
... 2차 thread_create: 자식 스레드 생성 & 구조체와 __do_fork함수 주입
... 3차 __do_fork: 구조체에서 상태값 가져오기, 자식 스레드에 주입하기
... 이후에는 → 자식 프로세스 활성화 & fb_table을 복제해서 넣어주기

*왜 fb_table을 바로 복제가 되지 않을까?*
테이블에 저장된 주소값은 파일 전용 상태(file 구조체)의 주소를 가짐
이것은 일종의 통로로, 만약 복제된 테이블이 동일한 `통로 주소값`을 가지면,
두 스레드가 하나의 통로 가지고 사용하는 충돌이 발생!

→ 하나의 파일이라 해도 연결 통로는 각 스레드 마다 가지고 있어야 한다.
따라서 file_duplicate를 활용하여 통로 또한 복제하여 fb_table에 주입해야 함.
<br>

### 프로세스 생애주기
**process_wait가 생애주기를 주관한다**
How? → 먼저 pintos 코어는 1개, 동시에 스레드 1개만 점유가 가능하다(교체를 해야 함)
process_wait 호출 시: 해당 부모 스레드의 child_list를 순회 → exit한 child를 remove

process_wait은 언제 호출 되는가? 
.. 자식 스레드가 process_exit호출: sema_up(wait_sema)가 발동
.. sama에서 잠들어 있던 부모 스레드가 ready_list로 이동 & 실행 

**두 개의 semaphore의 역할**
thread_status를 표현하는 구조체에는 두 개의 sema가 존재한다
둘 다 부모 스레드를 부르는 `트리거`역할을 한다
**wait_sema**: 자식이 exit할 때 → 부모 호출하여 list에서 빼기 & sema_up
**fork_sema**: 자식이 생성될 때 → 부모 호출하여 list에 넣기 & sema_down