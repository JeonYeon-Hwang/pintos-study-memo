## System call basic

### 1차: create

**create 테스트 통과 위해선?**
추정: syscall_handler와 추가 함수를 구현해야 한다.
구현 수준: `filesys.h`와 `file.h`에 구현된 함수를 끌어다 작성을 하는 것에 초점.
<br>

**로직 흐름 전반 파악하기**
create main 함수를 바탕으로 흐름 파악:
1. 먼저 이전 처럼 argument passing이 일어난다.
   - OS 부팅 시, 커널에서 `유저 영역의 스택`에 강제로 인자를 주입해야 함
   - 이전 테스트: 프로그램 명 `args-none`을 박아줌, 현 테스트: `create-*`박음.
2. 만들어진 유저 프로세스에서 `유저가 create을 요청`한다
   - 유저 함수에 create가 존재하며, 이는 syscall_handler에 접수가 되어야 함.
3. filesys_create 함수를 호출된다
   - 여기서 직접 파일을 읽기/쓰기 함수를 구현할 필요는 없음.
   - 실제 테스트에서 파일이 생성되고 물리적인 블록도 할당이 됨.
   - 다만! create인 만큼, 할당된 공간에 무언가를 쓰는 것은 아님.
<br>

**추가로 학습한 키 포인트**
유저 모드와 커널 모드 정확히 어떻게 굴러가는 것이고, 메모리에는 어떻게 적재되는 것일까?
→ (Pintos 기준) 프로세스 하나에는 스레드 하나가 존재한다.
→ 스레드에는 유저 스택 & 커널 스택이 별개로 존재한다.

CPU에는 두 가지 권한이 존재한다:
유저 모드(Ring 3): 유저 영역의 스택만 건드릴 수 있다.
커널 모드(Ring 0): 유저 영역과 커널 영역을 모두 건드릴 수 있다.

프로세스에는 User Territory, Kernel Territory가 나눠져 존재한다.
유저 모드일 때 건드리는 영역: *Heap(malloc 등으로 할당), User Stack, Data(전역 변수 공간) ..*
커널 모드일 때 건드리는 영역: *Kernal Stack, Thread Structure(프로세스, 스레드 상태 구조 정보 담은 곳) ..*
<br>

**구현&수정한 함수들**
syscall.c에 create() 추가 구현 및 syscall_hadler에 해당 분기만 추가하면 끝.

왜 비교적 간단할까? 
→ 이미 차려진 밥상: 반환값까지 셋팅된 filesys_create를 호출하면 알아서 공간을 생성
→ 생성은 state가 없다: 말 그대로 file을 만들고 공간을 할당하는 것으로 로직 종료

*state란? ... 예를 들어 open 할 경우, 데이터가 수송될 fd 등을 계속 저장해야 함*
<br>


### 2차: Open

**Open에 대해 이해하기**
fd(file description)이라는 int를 창구로 삼아서 소통하는 것을 이해할 필요있음.
open에 성공할 시, fd(0, 1, 2을 제외한)를 반환한다.
fd → 파일과 소통하는 연결망 객체를 가리키는 주소값.. 정도로 생각.

**create 함수와 다른 점**
create syscall 함수와 마찬가지로, filesys.h에 정의된 함수를 사용하면 된다.
그러나 발급된 fd가 반환되며, 이는 thread에 list 형태로 저장을 해 두어야 함.
→ 저장해둔 fd는 `소통창구`로서 추후 read/write등에서 사용된다.
<br>
