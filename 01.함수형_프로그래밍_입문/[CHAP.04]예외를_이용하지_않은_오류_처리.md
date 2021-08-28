# [CHAP.04] 예외를 이용하지 않은 오류 처리
- **예외**를 던지는 것이 하나의 **부수 효과**
- **함수적 코드**에서 **예외**를 사용하지 않는다면,
  - 그 대신 무엇을 사용할까?
- 오류를 **함수적**으로 제기하고, 처리하는데 필요한 **기본 원리**에 대해
- **실패 상황**과 **예외**를 **보통의 값**으로 표현할 수 있으며
  - 일반적인 **오류 처리/복구 패턴**을 추상화한 **고차 함수**를 사용할 수 있음
- **오류를 값으로 돌려준다는 함수적 해법**
  - 더 안전하고, **참조 투명성**을 유지함
- 고차 함수 덕분에 예외의 주된 이점인 **오류 처리 논리의 통합**(consolidation of error-handling logic)도 유지됨
- 표준 라이브러리의 `Option`과 `Either`를 직접 구성해보기

## 4.1. 예외의 장단점
- 예외가 **참조 투명성**을 해침
  - 함수/메서드가 **함수 외부의 영향을 받지 않음**
  - 함수 인자에만 의존적(입력 콘솔, 파일, 원격 url등에 의존적이지 않음)

### CODE.4.1. 예외를 던지고 받기
```scala
def failingFn(i: Int): Int = {
  val y: Int = throw new Exception("fail!")
  try {
    val x = 42 + 5
    x + y
  }
  catch { case e: Exception => 43}
}
```
- `failingFn`를 호출하면, 예상대로 오류 발생
- `y`가 **참조에 투명하지 않음**
  - `임의의 참조 투명 표현식을, 그것이 지정하는 값으로 치환해도 프로그램의 의미가 변하지 않아야 함`
  - `x + y`의 `y`를 `throw new Exception("fail!")`로 치환하면, 그전과는 완전히 다른결과가 나옴

### 참조 투명성
- 참조에 투명한 표현식의 의미는
  - `context`에 의존하지 않으며
  - 지역적으로 추론할 수 있지만, 참조에 투명하지 않은 표현식의 의미는
    - `context-dependent`하며, 좀 더 **전역의 추론**이 필요하다는 것으로 이해해도 됨
- 예를 들어 `42 + 5`의 의미는
  - **그 표현식**을 포함한 **더 큰 표현식**에 의존하지 않음
  - 이 표현식은 언제나 `47`이라는 값을 리턴함
- 하지만 `throw new Exception("fail")`이라는 표현식은 **문맥에 의존적**
  - `try`블록에 포함되어 있는지, 어떤 `try`블록인지에 따라 달라짐

### 예외의 주된 문제(2)
- **예외는 참조 투명성을 위반하고 문맥 의존성을 도입**
  - **치환 모형**의 간단한 추론이 불가능해지고,
  - **예외**에 기초한 혼란스러운 코드 유발
  - 예외를 **오류 처리**에만 사용하고, **흐름 제어에는 사용하지 말아야 함**
- **예외는 형식에 안전하지 않음**
  - `failingFn`의 형식인 `Int => Int`로는 예외를 던질지 알 수 없음
  - compiler는 `failingFn`의 호출자에게 **그 예외를 처리하는 방식을 결정하라고 강제 불가**
  - 예외 점검 코드를 추가하지 않게 되면, `RunTime`에서야 검출 됨

### 예외 대체 방안
- 기본 장점
  - **오류 처리 논리의 통합과 중앙집중화**를 유지하는 대안이 있으면 좋을 것
    - 오류 처리 논리를 **코드 기반**의 여기 저기에 널어놓지 않아도 되도록 하는 대안
- **예외를 던지는 대신, 예외적인 조건이 발생했음을 뜻하는 값을 돌려준다**라는 오래된 착안에 기초
- `C`에서는 예외 처리를 위해 `error code`를 돌려주는 방법과 유사
  - 단, 오류 부호를 직접 돌려주는 대신, 그런 **미리 정의해 둘 수 있는 값들**을 대표하는 새로운 일반적 형식을 도입하고,
  - 오류의 처리와 전파에 관한 **공통적인 패턴**들을 고차 함수들을 이용해 캡슐화
- `C` 스타일의 **오류 부호**와는 달리
  - **형식에 완전히 안전**하며
  - 최소한의 구문적 잡음으로도 스칼라의 **형식 점검기**의 도움을 받아 **실수를 미리 발견하는 것**

### 점검된 예외
- `Java`의 `checked exception`은 적어도 오류를 **처리할 것인지, 다시 발생시킬 것인지** 결정을 강제
  - 결과적으로 호출하는 쪽에 판에 박힌(**boilerplate**) 코드가 추가됨
- 점검된 예외는 **고차 함수에는 통하지 않음**
- 고차 함수에서는 인수가 구체적으로 **어떤 예외를 던질지 미리 알 수 없음**
- 예시 코드
  ```scala
  def map[A,B](l: List[A])(f: A => B): List[B]
  ```
  - 함수 자체는 일반적임이 명백하나, **점검된 예외**와는 잘 맞지 않음
  - 모든 가능한(`f가 던질 수 있는`) 점검된 예외마다, `map`의 개별적인 버전을 만들수는 없는 일
- `Java`에서도 **일반적 코드**가 `RuntimeException`이나 어떤 공통의 점검된 `Exception` 형식에 의존할 수 밖에 없는 이유

## 4.2. 예외의 가능한 대안들
- 평균을 구하는 함수 예시
  - 빈목록에 대해서 평균이 정의되지 않음
  - 코드
    ```scala
    // Seq : 선형 순차열 비슷한 컬렉션들의 공통 인터페이스
    def mean(xs: Seq[Double]): Double = 
      if (xs.isEmpty)
        throw new ArithmeticException("mean of empty list")
      else xs.sum / xs.length // sum은 순차열의 요소가 수치 형식일 때만 `Seq`의 메서드로 정의됨
      // 표준 lib에서는 implicit 클래스를 이용하여 구현
    ```
- `mean`은 `partial function`의 예시
- **부분 함수**란 **일부 입력**에 대해서는 정의되지 않는 함수를 의미
  - 자신이 받아들이는 입력에 대해
    - **입력 형식**만으로는 결정되지 않는 어떤 **가정을 두는 함수**
- 받아들여지지 않는 상황에 대해 **예외**를 던질 수 있으나, 꼭 그러지 않아도 됨

### mean 함수의 예외의 대안

#### 첫번째 대안
- `Double`형식의 가짜 값을 돌려주는 것
  - 빈 목록에 대해 `0.0/0.0` 연산, `Double.NaN`을 리턴
  - 또는 어떤 경계 값(`sentinel value`)를 돌려줄 수 있음
  - 상황에 따라 `null`로 리턴 가능
- 위 방법을 사용하지 않는 이유
  - **오류의 소리 없는 전파**
    - 어떤 오류 조건의 점검을 빼먹어도, `compiler`가 경고하지 않음
    - 오류의 코드 확인 시점이 늦어짐
  - 호출자가 **진짜 결과**를 받았는지 점검하는 `명시적 if문`으로 구성된 코드가 많아짐
  - 다형적 코드에는 적용 불가
    - 출력 형식에 따라 **경계값을 정할 수 없음**
      - e.g. 리턴값이 `Boolean`인 상황에서, 그 이외에 값 리턴 불가능 등
  - 호출자에게 **특별한 방침**이나 **호출 규약**을 요구
    - `mean`을 제대로 사용하려면, 결과를 받아내는 것 이외의 작업이 수반됨
    - **특별한 방침**이 많아질 수록,
      - 모든 인수를 **균일한 방식**으로 처리해야하는 **고차 함수**에 전달하기 어려워짐

#### 두번째 대안
- 함수가 입력을 처리할 수 없는 상황이 생겼을 때, 무엇을 해야하는지 **인수**를 호출자가 지정하기
- 코드
  ```scala
  def mean_1(xs: IndexedSeq[Double], onEmpty: Double): Double =
    if (xs.isEmpty) onEmpty
    else xs.sum / xs.length
  ```
- `mean`은 `total function`이 됨
- 그러나, **직접적인 호출자**가 이 함수의 처리방식을 알고 있어야 함
  - 그런 경우에도, 항상 하나의 `Double`값을 결과로 돌려주어야 함
- 결국, `정의되지 않는 경우`가 가장 적당한 수준에서 처리되므로,
  - 그 처리 방식의 결정을 미룰 수 있게 해야 함

## 4.3. Option 자료 형식
- 함수가 항상 **답을 내지 못한다는 점**을 **반환 형식**을 통해서 명시적으로 표현하기
- **오류 처리 전략**을 **호출자**에게 미루는 것
- 코드
  ```scala
  sealed trait Option[+A]
  case class Some[+A](get: A) extends Option[A] // 정의할 수 있는 경우
  case object None extends Option[Nothing] // 정의할 수 없는 경우
  ```
  - `Option`에는 위와 같이 두 개가 존재
- `mean`의 예시
  ```scala
  def mean(xs: Seq[Double]) : Option[Double] =
    if (xs.isEmpty) None
    else Some(xs.sum / xs.length)
  ```
  - 함수의 **결과가 항상 정의되지 않음**
    - **반환 형식**에 반영됨
  - `Option[Double]`이라는 결과를 돌려주어야 하기 때문에
    - `mean`은 **완전 함수**이다
  - 모든 **입력 형식**의 값에 **하나의 정의된 출력 형식 값**을 돌려줌

### 4.3.1. Option의 사용 패턴
- `FP`에서는 **부분 함수**의 **부분성**을 흔히
  - `Option` 같은 자료 형식(and `Either` 자료 형식)으로 처리
- 예시
  - `Map`에서 주어진 키를 찾는 함수, `Option` 반환
  - `headOption`, `lastOption`, 순차열이 비지 않은 경우 `Option`형태의 값 반환
- `Option`이 편리한 이유
  - **오류 처리**의 **공통 패턴**을 **고차 함수**를 이용하여 추출
  - 예외 처리 코드에 흔히 수반되는 판에 박힌 코드를 작성하지 않아도 됨

#### Option에 대한 기본적인 함수들
- `Option`은 최대 하나의 원소를 담을 수 있음
  - 이를 제외하면 `List`와 유사
- `map, flatMap, getOrElse, orElse, filter`등이 가능
- `non-strictness`(비엄격성)
  - `getOrElse`의 `default: => B`라는 형식 주해는
  - 인수 형식이 `B`이지만, **그 인수가 함수에 실제로 쓰일때까지**는
    - 평가되지 않음을 의미
- `getOrElse`에서 `B >: A`는
  - `B`가 반드시 `A`와 같거나, `A`의 **상위 형식**임을 의미
- `Option[+A]`를
  - `A`의 **공변 형식**으로 선언해도 안전하다고 추론하게 하려면, 이와 같이 지정

#### 기본적인 Option 함수들의 용례
- 위에서 말한 **고차 함수**를 주로 사용
- `map` 함수 : `Option`안에 결과가 있다면 변환
  - 예시
    ```scala
    case class Employee(name:String, department: String)
    def lookupByName(name: String): Option[Employee] = ...
    val joeDepartment: Option[String] = lookupByName("Joe").map(_.department)
    ```
  - 만약 `lookupByName`이 `None`을 돌려 주었다면, 계산의 나머지 부분이 **취소됨**
    - 그에 따라 `_.department` 함수를 전혀 호출하지 않음
- 예시2
  ```scala
  def variance(xs: Seq[Double]): Option[Double]
  ```
  - 어떤 단계라도 실패하면, 그 즉시 **나머지 모든 과정이 취소됨**
  - `None.flatMap(f)`가 `f`를 실행하지 않고 즉시 `None`을 돌려주기 때문
- 흔한 사용 패턴은, `map`, `flatMap`, `filter`의 임의 조합을 사용하여 `Option`을 변환한 뒤,
  - `getOrElse`를 이용하여 오류 처리를 수행하는 것
  - 코드
    ```scala
    val dept:String = lookupByName("Joe").map(_.dept).filter(_ != "Accounting").getOrElse("Default Dept")
    ```
    - `getOrElse`의 경우, `Joe`가 존재하지 않거나, `Accounting`일 경우 기본값 반환 목적
- `orElse`는 `getOrElse`와 유사하나
  - 첫 `Option`이 정의되지 않으면, 다음 `Option`을 리턴
  - 실패할 수 있는 계산들을 연결하여, 첫 계산이 성공하지 않으면, 둘째 것을 시도하기 위함
- 흔한 관용구로,
  ```scala
  o.getOrElse(throw new Exception("FAIL"))
  ```
  - `Option`의 `None`을 **예외**로 처리할 수 있게 함
- 관련된 **일반적인 법칙**은
  - 합리적인 프로그램이라면, 결코 **예외**를 잡을 수 없을 상황에서만 **예외를 사용하는 것**
- 호출자가 **복구 가능한 오류**로 처리할 수 있는 상황이라면,
  - 예외 대신 `Option`(또는 `Either`)을 돌려주어, 호출자에게 **유연성** 부여
- `Option`을 사용함으로써
  - 매 단계마다 `None`을 점검할 필요가 없음
  - 추가적인 안전성 확보
    - `None`일 수 있는 상황에서의 처리를 명시적으로 **지연** 또는 수행하지 않으면 **컴파일 에러**

### 4.3.2. 예외 지향적 API의 Option 합성과 승급, 감싸기
- `Option`을 받거나, 돌려주는 메서드를 호출하는 모든 코드를
  - `Some`이나 `None`으로 처리하도록 수정해야 한다?
  - 하지만 그럴 부담은 필요 없음
  - **보통의 함수를 `Option`에 대해 작용하는 함수로 승급시킬(lift) 수 있음**
- 코드
  ```scala
  // Option[A] 형식의 값을, A => B 함수를 이용하여, Option[B]를 리턴하도록 하기
  def lift[A,B](f: A => B): Option[A] => Option[B] = _ map f
  ```
- 위 함수를 사용하여 `Option` 값의 **문맥 안에서** 작용하도록 변경 가능
  ```scala
  val abs0: Option[Double] => Option[Double] = lift(math.abs)
  ```
- `Option`값에 작용하는 함수를 직접 구현하지 않고,
  - `Option` 문맥으로 `lift`하여 사용
- `lift`는 모든 함수에 적용 가능
- 예시 : 온라인 견적에 따른 `form`을 제출하는 페이지를 논리적으로 구현하기
  ```scala
  // 보험료를 계산하기
  def insuranceRateQuote(age: Int, numberOfSppedingTickets: Int): Double

  // 고객이 제출한 양식을 파싱하기(단, 문자열 파싱 실패 가능성 존재)
  def parseInsuranceRateQuote(
    age: String,
    numberOfSpeedingTickets: String): Option[Double] = {
      val optAge: Option[Int] = Try(age.toInt) // 모든 String에 대해 `toInt`메서드 사용 가능
      val optTickets: Option[Int] = Try(numberOfSpeedingTickets.toInt)
      insuranceRateQuote(optAge, optTickets) // 형식을 점검하지 않음
    }

  def Try[A](a: => A): Option[A] = // A 인수를 엄격하지 않은 방식으로 받아들임. a를 평가하는 도중에 예외 발생시 `None`으로 변환 가능하도록 하기 위함
    try Some(a)
    catch { case e:Exception => None} 
  ```
- `try` 함수는 **예외 기반 API**를 `Option 지향적 API`로 변환하는데 사용할 수 있는 **범용 함수**
- `lazy`인수를 사용
  - `a`의 형식 주해 `=> A`가 나타내는 의미
- `optAge`와 `optTickets`를 파싱하여 `Option[Int]`를 가지고 `insuranceRateQuote`를 호출해야 하지만,
  - 이 함수는 두개의 `Int` 값을 받음
- `insuranceRateQuote`를 변경하기엔, 나머지 문맥에서 충돌 가능성 존재
  - `optional`값들의 문맥에서 작동하도록 **승급**하는 것이 바람직
- 개선된 코드
  ```scala
  def parseInsuranceRateQuote(
    age:String,
    numberOfSpeedingTickets: String): Option[Double] = {
      val optAge:Option[Int] = Try {age.toInt} // 인수 하나를 받는 함수는 중괄호 대신, 대괄호 호출 가능(= Try(age.toInt))
      val optTickets:Option[Int] = Try {numberOfSpeedingTickets.toInt}
      map2(optAge, optTickets)(insuranceRateQuote) // 둘 중 하나라도 파싱 실패시, None 반환
    }
  ```
  - `map2`는 **인수가 두개인** 어떤 함수라도 `Option에 대응하게`만들 수 있음을 의미
    - 예전에 만들어둔 함수라도, `Option`의 문맥에서 작동하도록 수정 가능
- 실패할 수 있는 함수를 목록에 **사상**했을 때,
  - 목록의 원소중  하나라도 `None`을 돌려주면, 전체가 `None`이 되게끔 해야하는 경우도 존재
- 예시 : 목록에 담긴 `String`의 모든 값이 `Option[Int]`로 파싱되어야 하는 경우
  - `map`의 결과를 `sequence`로 순차 결합하면 됨
  - 코드
    ```scala
    def parseInts(a: List[String]): Option[List[Int]] = sequence(a map (i => Try(i.toInt)))
    ```
  - 하지만 위 방법은 목록을 두번 순회해야하기 때문에 **비효율적**
  - 다음과 같은 서명의 일반적 함수 `traverse`를 만들어두기
    ```scala
    def traverse[A, B](a: List[A])(f: A => Option[B]):Option[List[B]]
    ```
- `map, lift, sequence, traverse, map2, map3`와 같은 함수들이 있으면,
  - **생략적 값**을 다루기 위해, **기존 함수를 수정해야 할 일이 전혀 없어야 하는 것이 정상**
#### for-comprehension
- for-함축
- 자동으로 일련의 `flatMap`, `map` 호출들로 전개
- 원래의 버전 코드
  ```scala
  def map2[A,B,C](a:Option[A], b:Option[B])(f: (A,B) => C):
    Option[C] = 
      a flatMap (aa => 
        b map (bb => f(aa, bb)))
  ```
- 동일한 코드를 `for-comprehension`을 통해 구현
  ```scala
  def map2[A,B,C](a:Option[A], b:Option[B])(f: (A,B) => C):
    Option[C] = for {
      aa <- a
      bb <- b
    } yield f(aa, bb)
  ```
- `for-comprehension`은
  - **중괄호 쌍**에 `aa <- a`와 같은 일련의 묶음(binding)들이 있고,
  - 그 다음에 `yield` 표현식이 오는 형태
  - `yield` 키워드 다음의 표현식에는
    - 일련의 `<-` 묶음 **좌변에 있는 값**을 사용할 수 있음
- 컴파일러는 이 묶음들을 `flatMap` 호출로 전개하되,
  - 마지막 묶음과 `yield`는 `map` 호출로 변환
- 명시적인 `flatMap`, `map` 호출이 나열된 코드라면 사용해볼 것

## 4.4. Either 자료 형식
- **실패**와 **예외**를 보통의 값으로 표현할 수 있다는 점
- 오류 처리 및 복구에 대한 **공통의 패턴**을 추상화하는 함수를 작성할 수 있다는 점
- `Option`이 많이 쓰이긴 하나, 다른 자료형식도 많음
- `Option`은 **예외적인 조건**이 발생했을 때
  - 무엇이 잘못되었는지에 대한 **정보를 제공할 수 없음**
  - **실패**시 단순히, 유효한 값이 없음을 뜻하는 `None`을 돌려줄 뿐
- 좀 더 자세한 정보를 담은 `String`을 돌려준다거나,
  - 예외가 발생한 경우 **실제로 발생한 오류가 어떤 것인지** 알 수 있는 무언가를 돌려주면 좋음
- 실패에 관해 알고 싶은 정보가 어떤 것이든
  - 그것을 **부호화**하는 **자료 형식**을 만드는 것은 가능함
- 단순히 실패가 발생했음을 알면 충분할 때는 `Option`을 사용
- **실패의 원인을 추적할 수 있는 `Either` 자료 형식**
- 함수 시그니처
  ```scala
  sealed trait Either[+E, +A]
  case class Left[+E](value: E) extends Either[E, Nothing]
  case class Right[+A](value: A) extends Either[Nothing, A]
  ```
- `Option`과의 차이
  - 두 경우 모두 값을 가짐
- `Either`는 **둘 중 하나**일 수 있는 값을 대표 함
- 이 형식은 두 형식의 `disjoint union`(분리합집합)이라 할 수 있음
- 이 형식을 **성공** 또는 **실패**를 나타낼 때,
  - `Right` 생성자를 **성공**을 나타내는데 사용하며
  - `Left` 생성자를 **실패**에 사용
    - 왼쪽 형식 매개변수로 `error`를 의미하는 `E`를 사용

#### 표준 라이브러리의 Option과 Either
- `Option`, `Either` 모두 표준 라이브러리에 존재
- 물론 현재 구현하는 내용과 몇몇 함수는 지원하지 않음
  - `sequence`, `traverse`, `map2`, ...

#### mean의 예시
```scala
def mean(xs: IndexedSeq[Double]) : Either[String, Double] =
  if (xs.isEmpty)
    Left("mean of empty list!") // 실패일 경우에 String 반환
  else
    Right(xs.sum / xs.length)
```
- 예외에 대한 자세한 정보를 담을 때 `exception`을 돌려주면 됨
  ```scala
  def safeDiv(x:Int, y:Int): Eighter[Exception, Int] =
    try Right(x/y)
    catch {
      case e: Exception => Left(e)
    }
  ```
- Try 예시 : 던저진 예외를 **값**으로 변환
  ```scala
  def Try[A](a: => A): Either[Exception, A] =
    try Right(a)
    catch {
      case e: Exception => Left(e)
    } 
  ```

#### for-comprehension에서의 `Either`
```scala
def parseInsuranceRateQuote(
  age: String,
  numberOfSpeedingTickets: String): Either[Exception, Double] =
    for {
      a <- Try { age.toInt}
      tickets <- Try {numberOfSpeedingTickets.toInt}
    } yield insuranceRateQuote(a, tickets)
```
- 실패시 단순 `None`이 아니라, 실제 예외에 대한 정보를 얻을 수 있음


#### CODE.4.4. Either를 자료 유효성 점검에 활용
- `map2`에 적용한 예시
- `mkPerson`은 주어진 이름과 나이의 유효성을 점검한 후,
  - 유효한 `Persion`을 생성
- 코드
  ```scala
  case class Person(name:Name, age: Age)
  sealed class Name(val value: String)
  sealed class Age(val value: Int)

  def mkName(name: String): Either[String, Name] =
    if(name == "" || name == null) Left("Name is empty")
    else Right(new Name(name))
  
  def mkAge(age: Int): Either[String, Age] =
    if(age < 0) Left("Age is out of range")
    else Right(new Age(age))
  
  def mkPerson(name: String, age: Int): Either[String, Person] =
    mkName(name).map2(mkAge(age))(Person(_, _))
  ```

## 4.5. 요약
- 예외를 **보통의 값**으로 표현하고
  - 고차함수를 이용하여, 오류 처리 및 전파의 **공통 패턴**을 **캡슐화**
- 임의의 값으로 표현
- 예외는 정말 **복구 불가능한 조건**에서만 활용
- `non-strict`(비엄격성) 개념
  - `orElse, getOrElse, Try`
  - 비엄격성이 어떻게 함수적 프로그래밍의 **모듈성**과 **효율성**을 높여주는지