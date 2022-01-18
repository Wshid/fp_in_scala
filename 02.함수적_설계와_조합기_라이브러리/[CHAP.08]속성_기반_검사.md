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

## 8.2. 자료 형식과 함수의 선택
- **속성 기반 검사**를 위한 라이브러리 설계

### 8.2.1. API의 초기 버전
- 검사 라이브러리에 사용할 자료 형식?
- 어떤 기본 수단을 사용할지 및 어떤 의미를 부여할지?
- 그 함수들이 만족해야할 법칙은 어떤것일지
- ScalaCheck 예시 살펴보기
  ```scala
  val intList = Gen.listOf(Gen.choose(0, 100))
  val prop =
    forAll(intList)(ns => ns.reverse.reverse == ns) &&
    forAll(intList)(ns => ns.headOption == ns.reverse.lastOption)
  ```
- `Gen.choose`나 `Gen.listOf` 구현에 대해 아무것도 모르더라도,
  - 이들이 돌려주는 **자료 형식**은 반드시
  - 어떤 형식을 매개변수로 사용해야 함을 짐작할 수 있음
- `Gen.choose(0, 100)`은 `Gen[Int]`를 돌려주어야 하며
  - `Gen.listOf`는 서명이 `Gen[Int] => Gen[List[Int]]`인 함수를 의미
- `Gen.listOf`가 입력으로 주어진 `Gen` 형식에 구애받을 필요가 없을 것이므로
  - (`Int, Double, String`등의 목록별 개별 조합기가 필요 없다는 의미)
- 다음과 같이 다형적인 함수로 만드는 것이 합당함
  ```scala
  def listOf[A](a: Gen[A]): Gen[List[A]]
  ```
  - 생성할 목록의 **크기**를 명시하지 않음
  - 이 함수를 구현할 수 있으려면, 생성기가 특정 크기를 가정하고 있거나,
    - 또는 생성기에게 크기를 알려줄 수단이 필요
  - 특정 크기를 가정하는 것은 **유연하지 않음**
    - 모든 문맥에 `적합한 어떤 크기`가 존재하지 않을 것 같기 때문
- 크기를 명시적으로 정의하는 API는 다음과 같은 모습일 것
  ```scala
  def listOfN[A](n: Int, a: Gen[A]): Gen[List[A]]
  ```
  - 유용한 조합기이긴 하겠으나, **크기**를 명시적으로 지정할 필요가 없는 버전 역시
    - **강력한 조합기**
  - 그런 조합기가 있다면, 검사를 수행하는 함수가 **검례의 크기**를 임의로 정할 수 있는데
    - 그러면 앞에서 언급한 **검례 최소화**가 가능함
- 프로그래머가 지정한 크기의 검례들만 사용해야 한다면
  - **검사 실행 함수**가 그런식으로 유연하게 대응할 수 없음
  - 라이브러리의 설계 과정을 따라가면서, 이 관심사를 항상 염두하기
- `forAll` 함수
  - `Gen[List[Int]]`와 그에 대한 술어 역할을 하는 것으로 보이는 `List[Int] => Boolean`을 받음
  - `forAll` 역시 생성기와 **술어**의 형식을 구체적으로 알아야 할 필요는 없음
    - 둘의 형식이 맞아떨어지기만 하면 됨
      ```scala
      def forAll[A](a: Gen[A])(f: A => Boolean): Prop
      ```
- `Gen`과 술어를 묶은 결과를 나타내기 위해 `Prop`이라는 새로운 형식을 임의로 도입
  - `ScalaCheck` 관례에 따라 `property`를 줄인 이름
- `Prop`의 내부 표현이나, `Prop`이 지원하는 다른 함수들은 아직 못한다 하더라도,
  - `Prop`에 `&&` 연산자가 있음은 확실함
    ```scala
    trait Prop { def &&(p: Prop): Prop}
    ```