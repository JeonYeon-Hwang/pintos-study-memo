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
| **Process Control** | fork | 자식 프로세스 복제 |
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
→ 현재 이미 디스크에는 read할 특정 문자열(or 데이터)이 박혀 있다.
→ 마찬가지로, 커널에서 프로세스(유저 및 커널 영역)을 셋팅한다.
→ (유저모드) open syscall 요청.
→ (유저모드) filesize syscall 요청.
→ (유저모드) read syscall 요청.
→ 커널은 이 요청을 받고 실행한 다음 읽은 size를 반환.

filesys.c/file.c의 일부 함수들은 파일 명으로 찾고 시작 주소값을 반환한다.
*우리는 이것을 fd_table에 index 형태로 bind한다.*
<br>

**file_read(file.c)함수 분석**
```
file_read (파일 시작 주소, 유저 영역 buffer, 요청 한계 size) {
	읽은 바이트 수 = inode_read_at (file->inode, buffer, size, file->pos);
	file->pos += 읽은 바이트 수;
	return 읽은 바이트 수;
}
```
`file->pos`: 현재까지 읽은 파일 포인터 위치
`off_t`: 바이트 수를 표현하는 정수 타입(2GB 초과해 int:4byte 초과하는 문제 방지)
`결과값`: off_t를 반환하는 것 같지만... 알아서 에러 관련(0, -1)도 함께 반환
<br>

**함구 구상 및 작성**
다음과 같이 구상을 할 수 있다(한글로 표현):
```
read (fd, buffer, size) {
	struct file *f = fd로 파일 주소값 찾기;

	if (fd가 0일 경우: 키보드 입력) {
		while (size 보다 적게) {
			buf[type_size] = input_getc ();
			if (엔터가 입력된다면?)
				break;
			type_size++;
		}
		return type_size;
	} else {
		off_t s = file_read (f, buffer, size);
		return s;
	}
}
```
<br>

### 5차: ROX

**ROX란 무엇인가?**
Read Only Executable의 약어로, 


