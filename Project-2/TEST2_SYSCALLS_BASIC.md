## System call basic

### 1차: create

**create 테스트 통과 위해선?**
추정: syscall_handler와 추가 함수를 구현해야 한다.
구현 수준: `filesys.h`와 `file.h`에 구현된 함수를 끌어다 작성을 하는 것에 초점.

**로직 흐름 전반 파악하기**
create main 함수를 바탕으로 흐름 파악:
1. 먼저 이전 처럼 argument passing이 일어난다.
   - OS 부팅 시, 커널에서 `유저 영역의 스택`에 강제로 인자를 주입해야 함
   - 이전 테스트: 프로그램 명 `args-none`을 박아줌, 현 테스트: `create-*`박음.
2. 커널에서 만든 유저

<br>

**추가로 학습한 키 포인트**
유저 모드와 커널 모드 정확히 어떻게 굴러가는 것이고, 메모리에는 어떻게 적재되는 것일까?
→ (Pintos 기준) 프로세스 하나에는 스레드 하나가 존재한다.
→ 스레드에는 유저 스택 & 커널 스택이 별개로 존재한다.

CPU에는 두 가지 권한이 존재한다:
유저 모드(Ring 3): 유저 영역의 스택만 건드릴 수 있다.
커널 모드(Ring 0): 유저 영역과 커널 영역을 모두 건드릴 수 있다.

프로세스에는 User Territory, Kernel Territory가 나눠져 존재한다.
**유저 모드일 때 건드리는 영역**: Heap(malloc 등으로 할당), User Stack, Data(전역 변수 공간) ..
**커널 모드일 때 건드리는 영역**: Kernal Stack, Thread Structure(프로세스, 스레드 상태 구조 정보 담은 곳) ..