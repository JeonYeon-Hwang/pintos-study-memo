## Virtual Memory: Page Table

**PT나머지 케이스 분석하기**
현재 기본적인 구현 및 테스트 케이스 일부는 통과 상태.
나머지가 무엇이 문제인지, 함수 분석과 부족한 지식을 재귀적?으로 학습 병행할 것.

### pt-grow-bad
이름만 봐서는 스택 grow가 잘못된 방향으로 가는 것을 의미하는 듯. 함수를 보자:
```
asm volatile ("movq -4096(%rsp), %rax");
```
assembly로 해석 시, "레지스터 `%rsp`에 있는 값을 `%rax`로 옮겨라."를 의미.
mov + q에서 q는 8바이트 단위로 처리하겠다는 뜻.
`%rsp`는 Stack Pointer 레지스터로, 스택의 가장 위(최신 값)을 의미.
`%rax`는 Accumulator 레지스터로, 함수의 리턴값 or 산술 연산용으로 쓰임.
→ "스택의 꼭데기 주소를 `%rax`에 저장하라"를 의미. 

*앞에 -4096은 무슨 의미?*
현 `%rsp`에서 4096byte 뺀 곳의 (주소에 위치한) 데이터를 의미한다.
일반적으로 4096은 한 페이지이며, 한 페이지 앞으로 간 값을 `%rax`로 옮겨라를 의미.

*그렇다면, 왜 에러가 나야 할까?*
스택 성장은 현재 사용자 rsp를 기준으로 판단한다.
fault addr가 현재 rsp 근처라면, 정상적인 스택 성장으로 보지만,
rsp는 그대로인데 -4096(%rsp)처럼 멀리 아래 주소로 접근하면, 범위 벗어남으로 에러!
현 assembly 명령어: CR2 = rsp - 4096 .. addr(CR2)에는 rsp와 큰 차이나는 값.

*구연해야 할 부분들..*
1. `vm_try_handle_fault`함수에 추가 false 반환부를 추가해야 한다.
넣는 위치: spt_find_page 이후 스택 확장 이전에.. 이유는 잘 모름, 추가 학습 필요.
    ```
    if(!addr_range_check(f, addr)) return false;
    ```

2. exception.c에서 유저 모드일 때 exit(-1)을 호출해야 한다.
현재는 false 발생 시, kill(f)만 호출, 구현은 다음과 같이:
    ```
    if(user)
        exit(-1);
    else
        kill (f);
    ```
<br>

### pt-write-code
어떠한 테스트일까? → 원래는 write가 안되야 할 부분에 write을 막아야 하는 것. 함수를 보자:
```
*(int *) test_main = 0;
fail ("writing the code segment succeeded");
```
여기서 `*(int *) test_main`부분은 코드 영역, 프로그램 설계도를 작성한 곳이다.
이 곳은 **read only**가 되어야 하는 곳으로, 쓰기가 막혀야 함!

결정적 증거는 다음 출력 로그를 보면 알 수 있다: `Exception: 0 page faults`
쓰지 말아야 할 곳에 쓰게 되면 page_fault가 발생하게 된다. 
**페이지 폴트는 "페이지 부재"만 한정하지 않는다**가 핵심: 통상적 처리가 불가함을 알리는 것.

*문제 접근 및 해결하기*
하나의 프로세스에는 영역별로 페이지를 쪼개서 관리하고, 쪼개는 단위에는 각 권한이 부여됨.
예를 들어, 코드 영역은 read only이므로 해당 페이지는 별도로 나눠서 권한을 설정하는 식.
이러한 권한 정보는 segment table에 저장이 되어 있다.
→ struct page에 writable 변수를 추가하여 표식을 하자.

*경로 중 어디에서 감지하여 exit(-1)를 호출하게 해야 하나?*
현재는 `vm_try_handle_fault`까지 도달하지 못한 상황: page fault가 발생 안했으므로.
앞의 `lazy_load_segment`에서 램에 적재할 때, writable을 제대로 입력하는지 확인필요.
→ writable 변수 추가 및 page 생성 시 넣도록 구현하니 해결 완료!
<br>

### pt-write-code2
테스트 분석: 스크립트를 살펴보고 분석해보자.
```
int handle;

CHECK ((handle = open ("sample.txt")) > 1, "open \"sample.txt\"");
read (handle, (void *) test_main, 1);
fail ("survived reading data into code segment");
```
CHECK: `sampe.txt`파일을 열고 fd값을 `handle`에 넣는다. fd가 2이상이어야 함.
read: 해당 fd로 파일을 "1 size 만큼" 읽고 이를 `test_main`에 집어넣는다.
문제 → `test_main`은 실행코드 본체(코드 영역)이므로 page fault가 발생해야 함!

*문제 접근 및 해결하기*
현재 output상 page fault가 발생 안함 → 어디서 발생해야 하나? 경로 찾기
`syscall_handler` → `file_read` → `memcpy`(여기서 터짐) → `page_fault`

앞의 문제와 차이 비교하기:
| `*(int *) test_main = 0` | `read (handle, (void *) test_main, 1)` | 
| :--- | :--- |
| CPU가 직접 MMU를 만남 | syscall로 커널 모드 바로 진입 |
| MMU측에서 page의 writable권한 캐치함 | 커널 내부의 함수가 검수 없이 통과 |

따라서.. 이번 문제는 syscall을 건드려야 한다!

작성 및 통과한 함수(syscall.c) 기술내용:
```
check_writable (*buffer, size){
    struct supplemental_page_table *spt = spt 가져오기;
    char *ptr = pg_round_down하여 시작주소 찾기
    char *end = +size 하여 끝 점 찾기

    writable한지 권한 알아보기: 순회하면서
    while(ptr < end){
        struct page *page = spt로 page 찾기
        if => page가 없거나 or writable 권한이 없으면 exit(-1)
        ptr += 페이지 사이즈 만큼 더하기
    }
}
```
<br>

### pt-grow-stk-sc

**테스트 개요 및 분석**
write file한 다음(1) → read를 수행(2)하는 로직으로 보임.
대부분의 로직은 정상적으로 출력되고 있으나, 다음 줄을 기준으로 exit(-1):
```
CHECK (read (handle, buf2 + 32768, slen) == slen, "출력문구");
CHECK (!memcmp (sample, buf2 + 32768, slen), "출력문구");
close (handle);
```
memcmp란? → 바이트 단위로 상호 비교를 함. 문자열 길이와 일치한 길이 비교.

*명령어`buf2 + 32768`를 보자*
한 페이지(4KB)이상 건너뛴 (32KB)의 "일기" 명령이다: page fault가 발생해야 함.
현재는 exit(-1)을 호출하고 있는 상황이다.

**접근 및 개선하기**
현재 문제의 코드 부분은 다음과 같다:
```
check_writable (const void *buffer, unsigned size){
    ..생략..

    while(ptr < end){
        struct page *page = spt에서 페이지 찾기(ptr);
        if(page가 없다면?){
            if(유저 영역 내 ptr라면?)
                vm_stack_growth(ptr);
            else
                exit(-1);
        }
        else if(읽기 권한이 없다면){
            exit(-1);
        }

        ptr += PGSIZE;
    }
}
```
페이지가 없을 경우, 무조건 exit(-1)을 호출한다. 
그렇게 하기 보다는... syscall.c에서 stack_growth가 이뤄질 수 있도록 호출하면 됨.
해당 로직에서 while 문을 통해, 32KB까지 페이지가 계속 grow 했음.