# [CHAP.15] 스칼라 타입 시스템 2
- 스칼라를 배운지 얼마 되지 않은 경우에는 급하게 알 필요는 없음
- 단, 스칼라 프로젝트 진행, 서드파티 라이브러리를 사용함에 따라
  - 언젠가는 마주칠 내용
- 뒷부분에서 더 고급 예제를 다루며 **경로에 의존하는 타입**(path dependent type)의 케이스를 확인하지만,
  - 그에 대해 **깊이 이해할 필요는 없음**

## 15.1. 경로에 의존하는 타입
- **경로**식을 사용하여 내포시킨 타입 접근 가능
```scala
class Service {
    class Logger {
        def log(message: String): Unit = println(s"log: $message")
    }
    val logger: Logger = new Logger
}

val s1 = new Service

// Compile Error
val s2 = new Service {
    override val logger = s1.logger
}
```
- 스칼라는 각 `Service` 인스턴스의 `logger`를 **서로 다른 타입**으로 인식
  - 실제 타입은 **경로에 의존**(path-dependent)한다는 의미

### 15.1.1. C.this
- `C1` 클래스에 대해 `this`라는 이미 익숙한 식을 본문에 사용하여
  - **현재 인스턴스** 참조 가능
- 하지만 `this`는 스칼라에서 실제로 `C1.this`를 의미
  ```scala
  class C1 {
      var x = "1"
      def setX1(x: String): Unit = this.x = x
      def setX2(x: String): Unit = C1.this.x = x
  }
  ```
  - 어떤 타입의 본문 내부에서
    - 그러나 메서드 정의 바깥 부분에서
    - `this`는 해당 타입 자체를 의미
      ```scala
      trait T1 {
          class C
          val c1: C = new C
          // 메서드 정의 바깥에서 this 사용
          val c2: C = new this.C
      }
      ```
      - `this.C`의 `this`는 `T1`이라는 trait을 의미

### 15.1.2. C.super
- 어떤 타입의 부모를 `super`로 가리킬 수 있음
  ```scala
  trait X {
      def setXX(x: String): Unit = {}
  }
  class C2 extends C1
  class C3 extends C2 with X {
      def setX3(x:String): Unit = super.setX1(x)
      def setX4(x:String): Unit = C3.super.setX1(x)

      // 부모를 `C2`로 선택
      def setX5(x:String): Unit = C3.super[C2].setX1(x)

      // 부모를 `X`로 선택
      def setX6(x:String): Unit = C3.super[X].setXX(x)

      // Error, 조부모 타입을 참조 불가
      def setX7(x:String): Unit = C3.super[C1].setX1(x)
      // Error, super 연쇄 참조 불가
      def setX8(x:String): Unit = C3.super.super.setX1(x)
  }
  ```
  - `C3.super`는 `super`와 동일
  - 어떤 부모를 지칭하는지 `[T]`를 통해 알 수 있음
- 여러 조상이 있는 타입에
  - 특별히 타입 명시 하지 않고, `super`를 사용한다면
  - **선형화** 규칙이 `super`의 대상 결정
    - `CHAP.11.2` 객체의 상속 계층을 선형화하기 참조
  - `this`와 마찬가지로
    - **메서드 밖 타입 본문**에서는 `super`를 사용해서 부모 타입 참조 가능
      ```scala
      class C4 {
          class C5
      }
      class C6 extends C4 {
          val c5a: C5 = new C5
          val c5b: C5 = new super.C5
      }
      ```

### 15.1.3. 경로.x
- 내포된 타입 접근 => **마침표를 사용한 경로식**
- 어떤 **타입 경로**의 맨 마지막 부분을 제외하면
  - 나머지 부분은 **안정적**(stable)이어야 함
    - `package, singleton object, type alias, ...`
  - 타입의 마지막 부분은 **안정적이지 않아도 됨**
    - `class, trait, type member, ...`
- 예시
  ```scala
  package P1 {
      object O1 {
          object O2 {
              val name = "name"
          }
          class C1 {
              val name = "name"
          }
      }
  }
  class C7 {
      val name1 = P1.O1.O2.name // 필드에 대한 참조
      type C1 = P1.O1.C1 // 클래스에 대한 참조
      val c1 = new P1.O1.C1 // 클래스에 대한 참조
      
      // ERROR
      val name2 = P1.O1.C1.name // P1.O1.C1이 안정적이지 않음
  }
  ```
  - `name2`의 경우 `C1`과 같은 안정적이지 않은 요소가 중간에 쓰임
- **코드에서 복잡한 경로를 사용하지 않는 것이 더 좋음**

## 15.2. 의존적 메서드 타입
- `scala 2.10`에 도입, 의존적 메서드 타입(dependent method type)
- 경로에 의존하는 타입의 일종, 몇가지 설계 문제 해결시 유용

### 자석 패턴(Magnet Pattern)
- 처리를 위한 메서드 하나 존재, **자석**이라는 객체를 넘겨 받음
  - 그 객체(자석)은 `호환 가능한 반한 타입을 보장`
  - 기법의 자세한 예는 `spray.io` 블로그 참조
- 예시
  ```scala
  import scala.concurrent.{Await, Future} // 비동기 계산에 필요

  // 계산 응답을 반환하기 위한 두가지 case class 정의
  // 두 클래스는 super type이 존재하지 않음. 완전 별개
  case class LocalResponse(statusCode: Int) // 지역 프로세스내 호출
  case class RemoteResponse(message: String) // 원격 서비스내 호출
  
  sealed trait Computation {
      type Response
      // 수행할 작업은 Future로 감싸짐, 각 작업은 비동기 실행
      val work: Future[Response]
  }

  case class LocalComputation(work: Future[LocalResponse]) extends Computation {
      type Response = LocalResponse
  }

  case class RemoteComputation(work: Future[RemoteResponse]) extends Computation {
      type Response = RemoteResponse
  }
  ```
- `Service` 예시
  ```scala
  object Service {
      // handle: 단일 진입 지점
      // `Await`을 사용하여 `Future`가 완료될때까지 기다림
      def handle(computation: Computation): computation.Response = {
          val duration = Duration(2, SECONDS)
          // 입력으로 들어온 computation에 따라 `LocalResponse/RemoteResponse` 반환
          Await.result(computation.work, duration)
      }
  }

  Service.handle(LocalComputation(Future(LocalResponse(0))))
  Service.handle(RemoteComputation(Future(RemoteResonse("remote call"))))
  ```
- **`handle`이 공통의 super class instance를 반환하지 않음**
  - `LocalResponse`와 `RemoteResponse`는 연관이 없기 때문
- 단, `handle`은 **인자에 의존하는 타입 반환**
  - `RemoteComputation`이 `LocalResponse`를 반환하는 등의 케이스가 존재할 수 없음
  - **타입 검사**에서 걸러짐

## 15.3. 타입 투영
- 타입 안의 (nested) 타입 멤버를 레퍼런싱 하기 위한 문법
- `T#x` 라고 지칭하며, 타입 `T` 안의 `x` 라는 이름의 타입 멤버를 나타냄
- 조금 더 쉬운 예시
  - https://hamait.tistory.com/721
  - <img width="438" alt="image" src="https://user-images.githubusercontent.com/10006290/159693757-af74c228-a128-45e1-b985-aedbc493cf3c.png">
```scala
trait Logger {
    def log(message: String): Unit
}

// 콘솔에 로그를 출력하는 구체적인 Logger
class ConsoleLogger extends Logger {
    def log(message: String): Unit = println(s"log: $message")
}

// Logger에 대한 추상 타입 별명 정의
// 그에 대한 필드를 선언하는 Service trait
trait Service {
    type Log <: Logger
    val logger: Log
}

// ConsoleLogger를 사용하는 구체 서비스
class Service1 extends Service {
    type Log = ConsoleLogger
    val logger: ConsoleLogger = new ConsoleLogger
}
```
- Service1에 정의된 `Log`를 재활용하고 싶다고 가정
  ```scala
  // ERROR, not found: value Service
  val l1: Service.Log = new ConsoleLogger

  // ERROR, not found: value Service1
  val l2: Service1.Log = new ConsoleLogger

  // ERROR, type mismatch
  // found: ConsoleLogger
  // required: Service#Log
  val l3: Service#Log = new ConsoleLogger

  val l4: Service1#Log = new ConsoleLogger
  ```
  - `Service.log`와 `Service1.log`를 사용한다는 것은
    - `Service`나 `Service1`이라는 **객체**를 찾아야함
    - 하지만 해당 **동반 객체**는 존재하지 않음(object)
- 하지만, 원하는 타입 이름을 `#`을 통해 투영(project)할 수 있음
- `l3`는 **타입 검사**를 통과하지 못함
  - `Service.Log`와 `ConsoleLogger`가 모두 `Logger`의 서브타입
    - `Service.Log`는 **추상 타입**
    - 그 타입이 실제로 `ConsoleLogger`의 `super type`인지 아직 알 수 없음
    - 즉, 추상 타입을 마지막으로 **구체적**으로 정의하면서
      - `ConsoleLogger`와 호환되지 않은 다른 `Logger`의 서브타입 지정도 가능하기 때문
- `l4`는 정상 수행
  - **정적인 타입 검사** 통과

#### 타입 지정자(type designator)
- 매일 사용하는 간단한 타입 명세
  - 실제로는 **타입 투영**을 짦게 쓴 것
- 예시
  ```scala
  Int // scala.type#Int
  scala.Int // scala.type#Int
  package pkg {
      class MyClass {
          type t // pkg.Myclass.type#t
      }
  }
  ```

### 15.3.1. 싱글턴 타입
- `object` 키워드로 선언한 **싱글턴 객체**
- `singleton type` 존재
- `AnyRef`의 서브타입인 인스턴스 `v`에는 **모두 고유의 싱글턴 타입**이 존재(`null` 포함)
- `v.type`을 사용하면, 그 타입 사용 가능
- `v.type`은 타입 지정을 `v`가 지정하는 **오직 한 인스턴스**에만 한정함
- 예시
  ```scala
  val s11 = new Service1
  val s12 = new Service1

  val l1: Logger = s11.logger // ConsoleLogger...
  val l2: Logger = s12.logger // ConsoleLogger ...

  val l11: sll.logger.type = s11.logger

  // ERROR, s11.logger와 s12.logger는 호환 x
  val l12: s11.logger.type = s12.logger
  ```
- **싱글턴 객체**는 인스턴스를 하나 정의함과 동시에, `그에 대응하는 타입도 정의`
    ```scala
    // 아래 타입을 인자로 취하는 메서드를 정의하고 싶다면, Foo.type 사용
    case object Foo {
        override def toString = "Foo says Hello!"
    }

    def printFoo(foo: Foo.type) = println(foo)

    printFoo(Foo);
    ```

## 15.4. 값에 대한 타입
- **모든 값에는 타입이 있음**
- **값 타입**(value type)은
  - 이런 타입이 취할 수 있는 **모든 형태**를 의미
- 일반적인 관례로, `AnyVal`의 모든 서브타입을 의미
- **값 타입의 종류**
  - 매개변수화한 타입
  - 싱글턴 타입
  - 타입 투영
  - 타입 지정자
  - 복합 타입
  - 존재 타입
  - 튜플 타입
  - 함수 타입
  - 중위 타입(infix type)

### 15.4.1. 튜플 타입
- `Tuple3[A,B,C]`은 `(A,B,C)`로 표기 가능
  ```scala
  val t1: Tuple3[String, Int, Double]   = ("one", 2, 3.14)
  val t2: (String, Int, Double)          = ("one", 2, 3.14)
  ```

### 15.4.2. 함수 타입
- 화살표 표기 가능
```scala
val f1: Function2[Int, Double, String]  = (i, d) => s"int $i, double $d"
val f2: (Int, Double) => String         = (i, d) => s"int $i, double $d"
```

### 15.4.3. 중위 타입
- 타입 매개변수가 `2개`인 매개변수화한 타입은 **중위 표기법**으로 작성 가능
```scala
val left1: Either[String, Int]  = Left("hello")
val left2: String Either Int    = Left("hello")
val right1: Either[String, Int] = Right(1)
val right2: String Either Int   = Right(2)
```
- 중위 타입을 **내포**시킬수 있음
  - 타입 이름이 `:`로 끝나지 않은, 한 **왼쪽**을 우선으로 결합
  - `:`로 끝나는 타입은 **오른쪽**으로 결합
- 괄호를 사용하여 **기본적인 결합 순서** 변경 가능
  ```scala
  // Either[Either[Int, Double],String]
  val xll1: Int Either Double Either String = Left(Left(1))
  // Either[Either[Int, Double],String]
  val xll2: (Int Either Double) Either String = Left(Left(1))
  
  // Either[Either[Int, Double],String]
  val xlr1: Int Either Double Either String = Left(Right(3.14))
  // Either[Either[Int, Double],String]
  val xlr2: (Int Either Double) Either String = Left(Right(3.14))

  // Either[Either[Int, Double],String]
  val xr1: Int Either Double Either String = Right("foo")
  // Either[Either[Int, Double],String]
  val xr2: (Int Either Double) Either String = Right("foo")

  // Either[Int, Either[Double, String]]
  val xl: Int Either (Double Either String) = Left(1)

  // Either[Int, Either[Double, String]]
  val xrl: Int Either (Double Either String) = Right(Left(3.14))

  // Either[Int, Either[Double, String]]
  val xrr: Int Either (Double Either String) = Right(Right("bar"))
  ```
- Either, Left, Right에 대한 설명
  - https://hamait.tistory.com/649

## 15.5. 고계 타입
```scala
def sum(seq: Seq[Int]]): Int = seq reduce (_ + _)

// 15
sum(Vector(1,2,3,4,5))
```
- 원소 타입을 더 일반화 할 수 있게 하는
  - 타입 클래스(**CHAP.5.4. 타입 클래스 패턴**)의 개념 일반화
  ```scala
  // 덧셈을 추상화한 trait
  trait Add[T] {
    def add(t1: T, t2: T): T
  }

  object Add {
    implicit val addInt = new Add[Int] {
      def add(i1: Int, i2: Int): Int = i1 + i2
    }

    // 1의 trait를 두 Int 받아서 Int의 상을 반환하는 Add라는 암시적 값으로 정의한 동반 객체
    implicit val addIntIntPair = new Add[(Int, Int)] {
      def add(p1: (Int, Int), p2: (Int, Int)): (Int, Int) =
        (p1._1 + p2._1, p1._2 + p2._2)
    }
  }

  // implicitly works as a “compiler for implicits”. We can verify if there is an implicit value of type T. I
  // https://www.baeldung.com/scala/implicitly
  // 맥락 바운드와 implicitly 사용(CHAP.5.1.1), 시퀀스 원소의 합을 구함
  def sumSeq[T: Add](seq: Seq[T]): T = seq reduce (implicitly[Add[T]].add(_,_))
  
  sumSeq(Vector(1 -> 10, 2 -> 20, 3 -> 30)) // 내부적으로 addIntIntPair 호출
  sumSeq(1 to 10) // 내부적으로 addInt 호출

  // ERROR
  sumSeq(Option(2))
  ```
  - **맥락바운드 & implicitly (CHAP.5.1.1, p222)**
    - **매개변수화 한 타입의 암시적 인자가 필요한 특별한 경우**를 짧게 쓸 수 있는 방식
- `sumSeq` 메서드는 암시적인 Add 인스턴스 정의가 있는 모든 시퀀스 **합계**를 낼 수 있음
  - 하지만 `sumSeq` 메서드는 `Seq`의 서브타입만 지원할 뿐
- 좀 더 일반적인 `sum` 구현

#### 고계 타입 예시
- 스칼라는 **고계 타입**(higher-kinded type)
  - 이를 사용하면 **매개변수화한 타입** 또 **추상화**할 수 있음
- 예시 코드
  ```scala
  // reduce를 고계 타입(M[T])로 추상화한 trait
  // M을 사용하는 것은 여러 라이브러리 사용하는 비공식적인 관례
  trait Reduce[T, -M[T]] {
    def reduce(m :M[T])(f: (T, T) => T): T
  }

  // Seq, Option 값을 축약시키는 암시적 인스턴스 정의
  // 단순화를 위해 구현에 각 타입이 이미 제공하는 reduce 메서드 재활용
  object Reduce {
    implicit def seqReduce[T] = new Reduce[T, Seq] {
      def reduce(seq: Seq[T])(f: (T, T) => T): T = seq reduce f
    }

    implicit def optionReduce[T] = new Reduce[T, Option] {
      def reduce(opt: Option[T])(f: (T, T) => T): T = opt reduce f
    }
  }
  ```
  - 공변, 반공변
    - https://edykim.com/ko/post/what-is-coercion-and-anticommunism/
  - `reduce`는 `M[T]`에 대해 **반공변적**
    - 무공변(`-, +`가 없음)으로 만들면 
    - `M[T]`의 `M`이 `Seq`인 암시적 인스턴스는
      - `Vector`와 같은 `Seq`의 서브타입에 대해 사용할 수 없음
  - `reduce`는 `M[T]` 타입의 컨테이너를 **인자**로 전달받음
    - **메서드의 인자는 반공변적**인 위치에 있음
    - `Reduce`가 `M[T]`에 대해 **반공변**이어야 함(CHAP.14.2.2, 하위타입바운드)
  - `seqReduce[T] = new Reduce[T, Seq] {...}`라는 식에서
    - `Seq`는 **타입 매개변수**를 지정하지 않음
    - 타입 매개변수는 `Reduce`의 정의에서 추론됨
- `sum2`를 통한 `Option, Seq` 인스턴스 축약
  ```scala
  // 고계 타입과 동작하는 sum 메서드 정의
  def sum[T: Add, M[T]](container: M[T])(
    implicit red: Reduce[T,M]): T =
      red.reduce(container)(implicitly[Add[T]].add(_,_))
  
  sum(Vector(1 -> 10, 2 -> 20, 3 -> 30))
  sum(1 to 10)
  sum(Option(2))
  
  // ERROR
  sum[Int, Option](None)
  ```
  - `sum(reduce)`를 빈 컨테이너에 적용하는 것은 오류
    - 타입 시그니처를 `sum` 호출에 추가하여 컴파일러에 `None`을 `Option[None]`으로 해석하도록 함
  - 이렇게 하지 않으면, `Option[T]`애 대해 `addInt`, `addIntIntPair`사이의 모호성 해결 불가
    - 컴파일 오류 발생
  - 타입을 명시하면, `None`에 `reduce`를 호출할 수 없다는 **실행 시점 오류**가 발생함
- `sum` 구현은 간단하지 않음
  - `T : Add`라는 **맥락 바운드** 사용
  - `M[T]`에 대해 `M[T]: Reduce`와 같은 맥락바운드를 만들 수는 없음
    - `Reduce`가 타입 매개변수를 두 개 받고,
    - 타입 바운드는, **타입 매개변수**가 하나만 존재할때 사용 가능
    - 따라서 **두번째 인자 목록**을 추가하여, 암시적 `Reduce` 매개변수를 받도록 함
    - 이를 통해 입력 컬렉션에 대해 `reduce`를 호출
- 구현 단순화
  - `Reduce`를 **고계 타입 매개변수**를 하나만 받도록 오버라이딩
  - 그럴경우 **맥락 바운드 타입**을 사용할 수 있음
    ```scala
    // 타입 매개변수 M 하나만 존재
    // 고계 타입 M은 반공변적이지만, 그 안의 타입 매개변수는 여기서 지정하지 않음
    // 이는 존재 타입(CHAP.14.9)
    // 대신 T 매개변수를 reduce 메서드로 이전
    trait Reduce1[-M[_]] {
      def reduce[T](m: M[T])(f: (T, T) => T): T
    }

    // 암시인 seqReduce와 optionReduce를 메서드가 아닌 값으로 정의
    object Reduce1 {
      implicit val seqReduce = new Reduce1[Seq] {
        def reduce[T](seq: Seq[T])(f: (T, T) => T): T = seq reduce f
      }

      implicit val optionReduce = new Reduce1[Option] {
        def reduce[T](opt: Option[T])(f: (T, T) => T): T = opt reduce f
      }
    }
    ```
    - 앞에서는 타입 매개변수 T를 추론할 수 있도록 암시적 메서드를 사용했지만,
      - 이제는 실제 `reduce`를 호출하기 전까지 **타입 추론**을 유예시킬 수 있도록
      - **타입 인스턴스**가 하나만 존재
- 단순화에 따른 `sum`의 구현
  ```scala
  def sum[T : Add, M[_] : Reduce1](container: M[T]): T =
    implicitly[Reduce1[M]].reduce(container)(implicitly[Add[T]].add(_,_))
  ```
  - 이제는 **맥락 바운드**가 두개 존재
    - 하나는 `Reduce1`, 하나는 `Add`
    - `implicitly`에 주어진 **타입 매개변수**가 두 암시적 값 사이의 모호성을 없앰
  - 앞으로 보게될 대부분 **고계 타입 사용법**은
    - 이 예제와 비슷하게 `M[T]`가 아닌 `M[_]`을 사용

#### 고계 타입을 사용해야 할까?
- **추상화**는 한 단계 더 나아갈 수 있음
  - 단, 코드가 복잡해짐
- 고계 타입을 사용해야할까?
  - `Scalaz, Shapeless`와 같은 라이브러리는 **고계 타입**을 폭넓게 활용하여
  - 매우 간결하고, 강력한 코드 조합을 가능하게 됨
  - 하지만 **팀원의 능력**을 고려해야함
    - **추상화가 너무 심해, 활동이 어려운 코드를 만드는 일이 없도록 해야함**

## 15.6. 타입 람다
- `type lambda`, 다른 함수 안에 내포된 함수와 비슷한 개념을
  - **타입 수준**에서 적용한 것
- 맥락에 비해 **너무 많은 타입 매개변수**가 필요한 **매개변수화한 타입**이 필요한 상황에서 유용
- 타입 시스템이 제공하는 구체적인 기능이 아닌,
  - 일종의 **코딩 관용구**
- 예시: `map`의 구현
  ```scala
  // Functor, 맵연산을 제공하는 타입에 널리 사용되는 이름
  // 이 타입은 컬렉션을 메서드의 인자로 넘기지 않음
  // 단, `map2`라는 메서드를 제공하는 `Functor` 클래스로 변환하는 암시적 변환 정의
  // `M[T]`는 반공변일 필요는 없으며, 실제로 공변적으로 정의하는것이 유용
  trait Functor[A, +M[_]] {
    def map2[B](f: A => B): M[B]
  }

  // `Seq`와 `Option`에 대한 암시적 변환 정의
  // 단순화를 위해 각각의 `map`을 활용하여 `map2` 구현
  // Functor가 `M[T]`에 대해 공변적이기 때문에, `Seq`에 대한 암시적 변환을 모든 `Seq`의 서브타입에 사용 가능
  object Functor {
    implicit class SeqFunctor[A](seq: Seq[A]) extends Functor[A, Seq] {
      def map2[B](f: A => B): Seq[B] = seq map f
    }

    implicit class OptionFunctor[A](opt: Option[A]) extends Functor[A, Option] {
      def map2[B](f: A => B): Option[B] = opt map f
    }

    // Map에 대한 변환 정의
    // 타입 매개변수를 2개 받음
    implicit class MapFunctor[K, V1](mapKV1: Map[K, V1])
      // 타입 람다를 사용해서 추가 타입 매개변수 처리
      extends Functor[V1, ({type λ [α] = Map[K, α]})#λ] {
        def map2[V2](f: V1 => V2): Map[K, V2] = mapKV1 map {
          case (k,v) => (k,f(v))
        }
      }
   }
  ```
  - `MapFunctor`에서 `Map`에 대한 매핑이
    - `key`를 동일하게 유지하고
    - `value`만 변경한다고 **결정**함 -> 실상 `mapValues`의 구현
- **타입 람다** 관용구의 구문은, 다소 번잡스러움
- 확장한 이해 코드
  ```scala
  ... Functor[V1,                 // 1
    (                             // 2
      {                           // 3
        type λ [α] = Map[K, α]    // 4
      }                           // 5
    )#λ                           // 6
  ]
  ```
  - 1
    - V1, 타입 매개변수 목록 시작, `Functor`의 두번째 타입 매개변수는 컨테이너 타입
    - 그 컨테이너 타입 자체도, 타입 매개변수를 하나 취함
  - 2, 6
    - 두번째 매개변수 정의 시작
  - 3
    - 구조적 타입 정의(CHAP.14.7)
      - https://twitter.github.io/scala_school/ko/advanced-types.html
      - 해당 타입은 어떤 형태를 유지해야함!(메서드 포함 여부 등)
  - 4
    - `Map`에 대한 별명인 **타입 멤버**를 정의
    - `λ`라는 이름은 임의로 정함(타입 멤버의 경우 항상 해당)
    - `λ`를 사용하는 것이 일반적
      - 이 패턴의 이름도 여기서 온 것
    - 이 타입에는 자체 타입 매개변수인 `α`가 있음
    - 이 경우에는 `Map`의 값 타입을 지정하기 위해 `α`를 사용
  - 5
    - 구조적 타입 종료
  - 6
    - 2에서 시작한 식을 끝냄
    - 구조적 타입에 있는 `λ`에 대한 **타입 투영**을 사용
    - `λ`는 이후에 올 코드에서 추론될 **내포된 타입 매개변수**를 받는 `Map`의 별명
- **타입 람다**는 `Functor`가 지원하지 못하는
  - `Map`에 필요한 **추가 타입 매개변수**를 처리함
- `α`는 이어지는 코드에서 추론
- `λ`나 `α`에 대해 명시적으로 재참조할 필요는 없음
- 스크립트의 동작
  ```scala
  List(1,2,3) map2 (_ * 2) // List(2,4,6)
  Option(2) map2 (_ * 2) // Some(4)
  val m = Map("one" -> 1, "two" -> 2, "three" -> 3)
  m map2 (_ * 2) // Map(one -> 2, two -> 4, three -> 6)
  ```
- 타입 람다 관용구가 자주 필요하지는 않을 것
  - 하지만 여기서 설명한 문제를 해결하고 싶은 경우에 유용할 것

## 15.7. 자기 재귀 타입: F-바운드 다형성
- **F-바운드 다형 타입**(F-bounded polymorphic type)
  - 자기 재귀 타입은 자기 자신을 참조하는 타입을 말함
- 예시: 자바의 Enum 추상 클래스
  ```java
  public abstract class Enum<E extends Enum<E>>
  extends Object
  implements Comparable<E>, Serializable
  ```
  - `Enum<E extends Enum<E>>`
    ```java
    int compare(E obj)
    ```
- 같은 타입에 정의된 열거값 중 하나가 아닌 객체를 `compareTo`에 넘기는 것은 **컴파일 오류**
  ```scala
  TimeUnit.MILLISECONDS compareTo TimeUnit.SECONDS
  Type.HTTP compareTo Type.SOCKS
  TimeUnit.MILLISECONDS compareTo Type.HTTP
  ```
  - 스칼라에서는 **재귀적인 타입**을 사용해서 
    - **반환타입**이 호출한 타입과 **동일한 메서드**를 편하게 정의할 수 있음
  - 타입 계층구조 안에서 이런 타입의 정의 가능
- 예시: `make` 메서드가 정의된 `Parent` 타입의 인스턴스가 아닌, 그 메서드를 호출한 객체와 같은 타입의 인스턴스 반환
  ```scala
  // Parent는 재귀적 타입, 위 java Enum과 같은 역할을 하는 스칼라 구문
  trait Parent[T <: Parent[T]] {
    def make: T
  }
  
  // 파생 타입은 `X extends Parent[X]` 관용구를 따름
  case class Child1(s: String) extends Parent[Child1] {
    def make: Child1 = Child1(s"Child1: make: $s")
  }

  case class Child2(s: String) extends Parent[Child2] {
    def make: Child2 = Child2(s"Child2: make: $s")
  }

  val c1 = Child1("c1") // c1: Child1
  val c2 = Child2("c2") // c2: Child2
  val c11 = c1.make // c11: Child1 = Child1(child1: make: c1)
  val c22 = c2.make // c22: Child2 = Child1(child2: make: c2)

  val p1: Parent[Child1] = c1 // p1: Parent[Child1] = Child1(c1)
  val p2: Parent[Child2] = c2 // p2: Parent[Child2] = Child2(c2)
  val p11 = p1.make // p11: Child1 = Child1(Child1: make: c1)
  val p22 = p2.make // p22: Child2 = Child2(Child2: make: c2)
  ```
  - `p22`는 `Parent`의 참조를 통해 `make`를 호출했으나, `Child2` 타입

## 15.8. 마치며
- 타입 시스템을 극한까지 시험해보는 프로젝트의 좋은 예
  - Shapeless(세이프리스)
  - Scalaz(스칼라제드)
- 타입 시스템에 통달하기 위해 공부해볼만 함
  - 설계 문제를 해결하기 위한 여러 혁신적인 도구 제공
- **타입 시스템을 더 잘 이해하게 되면**
  - **그런 타입 시스템을 채용하는 외부 라이브러리를 활용하기 훨씬 쉬워질 것**