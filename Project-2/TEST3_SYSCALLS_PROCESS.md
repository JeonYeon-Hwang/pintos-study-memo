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