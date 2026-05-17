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