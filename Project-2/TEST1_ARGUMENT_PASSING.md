## Argument passing

### 1차: 커널 모드(파싱 및 메모리 적재)

**가장 기초가 되는 로직: 반드시 구현**
기본 바닐라 상태: `Run didn't produce any output` → 무언가 로직이 구현이 없어 전달 및 형성이 안되고 있음

**혼란과 정리**
**통상적인 OS에서 app 실행 순서**: 유저모드 → syscall → 커널 → 권한하강 → 유저
**Pintos에서 app 실행 순서(現)**: 유저모드(없음), 따라서 syscall(없음) ... 커널 → do_iret() → 유저

**호출 순서 분석**
Argument passing 기준으로 성공적인 실행 흐름 분석:
1. 명령어: `pintos -- ... run 'args-none'`에서 -- 뒤는 커널에 요청하는 것
2. `threads/init.c`실행 → `run_actions(argv)`실행 ... run 'args-none'를 여기서 읽음
3. `process_create_initd(file_name)`실행 → `process_exec(f_name)` ... `load()` 메모리 적재 시작
4. 유저 모드... printf() 명령어 호출 → 시스템 콜: write로 변경 → 커널 진입
5. 커널 → 유저... app이 터미널에 로그를 출력

출력되는 문구는 다음과 같아야 함:
```
(args) begin
(args) argc = 2
(args) argv[0] = 'args-single'
(args) argv[1] = 'onearg'
(args) argv[2] = null
(args) end
args-single: exit(0)
```
<br>

**탐색 및 구현**
시스템 콜은 file 상대, process 상대로 나누어진다.
**file**: read, write, open ..
**process**: exec, wait ..

현 main은 **부트스트래핑(Bootstraping)** 하는 함수이다.
os가 부팅 과정에서 유저의 명령 없이 강제로 기본 app을 띄우는 것.
일반 OS ... init, systemd가 이런 app, Pintos에선 ... 현 args-none이 그 역할.
main 함수와 별개로 터미널에 입력받은 인자는 커널 영역에서 과정을 밟음.
**과정은 대략 다음과 같음**: → process_exec → load → setup_stack
<br>

**세부 경로**
세부 경로를 표현하자면 다음과 같다:

process_exec까지의 경로:
**threads/init.c**에서 main() → run_actions() → run_task() .. *USERPROG가 정의된 상태*
**userprog/process.c**에서 process_wait() → process_create_initd() → → process_exec()

여기서 바닐라 상태인 process_exec()을 수정해야 한다: 문자열 쪼개기 + 메모리 올리기 + 스택 쌓기
한 마디로.. 메모리에 포인터 주소값과 실제 값을 쌓아 넣고, 최종 시작 주소를 반환해야 한다.
process_exec()을 한글과 함께 표현하자면 다음과 같다:
```
process_exec (f_name) {
    1. 변수 선언부
    2. 방어 코드

    3. 기존 페이지 삭제
    process_cleanup ();

    4. 메모리에 적재
    success = load (file_name, &_if);

    5. 변수 메모리 해제
    palloc_free_page (file_name);

    6. 프로세스 전환
    do_iret (&_if);
}
```
load함수의 적재 로직이 부실함을 파악하였다 → 코드를 load 함수에서 보강
<br>

**load 함수 수정하기**
```
load (char 파일명, intr_frame 구조체) {
    1. 변수 선언부

    2. 가상 메모리 지도 생성: ... 나중에 자세히

    !! 여기에 file_name 파싱 코드 추가 !!

    t->pml4 = pml4_create ();

    3. 디스크 접근: ELF 펼치기?
    file = filesys_open (file_name);

    4. ELF 헤더 검증
    5. 헤더를 기준으로 공간 셋팅을 한다: Code영역 & Data영역
    file_ofs = ehdr.e_phoff;
    for (i = 0; i < ehdr.e_phnum; i++) {
        내용이 엄청 많음
    }

    6. 스택 생성: 한 페이지(4KB)를 할당한다
    7. 스택 시작 주소
    if_->rip = ehdr.e_entry;

    !! 여기에 메모리 적제 코드 추가 !! 

    8. 성공여부 반환
}
```
<학습 키포인트>
왜 `명령어를 stack에 저장하는 단계`에서 디스크에 접근을 하는가?
→ 유저모드로 돌아가기 전, 생성한 프로세스에 특정 명령어 헤더(예: ls => 파일 탐색)에 맞는 환경을 셋팅해야 함.
→ 그 환경에 관련 정보는 디스크에 저장되어 있어, 접근해서 불러와야 한다.

_if->rsp는 현 스택의 시작점인 동시에 유저 모드에서도 여기서 읽기를 시작해야 한다.
커널에서 stack에 load하기: -8 바이트 하면서 쓴다.
유저모드에서 읽기: +8 바이트 하면서 읽는다.

커널도 엄밀히 말해 별개의 프로그램이다. 
커널영역에도 stack이 존재하며, 경우에 따라선 할당/해제 로직이 필요할 때도 있다.
<br>

### 2차: 유저 모드

**호출 순서 분석**
커널모드에서 do_iret함수 호출로 유저 모드로 넘어간다.
유저모드에서는 다음 순으로 함수가 호출: main → write → exit
유저모두 호출 함수에 커널이 응답한다: syscall_handler → (sys_write & process_exit) → thread_exit

**주요 함수 분석**
**write**: 화면에 글자를 출력하기 위해 호출, 출력 내용 및 길이 → syscall 함
**syscall_handler**: syscall을 감지하는 함수이다 ... 요청을 감지하고 분류하여 다른 처리기에 전달
**sys_write**: 핸들러를 통해 호출된 함수. validation 이후, 실제 모니터 자원에 접근하여 뿌려줌

추가 궁금점: exit는 서로 독립적으로 시행이 되는가? → NO
먼저 유저모드의 exit 호출 이후 핸들러가 이를 받은 다음 process와 thread의 exit 함수를 호출한다

**유저 프로그램(main 함수) 분석**
```
main (int argc, char *argv[]) 
{
  msg ("begin");
  msg ("argc = %d", argc);
  return 0;
}
```
일부 생략...
msg는 lib에서 호출을 받는, 사실상 write 함수와 동일하게 작용.
return 0 또한 exit으로서 작용.

**주의**: 유저 모드는 별도로 건들 필요가 없다. 이미 syscall등 충분히 호출 로직 구현됨
<br>

### 3차: 커널 모드(핸들러 분기)

**syscall_handler 함수**
개요: 비워져 있음. 추가 구현 필요.
레지스터 인자 필요: 유저모드와 커널모드는 특정 레지스터 값으로 소통을 함.

**레지스터 인자 표 요약**
| 인자 | 역할 |
| :--- | :--- |
| RAX | "어떤 업무(시스템 콜 번호)인가?" |
| RDI | **첫 번째 재료** .. 어떤 파일에 쓸 건지 - fd |
| RSI | **두 번째 재료** .. 어떤 글자를 쓸 건지 - buffer 주소 |
| RDX | **세 번째 재료** .. 몇 글자나 쓸 건지 - size |

<br>

**구현해야 하는 함수들**
**syscall_handler**: 현재 골격만 구성되어 있음, switch 문으로 분기를 만들고 함수를 호출하도록 구성.
**write**: 화면 및 파일에 입력하는 로직, 유효성 검사부터 해서 buffer 보내기. 기본 함수는 이미 있음.
**디폴트 구현**: halt(셧다웃), exit(나가기)

**syscall_hadler 구현 예시**:
```
syscall_handler (struct intr_frame *f UNUSED) {
    switch (f->R.rax)
    {
    각 케이스(SYS_HALT, SYS_WRITE, SYS_EXIT) 별:
    만족하는 함수를 구현하고 호출하면 됨

    default:	
        break;
    }
}
```