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