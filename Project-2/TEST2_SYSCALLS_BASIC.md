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