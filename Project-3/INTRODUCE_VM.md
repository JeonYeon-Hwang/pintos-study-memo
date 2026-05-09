## Virtual Memory: 개요

### Introduce
**가상 메모리란 무엇인가?**
기존 지식: 주 메모리 공간이 부족하여, 파일(보조 메모리)를 가상으로 사용하여 스와핑한다.
실제 이번 pintos에서도 `스와핑`을 구현 ... 다만, 밑바닥인 페이지 구조부터 출발!
<br>

**kaist.github.io를 통해 대략적인 이해 & 정리**:
톱 다운? 방식의 학습을 통해 대략적인 감을 잡는 것이 목표..
페이지 안에는 프로세스 등이 존재한다.

1. 메모리 관리
각 파일 위치와 이름을 통해 기본적인 구조를 파악하자
*vm.h* & *vm.c*:
`struct page` 페이지 구조체 기본 구성 .. 內 union 구조체(Type 하나만 가능)
`struct page_operations` swap in/out, destroy을 집행하는 메크로와 연관된 듯
...
**supplemental page table** 도입하기 .. pml4페이지 할당: 추가 데이터 주입 필요
함수: `supple~init`(초기화), `spt_find~`(spt에서 찾기), `spt_insert~`(spt에 삽입)
...
**Frame** 이란? .. 물리 메모리 진짜 공간, 정확히는 접근 가능한 구조체로 보임.
일종의 페이지를 나타내는 `틀`로 보이는데, MMU(물리주소와 매핑)가 개입하는 것으로..
함수: `~get_frame`(user pool → 획득), `do_claim_page`(MMU 씀), `claim_page`(할당)

<br>

2. 무명의 페이지
페이지의 생애 주기: `page fault` → `lazy load` → `swap in` → `swap out` 정보 획득
page in/out과 swap in/out 차이: swap은 프로세스 단위, page는 더 작은 페이지 단위
... 프로세스 조차 page로 작게 단위를 쪼개서 '부분적인' 교체가 가능하다.
... in: 주 메모리로 페이지가 들어가는 것, out: 반대로 나오는 것
...
*lazy loading란 무엇일까?*
프로그램이 돌 때: 모두 메모리에 올리지 않고, 그때 그때 필요한 것만 올리는 방식(개념)
페이지 타입(VM_TYPE)은 3종류, 타입에 따라 절차가 다소 차이남
순서: `커널(새 페이지 요청 받음)` → `vm_al~initializer` → `생애 주기` → `destroy` 순
...
**Supplemental Page Table(SPT)**
fb_table이 기억나는가? ... 페이지를 관리하는 일종의 **장부**이다.
page fault가 발생할 때: 커널이 이 SPT 장부를 살펴보며, hash_table 형태로 구현됨.
...
페이지 종류: `UNINIT`, `ANON`, `FILE`

<br>

3. 스택 늘리기
(이전 플젝의) 프로세스의 스택영역은 정해져 있으나, 꽉 차면 페이지 추가 할당을 함.
스택 증가 기능을 구현하려면 다음과 같은 함수가 수정 및 구현되어야 함:
`vm_try_handle_fault`: 페이지 폴트 발생 이후 → 스택 확장으로 인한 것인지 판별함
`vm_stack_growth`: 판별(참) 이후 호출, addr값(한계값)을 PGSIZE로 내림 

<br>

4. 메모리에 매핑된 파일들
두 개의 함수를 이해하는 것이 중요하다: `mmap`, 그리고 `munmap`
...
**mmap 함수**
물리 메모리는 실제로는 순서가 엉망진창 뒤바뀌어 있을 수 있다. 
한 프로세스는 여러 페이지로 쪼개져 있고, 그 페이지의 실제 메모리의 순서는 보장 못함.
엉망인 물리 메모리 =(mapping)=> 예쁜 순차적인 가상 주소(ex. 0, 1..)로 "환상"을 만듦
...
**munmap함수**
mapping을 해제하는 함수이다. 
해당 영역의 물리 메모리들은 가상 페이지 list에서 삭제되며, file 상태로 되돌아온다.
...
*baked page는 무슨 뜻일까?* ... backed: 뒤를 받쳐주고 있다. 의미
Anonymous Page: 아무 것도 없는? 상태의 빈 페이지
Baked Page: 파일의 내용이 그대로 복사되어 들어있는 페이지
일반적으로 "프로세스"용 내용이 아닌, file 저장소 자체의 내용이 "복사되어" 들어감
why? → CPU는 램과만 소통, baked 페이지로 빠르게 작업 可 .. 이후 disk에 반영

<br>

5. swap in & swap out

<br>
