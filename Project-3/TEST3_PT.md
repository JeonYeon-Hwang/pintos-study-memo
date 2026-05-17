## Virtual Memory: Page

**테스트 목적 분석하기**
앞의 Pt 시나리오는 stack_growth 기준 확인, writable 체크, page fault 판별 등을 다룬다
본 테스트는 사뭇 다른데, 이는 소규모 두 가지로 나눠진다. 표로 정리하면 다음과 같다:
| 테스트 명 | 설명 |
| :--- | :--- |
| pt-* | 기본 VM 동작 검사: lazy_loading, page_fault, 랜덤 읽기/쓰기, 병렬 접근 |
| pt-merge-* | fork 이후의 검사: 온전한 child에 전달, parent-child 독립성 등 |

현재 `pt-*`는 통과가 된 상태로, `pt-merge-*`를 중심으로 살펴보기로..
<br>

### page-merge-seq

**test_main 함수 분석하기**
```
test_main (void)
{
  init () => 암호화 이후, histogram[256]으로 정규 분포도를 그린다
  sort_chunks () => 병합정렬 1단계: 조각단위로 정렬한다
  merge () => 병합정렬 2단계: 각 조각을 병합한다
  verify () => 검증을 수행할 듯
}
```
*테스트는 어떻게 돌아가는가? ..* 예시를 보자:
- 정렬 전 원본 (buf1): [3, 3, 5, 3]
  histogram 결과: 3은 3개, 5는 1개
<br>
- 자식에서 정렬된 결과: [3, 3, 3, 5]
  histogram 결과: 3은 3개, 5는 1개

이렇게 두 가지의 histogram을 비교한 다음 일치 여부를 체크하는 것으로 이해.
<br>

**오류 분석 및 수정하기**
```
bad value 18 in byte 0
```
→ histogram의 0번째 바이트 부터 일치하지 않는다는 것(처음부터 무언가 잘못됨)

*현재 테스트 시나리오 흐름을 설명하자면..*
1. 부모가 랜덤 데이터를 만듦 → histogram에 이를 적음(각 byte 값의 반복 횟수 분포도)
2. 이를 file에 chunk 단위로 저장한다
3. 자식이 chunk 단위로 읽음 → 정렬을 한 다음 → 다시 file에 write한다.
4. 부모 file의 chuck를 merge함 → 부모 램에 올림 .. 요거 이해가 안 감: 일단 패스
5. 올려진 결과와 원본 histogram과 비교를 통해 verify를 한다

*오류 위치를 추정하자면..*
fork/exec/read/write 이후 파일 또는 메모리 데이터가 깨진 상태, 추적 및 도움 받은 결과:
.. 대규모 데이터(14 page 규모)가 buffer에 할당에 제대로 안되는 상태로 파악.
.. 이는 read(file → buffer:page)와 write(buffer:page → file)하는데 페이지 연장 필요.

*해결 방식은..*
syscall의 read & write에 stack_growth와 vm_claim_page 경우의 수를 추가.
기존에 놓쳤던 file을 seek하는 syscall 로직 추가.
<br>

### page-merge-*
세 테스트 모두 `parallel_merge`함수를 호출하여 시나리오를 진행한다.
세 시나리오는 기본적으로 동일한 부분이 있으며 이를 설명하면 다음과 같다:
1. 부모 프로세스에서 buf에 대량의 데이터(난수)를 생성
2. 이 데이터를 파일로 디스크에 write, 이후 wait
3. 복수의 자식 프로세스들은 균등하게 나눠서 파일을 read하여 가져옴
4. 자식 프로세스들은 램에서 이들을 연산을 하여 정렬을 함
5. 이를 부모 프로세스가 가져와서 merge하고 검증을 함

*여분의 사실들..*
검증은 오름차순 정렬 여부 & histogram 데이터 일치 여부 둘을 검증한다.
각 자식이 받아온 데이터는 서로 최소-최대가 겹치지 않는다.
<br>

**3가지 테스트 케이스 분석**
세 시나리오는 위의 시나리오를 공유하지만 다음과 같은 상세 내역에서 차이가 난다:
| 테스트 명 | 해설 | 요구사항 |
| :--- | :--- | :--- |
| page-merge-par | 파일 읽기/쓰기 | 시스템이 동기화없이 독립적 운영 여부 |
| page-merge-stk | 스택 메모리 | 스택 확장 로직 정상적 운영 여부 |
| page-merge-mm | mmap()사용  | read/write콜이 아닌, 가상 메모리 운영 여부 |

<br>

**page-merge-par**
`filesys_open()`함수를 기준으로 디버깅을 수행하였다:
```
filesys_open (const char *name) {
  printf("다음 파일을 열음: %s \n", name);
  .. 생략 ..
  return file_open (inode);
}
```
이를 통해 일부 오류를 발견한 부분(일부)은 다음과 같다:
```
(page-merge-par) sort chunk 5
다음 파일을 열음: buf5 
다음 파일을 열음: buf4 
다음 파일을 열음: child-sort 
child-sort: exit(123)
```
정해진 파일이 단일하게.. 가 아닌, 난잡한 순서로 병렬로 출력됨.
이는 lock이 제대로 수행이 되지 않아서 생기는 문제라고 함(자세히는 나중에 학습)

*해결한 접근 방식*
→ `load()`에 `filesys_open()` 호출 전후에 filesys_lock 공유/제약 추가
→ spt에 접근하는 주요 로직에는 별도의 spt_lock을 만들어 공유/제약을 추가함
.. 신가하게, page-merge-stk도 pass함. 아마 동일한 동시성 문제일 듯.
<br>

**page-merge-mm**