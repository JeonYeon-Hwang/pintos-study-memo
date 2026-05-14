## Virtual Memory: Page Table

**메인 함수 분석하기**
```
test_main (void)
{
  char stack_obj[4096] => 2^12 바이트 덩어리
  struct arc4 arc4 => 암호화 도구

  arc4_init (&arc4, "foobar", 6)
  memset (stack_obj를 모두 0으로 채우기)
  arc4_crypt (&arc4, stack_obj, sizeof stack_obj)

  write로 출력하기, cksum은 무결성 검사 로직이라 함
  msg ("cksum: %lu", cksum (stack_obj, sizeof stack_obj))
}
```
main 함수에 인자가 없지만, 여전히 user program이다.
user program에서도 변수 선언 및 할당이 가능하다.
arc4(init & crypt 함수)는 난수 생성용 <u>보조 도구</u>이다.
... 난수화 한 다음 → 해독화 하여 이게 맞는지 확인하는 용도.

*어떻게 수행이 되는가?*
2^12, 즉 1페이지 분량의 스택 확장 변수(char 이름[4096]) 선언,
다만... `변수 선언`은 `레지스터 값`만 바꾸며, 실제 RAM내 칸에 표식하는 행위는 일어나지 않음.

memset → 가상 주소에 2^12만큼 0 쓰기 .. **page fault 발생**!
여기서 수행 로직을 구현해야 한다
<br>

**구현 단계 탐색하기**
예시로 흐름을 이해하자면 다음 아래와 같다:
→ 현재(가상 메모리에서) 스택의 가장 끝 주소는 0x5000
→ memset으로 계속 읽다가 주소 0x4999를 건드림.. 커널은 주소 침범을 인식
→ 여기서 즉시 page fault(예외)를 호출
→ CPU가 사건을 받어서 `page_fault_handler`따위를 실행함(말 그대로 핸들러)
→ `handler`에서 검증 이후, page를 할당하는 분기로 본격 넘어가는 것으로..
<br>

**process.c 분기 파악하기**
먼저 빌드 에러 처리과정을 보자:
```
#ifndef VM
#else
#endif
```
다음과 같은 문구가 박혀있다. 즉 vm할 때는 else 이후의 함수가 실행된다는 것.
만약 이 둘 사이에 동일한 함수명이 중복되어 있어도, 분기에 따른 처리를 하므로 이상 無!
→ *VM용 validate_segment함수를 먼저 구현해야 한다! (일단 복붙)*

memset부터 시작하여.. 어디 단계에서 막히는가?
`memset` → `page fault(exception.c)` → `vm_try_handle_fault(vm.c)`
vm~fault함수에 미완성 구현부가 있음.

vm_try_handle_fault 함수 구현부: 
```
bool vm_try_handle_fault (intr_frame *레지스터 값, void *addr UNUSED,
        bool user UNUSED, bool write UNUSED, bool not_present UNUSED) {
	
    struct supplemental_page_table *spt UNUSED = 현 스레드 spt 가져오기
    struct page *page = NULL;
    /* TODO: Validate the fault */
    /* TODO: Your code goes here */

    return vm_do_claim_page (page) => 페이지에 물리적 프레임을 할당함
}
```

어디서 부터 시작할까? .. 꼬리에서 꼬리를 물기 .. page fault 발생 시
1. vm_get_frame: 프레임 하나 얻기 → 프레임 *반환
2. vm_do_claim_page: 프레임(물리 주소)와 가상 주소 매핑(MMU) → swap in 함
3. vm_try_handle_fault: 여기로 bool 값 반환
4. exception.c에서 성공/실패 여부를 최종 반환

*추가사항*
```
vm_try_handle_fault (struct intr_frame *f, void *addr,
		bool user, bool write, bool not_present)
```
여기에 수 많은 인자값 → 분기를 결정하기 위해서 온 것. 표로 정리하면 다음과 같다:
| 분기명 | 설명 |
| :--- | :---- | 
| not_present | 물리 메모리 부족 → Stack Growth *or* Lazy Loading |
| write | 쓰려고 할 때 나온 Fault |
| user | 문제가 유저 모드일 때 발생하였나? |

그 외에: *addr → 사건 발생 장소 .. 예시: stack 마지막 주소 100에서 99로 초과
<br>

### 구현 단계

**handler에 분기 구현하기(시작은 미미하더라도)**
1 차: validation 필터링

2 차: 분기 처리
.. 1. 만약 장부(SPT)에 페이지가 있다면?
.. 2. 장부에 페이지가 없다면? → stack growth 함수를 구현

*이와중에 알게 된 내용들..*
spt는 hash table인데, key인 addr는 고유한가? 가상주소이면 셋팅되어 겹칠 경우?
.. yes. spt는 각 스레드에 고유하게 존재하기 때문에 addr가 겹치지 않는다.
.. 따라서, 하나의 spt에는 여러 페이지라 해도 addr는 고유해야 한다.
<br>

**stack growth함수 구현**
순서는 다음과 같다...
1. **주소 정렬**
문제 시작점 주소(fault_addr)를 기준으로 페이지 사이즈(PGSIZE)로 주소를 내림
그 내린 (가상) 주소에서 시작하는 페이지를 생성하게 됨
<br>

2. **페이지 생성 & SPT에 등록**
`vm_alloc_page_with_initializer`함수를 호출한다. 해당 함수에서 SPT 등록 구현
.. *함수 `vm~initializer`도 구현해야 한다*:
해당 addr로 시작하는 페이지가 없다면? → 페이지 생성 진행.
페이지 공간 할당(OS메모리) → `uninit_new`함수로 골격을 설정
이후 SPT 등록 함수 `spt_insert_page` 호출
.. *함수 `~insert_page` & 구조체 `SPT`도 구현 필요*:
SPT는 hash table로 구현하기로 ~ 비교적 간단: hash 메서트 import하면 됨.
사용하기 위해 process_exec에서 `spt_init`을 해야 한다.
.. *`spt_init`위해 또 추가 구현을 해야 한다..*
`page_hash(*e, *aux)` → *e를 통해서 hash_int를 반환
`page_less(*a, *b, *aux)` → 비교함수, 두 page의 addr로 비교
<br>

3. **프레임 할당**
`vm_claim_page`호출 
→ 여기서 `vm_get_frame`으로 물리 메모리 가져오기
→ 이후 `do_claim`으로 물리-가상 주소 매핑하기(MMU)
.. *함수 `vm_claim_page` 구현하기*
spt에서 이전에 등록된 페이지를 찾고 넘겨주면 끝!
.. *함수 `vm_get_frame` 구현하기*
여기서 frame은 램에 할당하는 것을 의미한다.
frame 구조체 메모리 할당 → 內 kva에 palloc 시행.
(추가사항) frame_table에 kva 요소를 넣는다: 왜? 아직 모름.
.. *함수 `vm_do_claim_page` 구현하기*
MMU 수행 함수는 내장되어 있음. `install_page`가져와서 매핑 수행.
<br>

4. **swap_in 수행**
swap_in은 "디스크 상 가져오기"만 의미하는 것이 아니다. 내용물 채우기도 함.
swap_in 메크로가 가는 곳은 다음과 같다: .. `uninit_initialize`로 이동
.. *`uninit_initialize`의 역할?*
초기화 및 가동 시도를 하는 함수라고 한다. 여기서 type을 정하고 분기를 시작
...
~ 내용이 너무 난해하다. 가능하면 스킵 ~
...
→ `page_initializer`로 넘어감 
이전에 `uninit_new`에서 설정한 ANON 타입 기준, `anon_initializer`로 이동
.. *anon_swap_ 구현하기?*
일단 pt-* 테스트에서는 swaping이 수행될 필요가 없으므로? 스킵하기로 결정.
<br>


**setup_stack함수 구현**
왜 구현해야 하는가? → vm에선 생성초기부터 스택에 spt에 넣어 페이지 생성함

*표로 정리해보자*
| 방식명 | 설명 | 
| :--- | :--- |
| user_prog | 단순무식 <br> palloc_get_page로 하나 받고, install_page에 바로 꽂는다 |
| vm | 선 장부, 후 할당 <br> vm_alloc_page을 하여 SPT에 꽂기, 이후 바로 claim_page |

<br>

**lazy_load_segment함수 구현**
왜 구현해야 하는가? → page fault시 반드시 거쳐가야 하는 함수이기 때문.
경로: `page_fault` → 핸들러 → page요청 → `uninit_initialize` → `lazy_load_segment`

*두 함수 간의 관계를 이해해보자*
| 함수명 | 설명 | 
| :--- | :--- |
| load_segment | spt에 기록만 한다, 페이지는 VM_UNINIT상태 |
| lazy_load_segment | page_fault시, load_segment에서 설정한 방침대로 램에 복제 |

*이와중에 알게 된 내용들..*
lazy_load_segment는 단순히 page fault후 디스크에서 페이지를 가져오는 역할에 한정안됨
초기, 프로그램이 실행될 때 "실행파일"을 램에 적재하는 데에도 사용이 됨!
<br>

### 트러블 슈팅
**현재 오류 상태**
```
read_bytes -= page_read_bytes;
zero_bytes -= page_zero_bytes;
upage += PGSIZE;
```
load_segment()의 ofs 미증가가 문제라고 함 → upage만 PGSIZE만 오름.
ofs도 증가해야 하는데.. 그럼 ofs는 무엇? → 파일 안에서 몇 번째 binary 읽을 지.
ofs 또한 각 page.. segment의 시작점을 가리켜야 한다.

이전 오류 출력 문구는 다음과 같다:

<small>Acceptable output:
  (pt-grow-stack) begin
  (pt-grow-stack) cksum: 3424492700
  (pt-grow-stack) end
Differences in `diff -u' format:
마 (pt-grow-stack) begin
마 (pt-grow-stack) cksum: 3424492700
마 (pt-grow-stack) end
뿔 pt-grow-stack: dying due to interrupt 0x06 (#UD Invalid Opcode Exception).
뿔 Interrupt 0x06 (#UD Invalid Opcode Exception) at rip=40103b
뿔  cr2=00000000000000ff error=               0
뿔 rax 00000000000000ff rbx 0000800424305000 rcx 00000000000000ff rdx 0000000000406dc0
뿔 rsp 000000004747ff78 rbp 000000004747ff90 rsi 0000000000000400 rdi 0000000000000000
뿔 rip 000000000040103b r8 0000000000401323  r9 00008004243f5000 r10 0000010424302800
뿔 r11 00008004243ef000 r12 0000800422bcd800 r13 00008004243f1000 r14 0000800420a1b100
뿔 r15 00008004243f0000 rflags 00000202
뿔 es: 001b ds: 001b cs: 0023 ss: 001b</small>

*추후 분석하기!*