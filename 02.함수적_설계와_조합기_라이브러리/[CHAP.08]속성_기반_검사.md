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

## 8.2.2. 속성의 의미와 API
- `Prop` 형식을 위한 함수는 세 가지
  - 속성을 생성하는 `forAll`
  - 속성들의 합성을 위한 `&&`
  - 속성의 점검을 위한 `check`
    - console 출력이라는 부수 효과도 존재
    - 이 함수를 **편의용 함수**로 노출하는건 괜찮으나,
    - **합성의 기저**로 사용할 수는 없음
    - e.g. `&&`의 표현이 단순 `check`메서드라면, `Prop`을 위한 `&&`을 구현할 수 없음
- 코드
  ```scala
  trait Prop {
    def check: Unit
    def &&(p: Prop): Prop = ???
  }
  ```
- `check`에는 부수효과가 있기 때문에
  - `&&`을 구현하는 유일한 선택은
  - 두 `check`메서드를 모두 실행하는 것
- 만일 `check`가 **검사 결과 메세지**를 출력한다면
  - `&&`에 의해 두 개의 메시지가 출력
  - 그 메세지들은 두 속성의 **성공/실패 여부**를 각자 따로 판정한 결과
  - 이는 정확한 구현은 아님
- `check`에 `부수 효과가 있다는 것` 자체는 큰 문제는 아님
  - 더 일반적으로 보았을때, `check`가 정보를 **폐기**하는 것이 중요한 문제
- `Prop` 값들을 `&&`와 같은 조합기를 이용하여 조합하려면,
- `check`(또는 그 속성을 **실행**하는 어떤 함수)는 뭔가 **의미 있는 값**을 돌려주어야 함
  - 그 형식은?
- 형식 선택을 위해서,
  - 속성을 검사하여 얻고자 하는 정보를 먼저 구상
  - 적어도 속성 검사의 **성공/실패 여부**는 알아야 함
- 코드
  ```scala
  trait Prop {
    def check: Boolean
  }
  ```
  - 위 표현에서 `Prop`은 **비엄격 Boolean**값
  - 모든 `Boolean`에 대해 정의할 수 있음
  - 단, `Boolean`만으로 부족할 수 있음
    - 어떤 속성이 실패했을 때, `성공한 검례 수`, `실패 유발한 인수들이 무엇인지`에 대한 정보 등 ..
- 위 내용을 보완하여,
  - **성공/실패 여부**를 나타내기 위해 `Either`를 사용한 코드
    ```scala
    object Prop {
      // 이런 별칭은, API의 가독성에 도움이 됨
      type SuccessCount = Int
      ...
    }
    trait Prop { def check: Either[???, SuccessCount]}
    ```
- 실패의 경우에는
  - 실패 사실을 화면에 출력, 검사를 실행한 사람이 볼 수 있도록 해야 함
  - 검사의 목적은 버그를 찾아내고, 그 버그를 드러나게 한 **검례**가 무엇인지 제시해야 함
- 일반적인 규칙으로
  - 계산에 사용할 자료를 나타내는데 `String`은 부적합
  - 사람에게 표시할 자료라면, `String`을 사용하는데 문제 없음
- 위 논의를 정리한 코드
  ```scala
  object Prop {
    type FailedCase = String
    type SuccessCount = Int
  }

  trait Prop {
    def check: Either[(FailedCase, SuccessCount), SuccessCount]
  }
  ```
  - 실패의 경우 `Left((s, n))`을 돌려줌
    - `s`: 속성이 실패하게 만든 값을 나타내는 `String`
    - `n`: 실패 이전에 성공한 검례들의 수
- `check`의 인수
  - 현재는 인자가 없으나 충분할까?
  - `Prop`이 접근해야 할 정보는 `Prop`이 생성되는 방식을 보면 파악 가능
    ```scala
    // Prop을 생성하는 forAll 함수
    def forAll[A](a: Gen[A])(f: A => Boolean): Prop
    ```
- `Gen`의 구체적인 표현을 모르기 때문에,
  - `A`형식의 값을 생성하는데 충분한 정보(`check`의 구현에 필요한)가 갖춰졌는지 판단하기 어려움


## 8.2.3. 생성기의 의미와 API
- `Gen[A]`, A형식의 값을 생성하는 방법을 아는 어떤 것
  - 그런 값들을 생성하는 방법?
- 한 가지: 그런 형식의 값을 **무작위**로 생성
- `Gen`을 **난수 발생기**에 대한 `state` 전이를 감싸는 하나의 형식으로 만들기
  - `case class State[S,A](run: S => A,S))`
- 형태
  ```scala
  case class Gen[A](sample: State[RNG, A])

  def choose(start: Int, stopExclusive: Int): Gen[Int]
  def unit[A](a: => A): Gen[A] // 항상 a값 생성
  def boolean: Gen[Boolean]
  def listOfN[A])(n: Int, g: Gen[A]): Gen[List[A]] // 생성기 g를 이용하여 길이가 n인 목록 생성
  ```
- 어떤 연산들이 **기본 수단**이고
  - 어떤 연산들이 그로부터 **파생된 연산**들인지 이해하고
  - 작지만 표현력 있는 **기본수단**들의 집합을 구하는 것이 핵심
- 기본수단들의 집합으로 표현할 수 있는 대상들을 찾는 좋은 방법 중 하나는
  - 표현하고자 하는 `구체적인 예제`들을 선택하고
  - 그에 필요한 **기능성**을 조합할 수 있는지 **시험**
- 그 과정에서 **패턴**들을 인식하고,
  - 그 패턴들을 추출하여 조합기를 만들고
  - 기본수단들의 집합을 **정련**해 나감
- 구체적인 예제
  - 어떤 구간안의 `Int` 값 하나를 생성하는 **기본 수단**이 있다고 할 때,
    - 어떤 구간 안의 `(Int, Int)` 쌍을 생성하기 위해, 새로운 기본수단이 필요할까?
  - `Gen[A]`에서 `Gen[Option[A]]`를 얻을 수 있을까?
    - `Gen[Option[A]]`에서 `Gen[A]`를 얻는것은 어떨까?
  - 기존의 기본수단들을 이용해서 어떤 방식으로든 **문자열**을 생성할 수 있을까?

#### 놀이의 중요성
- API 설계시 너무 구체적인 예시에 집중하면 
  - 설계 공간의 일부 측면 간과
  - 임시방편적이고, 과도하게 세부적인 기능을 가진 API 생성
- 바람직한 접근 방식
  - 주어진 문제를 **정수**(essense)로 응축하는 것
  - 그 축약을 위한 최고의 방법은 **놀이**
- 중요한 문제를 풀거나, 유용한 기능성을 만들어 내려고 노력하지 말기
  - 그냥 서로 다른 표현과 기본수단, 연산을 시험해보면서 의문이 자연스럽게 떠오르게 하기
    - `이 두 함수는 비슷해 보인다. 좀 더 일반적인 연산이 숨어있지는 않을까?`
    - `이 자료 형식을 다형적으로 만드는 것이 합당할까?`
    - `표현의 이 측면을 하나의 값에서 값들의 List로 바꾸는 것은 어떤 의미일까`
  - 그 어떤 것이든 독자의 궁금증을 자극하는 것을 탐구할 것
- 어디에서 시작하는 것은 중요하지 않음
  - 계속 놀다 보면, 모든 **설계상**의 선택을 독자가 결정하게 될 때까지
  - 문제 영역이 독자를 이끌 것

## 8.2.4. 생성된 값들에 의존하는 생성기
- `둘쨰 문자열`이 `첫 문자열`의 문자들로만 이루어진다는 조건을 만족하는
  - **문자열 쌍 생성기**(`Gen[(String, String)]`)이 있다고 가정
- 또는 `Gen[Int]`로 `0 ~ 11` 사이의 정수를 선택하고,
  - `Gen[List[Double]]`이 그 값을 **목록의 길이**로 사용해서 목록을 생성하나독 가정
- 두 경우 모두 **일정한 의존성**이 존재
- 먼저 `하나의 값`이 생성되고
  - 그 값은 **생성기**가 다음에 생성할 값들을 결정하는데 쓰임
- 이런 생성 방식을 지원하려면
  - **한 생성기**가 **다른 생성기**에 의존하게 만드는 `flatMap`이 필요
- 예시
  ```scala
  def flatMap[B](f: A => Gen[B]): Gen[B]
  // listOfN의 동적인 버전
  def listOfN(size: Gen[Int]): Gen[List[A]]

  // 생성기를 결합하는 `union`, 각 생성기의 값을 동일 확률로 뽑아서 취합
  def union[A](g1: Gen[A], g2: Gen[A]): Gen[A]

  // union과 비슷하지만, 가중치를 받아 `Gen`에서 가중치에 비례하는 확률들로 뽑는 weighted
  def weghted[A](g1: (Gen[A],Double), g2: (Gen[A],Double)): Gen[A]
  ```

## 8.2.5. Prop 자료 형식의 정련
- `Gen`의 표현을 고찰하면서 `Prop`의 요구사항에 관한 정보를 얻음
- `Prop`의 정의
  ```scala
  trait Prop {
    def check: Either([FailedCase, SuccessCount), SuccessCount])
  }
  ```
- 단순 **비엄격 `Either`**
- 여기엔 몇가지 정보가 빠져 있음
  - **성공한 검례 수**는 `SuccessCount`에 담겨있지만,
  - 속성이 **검사**를 통과했다는 결론을 내리는 데 **충분한 검례의 수**는 지정하지 않음
- 그 개수를 코드에 직접 작성할 수 있으나,
  - 그러한 **의존성**을 적절히 **추상화**하기
    ```scala
    type TestCases = Int
    type Result = Either[(FailedCase, SuccessCount), SuccessCount]
    case class Prop(run: TestCases => Result)
    ```
- 현재 **성공적인 검사의 수**를 `Either` 양변 모두에 기록
  - 그러나 어떤 한 **속성**이 **검사**를 통과했다면
  - 이 **통과된 검례의 수**가 `run`의 인수와 동일함을 의미
- 따라서 **성공 횟수**를 알려준다고 해도
  - `run`의 호출자가 새로운 것을 알게되는 것은 아님
- `Either`의 `Right` 경우에 어떠한 정보도 필요하지 않으므로, 이를 `Option`으로 변경
  ```scala
  type Result = Option[(FailedCase, SuccessCount)]
  case class Prop(run: TestCaes => Result)
  ```
- `None`은 모든 검례가 **성공**해서
  - 속성이 검사를 통과했음을 뜻함
- `Some`은 **실패**가 있었다는 것을 뜻함
  - 이상한 점?
    - 기존에는 `None`이 실패를 의미했지만, 지금은 **부재**를 나타내는 용도로 사용
    - `Option`에 적법한 용도이나, **의도가 명확하지 않음**
- `Option[(FailedCase, SuccessCount)]`에 해당하되,
  - 의도를 명확히 나타낼 수 있는 새로운 자료 형식 구성

#### CODE.8.2. 새로운 Result 자료 형식
```scala
sealed trait Result {
  def isFalsified: Boolean
}
case object Passed extends Result { // 모든 검사를 통과했음을 나타냄
  def isFalsified = false
}
case class Falsified(failure: FailedCase, // 검사들 중 하나가 속성을 반증
                     successes: SuccessCount) extends Result {
  def isFalsified = true
}
```
- 위 내용으로 `Prop`의 표현으로 충분할지?
- `forAll`이 구현 가능한지 살펴보기
  ```scala
  def forAll[A](a: Gen[A])(f: A => Boolean): Prop
  ```
- `forAll`이 가진 정보는 `Prop`을 돌려주기에 충분하지 않음
  - `Prop.run`에는 **시도할 검례 수**외에도
  - **검례들을 생성하는 모든 정보**가 필요함
- `Prop.run`이 `Gen`의 현재 표현을 이용해서 **무작위 검례**를 생성해야 한다면
  - `RNG`가 필요할 것
- 해당 의존성으로 `Prop`에 표현
  ```scala
  case class Prop(run: (TestCases, RNG) => Result)
  ```
- 추후 **검례 개수**와 **무작위성의 원천**외에도, 다른 **의존성**이 필요할 경우
  - 그에 해당하는 **매개변수**를 `Prop.run`에 추가하면 됨

#### CODE.8.3. forAll의 구현
```scala
def forAll[A](as: Gen[A])(f: A => Boolean): Prop = Prop { // (a,i) 쌍들의 스트림, a는 무작위 값, i는 스트림 안에서의 그 값의 색인
  (n, rng) => randomStream(as)(rng).zip(Stream.from(0)).take(n).map {
    case (a, i) => try {
      if (f(a)) Passed else Falsified(a.toString, i) // 검사 실패시, 실패한 검례 및 그 색인을 기록. 이를 통해 성공한 검사들의 개수 파악 가능
    } catch { case e: Exception => Falsified(buildMsg(a, e), i)} // 검례 예외 발생시, 그 사실을 결과에 기록
  }.find(_.isFalsified).getOrElse(Passed)
}

def randomStream[A](g: Gen[A])(rng: RNG): Stream[A] =
  Stream.unfold(rng)(rng => Some(g.sample.run(rng))) // 생성기에서 표본을 거듭 추출, A의 무한 스트림 생성

def buildMsg[A](s: A, e: Exception): String =
  s"test case: $s\n" + // 문자열 보간 구분
  s"generated an exception: ${e.getMessage}\n" +
  s"stack trace:\n ${e.getStackTrace.mkString("\n")}"
```
- 예외가 발생하면 `run`이 그것을 던지게 되면,
  - **실패**를 유발한 인수에 관한 정보가 사라짐
- 이에, **예외**를 잡아서 **실패**로서 보고하게 한다는점에 주목

## 8.3. 검례 최소화
- 검례 최소화
  - **검사 실패**시 **검사 프레임워크**가
  - 해당 검사를 `실패하게 만드는` 가장 작은 또는 **가장 간단한 검례**를 찾아내는 기법
- **최소 검례**는 `문제의 이해`와 `디버깅`에 도움이 됨
- 검례 최소화를 위한 접근방식(2)
  - **수축**(shrinking)
    - 실패한 검례 발생시
    - **개별적인 절차**를 띄워서
      - 그 검례의 **크기**를 점차 줄여가면서, 검사를 반복
      - 검사가 더 이상 실패하지 않을 경우 멈춤
    - 이를 구현하려면, **각 자료 형식**마다 **개별적인 코드 작성**이 필요
  - **크기별 생성**(sized generation)
    - **수축**하는 대신
      - 애초에 `크기와 복잡도`를 늘려가면서 **검례** 생성
      - 검사 실행기가 **적용 가능한 크기**들의 공간을 크게 건너 뛰면서도
        - 여전히 가장 작은 **실패 검례**를 찾을 수 있도록 **확장**하는 방법도 여러가지
- `ScalaCheck`는 **수축**을 사용
  - 이 책에서는 이와 다른 **크기별 생성 방식**을 사용
  - 조금 더 간단하며, 어떤 의미로는 **모듈성**이 더 좋기 때문
    - 생성기들이 `주어진 크기의 검례를 생성하는 방법`만 알면 되기 때문
    - 생성기는 **검례 공간**들의 공간을 검색하는데 쓰이는 `일정`에 신경쓸 필요가 없음
    - 구체적인 일정은 **검사들을 실행하는 함수**가 선택 가능
- `Gen` 자료 형식에 대한 **유용한 조합기**들을 여러개 작성했으므로, 이 형식은 더 이상 고치지 말고
  - 크기별 생성을 **개별적인 계층**으로서 라이브러리에 도입
- 크키별 생성기의 간단한 표현, 크기를 받아 **생성기** 산출
  ```scala
  case class SGen[+A](forSize: Int => Gen[A])
  ```
- `SGen`이 `Prop`과 `Prop.forAll` 정의에 미치는 영향
  ```scala
  def forAll[A](g: SGen[A])(f: A => Boolean): Prop
  ```
- 위 함수는 구현할 수 없음
  - `SGen`이 크기를 알아야 하나, `Prop`에는 어떤 크기 정보도 주어지지 않음
- 무작위성의 원천과 검례의 개수 정보를 처리할 떄 처럼
  - 그냥 이것을 `Prop`의 한 의존성으로 추가하면됨
- 다양한 크기의 바탕 생성기들을 실행하는 **책임**을 `Prop`에 부여하고자 함
  - 그러려면 `Prop`에 **최대 크기**를 지정할 필요가 있음
  - `Prop`은 지정한 **최대 크기**까지의 **검례**들을 생성
  - 이럴 경우, `Prop`이 **가장 작은 검사 실패 검례**를 검색하는 것도 가능

#### CODE.8.4. 주어진 최대 크기까지의 검례들을 생성
```scala
type MaxSize = Int
case class Prop(run: (MaxSize,TestCases,RNG) => Result)

def forAll[A]](g: SGen[A])(f: A => Boolean): Prop = forAll(g(_))(f)

def forAll[A](g: Int => Gen[A])(f: A => Boolean): Prop = Prop {
  (max,n,rng) => 
    val casePerSize = (n + (max - 1)) / max // 각 크기마다 이만큼의 무작위 검례 생성
    val props: Stream[Prop] =
      Stream.from(0).take((n min max) + 1).map(i => forAll(g(i))(f)) // 각 크기별 하나의 속성 생성. 전체 속성 개수가 n개를 넘지는 않도록
    val prop: Prop =
      props.map(p => Prop { (max, _, rng) => 
        p.run(max, casesPerSize, rng)
      }).toList.reduce(_ && _) // 모든 속성을 하나의 속성으로 결합
    prop.run(max, n, rng)
}
```