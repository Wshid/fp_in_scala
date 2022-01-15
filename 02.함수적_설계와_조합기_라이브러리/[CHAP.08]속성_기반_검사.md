# [CHAP.08] 속성 기반 검사
- 속성 기반 검사(property-based testing)
  - 간단하지만 강력한 라이브러리
  - 프로그램 **행동 방식**의 서술과
    - 그런 행동 방식을 검사하는 **검례**(test case)의 생성을 분리
- 프로그래머는
  - 프로그램 행동 방식을 명시하고
  - 검례들에 대한 **고수준 제약**을 제시하는 데에만 집중
- 그러면 **프레임워크**가
  - 그런 제약을 만족하는 **검례**들을 자동 생성
  - 그 검례를 실행해서 **프로그램이 명시된 대로 작동하는지 확인**

## 8.1. 속성 기반 검사의 간략한 소개
- 속성 기반 검사 라이브러리(`ScalaCheck`)의 `property`는 다음과 같은 형태

#### CODE.8.1. ScalaCheck의 속성
```scala
// 0 ~ 100 사이의 정수 목록을 생성하는 생성기
val intList = Gen.listOf(Gen.choose(0, 100))

// List.reverse 메서드와 행동 방식을 명시하는 속성
val prop =
  // 목록을 두 번 뒤집으면, 다시 원래의 목록이 되는지 검사
  forAll(intList)(ns => ns.reverse.reverse == ns) &&
  // 목록을 뒤집은 후 첫 요소가 마지막 요소가 되는지 검증
  forAll(intList)(ns => ns.headOption == ns.reverse.lastOption)

// 명백히 거짓인 속성
val failingProp = forAll(intList)(ns => ns.reverse = ns)
```
- `intList: Gen[List[Int]]`형태
  - `List[Int]`가 아님
  - `List[Int]` 형식의 **검사 자료**를 생성하는 방법을 아는 어떤 것
- 이 `generator`는 `sample`을 제공
- **속성 기반 검사 라이브러리**의 생성기들을
  - 여러 가지 방법으로 `조합, 합성, 재사용` 가능
- `forAll`는
  - `Gen[A]`의 형식의 생성기와,
  - `A => Boolean`형식의 `predicate`를 조합하여 하나의 **속성**을 만들어냄
- 이 속성은
  - 생성기가 생성한, 모든 값의 그 술어을 만족해야 함을 단언(`assert`)함
- 생성기처럼 **속성**에도 다채로운 API를 지원
- `prop.check`를 호출하면
  - `ScalaCheck`는 무작위로 `List[Int]`를 검사
  - 주어진 `predicate`를 반증하는 사례가 있는지 검증
- `failingProp.check`를 통해
  - 일부 입력에 대해 거짓으로 평가되는 것을 확인

### 속성 검사 기반 라이브러리의 기능

#### 검례 최소화(test case minimization)
- 검사가 실패하면, 프레임워크는 검사에 실패하는 **가장 작은 검례**에 도달할 때까지
  - 더 작은 검례를 시도
- e.g. 크기가 10인 목록에 한번 실패했다면,
  - 그 검사에 실패하는 **가장 작은 목록**에 도달할때까지 검사를 수행
  - 찾아낸 최소의 목록을 보고
  - 최소의 검례는 **디버깅에 도움을 줌**

#### 전수 검례 생성(exhausitive test case generation)
- `Gen[A]`가 생성할 수 있는 모든 값의 집합을 **정의역**(domain)이라고 함
- 정의역이 충분히 작다면,
  - 표본 값들을 생성하는 대신, **정의역의 모든 값을 검사**할 수 있음
- 만약 **정의역**의 모든 값에 대해 속성이 성립한다면
  - 반례가 없음을 확인한 차원 외에
  - 속성이 **실제로 증명**된 것