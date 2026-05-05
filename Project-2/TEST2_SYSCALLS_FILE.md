## System call: File sys

### 분기 설정

**syscall 구현을 File Op와 Process Ctrl로 나눔**
표로 정리하자면 다음과 같다:
| 분류 | 항목 | 한 줄 요약 |
| :--- | :--- | :--- |
| **File Operation** | create | 파일 생성 → 공간 할당 |
| | open | 파일 열기 & fd 번호 부여 |
| | read | fd를 통해 버퍼로 읽어오기 | 
| | write | 버퍼 데이터를 fd로 특정 파일에 쓰기 | 
| | close | 연결 통로를 닫고, fd도 fd_table에서 제거 | 
| | rox | 실행 중일 파일 쓰기 시도 → Block하기 |
| | bad | 잘못된 fd → 커널이 예외처리 |
| | dup2 | fd복제: 동일 파일 접근하기 |
| **Process Control** | fork | 자식 프로레스 복제 |
| | exec | 현 프로세스 이미지(?)를 교체 |
| | wait | 자식 프로세스 종료 → 부모 프로세스가 status 회수 |
| | args | 명령줄 인자를 파싱하여 넘겨주기 |
| **기타** | other | 시스템 전반 동작 |
| | no vm | project3의 가상메모리 없는 상태에서 동작 테스트 |

<br>

### 3차: close

**의문**
현재 close 관련 아무런 구현이 되지 않은 상황에서 모든 case가 pass 됨.
why? → 테스트 코드 살펴보기: `CHECK ((handle = open ("sample.txt")) > 1 ..)`
탐색결과: 반환값이 1 초과이면 자동으로 PASS 처리한다는 것이 한계.
하지만 실제 닫히지도, 메모리가 해제되지도 않았음.

**fd_table을 활용하라**
수정한 open 함수:
```
open (파일 주소값){
	struct file *f = filesys_open(파일 주소값);
	struct file **fd_table = 현 스레드->fd_table;

	단순 순회 => 순차적으로 fd 값 넣기
	for(int i = 2; i < 512; i++){
		if(fd_table[i] == NULL){
			fd_table[i] = f;
			return i;
		}
	}

	return -1;
}
```
<br>

구현한 close 함수:
```
void close(fd 값){
	struct file *f = 현 스레드->fd_table[fd];

	찾았을 경우 => 로직을 다음과 같이 NULL 처리
	if(f != NULL){
		file_close(f);
		curr->fd_table[fd] = NULL;
	}
}
```
<br>

트러블 슈팅:
palloc_get_page() → 단순 페이지 공간 할당 NO, Lock도 함께 딸려옴.
<br>


### 4차: Read

**Read의 흐름 이해하기**
→ 현재 이미 디스크에는 특정 문자열(or 데이터)이 박혀 있다.
→ 마찬가지로, 커널에서 프로세스(유저 및 커널 영역)을 셋팅한다.
→ 유저 모드 전환, open 실행 후 특정 fd를 기준으로 읽기 syscall 요청.
→ 커널은 이 요청을 받고 실행한 다음 읽은 size를 반환.
