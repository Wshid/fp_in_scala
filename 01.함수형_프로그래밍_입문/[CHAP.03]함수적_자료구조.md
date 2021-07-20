# [CHAP.03] 함수적 자료구조
- 함수적 프로그램은
  - 변수를 갱신하거나, mutable 자료구조를 수정하는 일이 없음
  - FP에서 사용할 수 있는 자료구조?
- functional data structure
- 패턴 부합

## 3.1. 함수적 자료구조의 정의
- 함수적 자료구조 : 오직 **순수 함수**로만 조작되는 자료구조
- 그 자리에서 **변경**하거나 기타 **부수 효과**를 수행하는 일이 없음
- 함수적 자료구조는 **정의에 의해** `immutable`함
- 입력 목록은 변하지 않고, 새로운 목록이 만들어짐
- 그런 구조라면 **복사**가 많이 일어나지 않을까?
  - 그러지 않음
- 예시 코드
  ```scala
  sealed trait List[+A] // 형식 A에 대해 매개변수화된 List 자료 형식
  case object Nil extends List[Nothing] // 빈목록

  case class Cons[+A](head: A, tail: List[A]) extends List[A] // 비지 않은 목록을 나타내는 또 다른 자료 생성자
  // tail은 또 다른 List[A]로, Nil | Cons

  object List { // List companion 객체 목록의 생성과 조작을 위한 함수를 담음
    def sum(ints: List[Int]):Int = ints match { // 패턴 부합을 이용해서 목록의 정수들을 합하는 함수
      case Nil => 0
      case Cons(x, xs) => x + sum(xs) // x로 시작하는 목록의 합은, x + 나머지 목록
    }

    def product(ds: List[Double]): Double = ds match {
      case Nil => 1.0
      case Cons(0.0, _) => 0.0
      case Cons(x,xs) => x + product(xs)
    }
    

    def apply[A](as: A+): List[A] = // 가변인수 함수 구문
      if (as.isEmpty) Nil
      else Cons(as.head, apply(as.tail: _*))
    
  }
  ```
  - `sealed trait`
    - 일반적으로 자료형식 도입시, `trait` 키워드 사용
    - 하나의 추상 인터페이스
    - 필요하다면, 일부 메서드의 구현을 담을 수 있음
    - `trait` 앞에 `sealed`를 붙이면,
      - 이 `trait`의 모든 구현이, 이 **파일 안에 선언 되어야 함**
  - `case`로 시작하는 두 줄
    - `List`의 두가지 구현, 두 가지 자료 생성자(data constructor)
    - `List`가 취할 수 있는 두 가지 형태를 나타냄
      - `Nil`과 같은 빈목록 이거나, `Cons`(construct의 줄임말)로 표기되는 비지 않은 목록 일 수 있음
    - 비지 않은 목록은, 초기 요소 `head` 다음에 나머지 요소들을 담은 `List`로 구성
- 함수를 **다형적**으로 만들 수 있듯이, **자료 형식**도 다형적으로 구성 가능
  - `sealed trait List` 다음에 형식 매개변수(`[+A]`)를 두고,
  - `Cons` 자료 생성자 안에서 `A`를 사용함으로써,
    - 목록에 담긴 요소들의 형식에 대해 다형성이 성립함
- 그러면 하나의 정의로 `List[Int]` 및 `List[Double]`등이 가능해짐
- 형식 매개변수의 `+`는 `covariant`를 의미함

### 공변과 불변에 대해
- 가변 지정자(variance annotation)
- positive 매개변수
- `extends`와 같은, 상/하위 형식을 의미할때 사용
- `+`가 없다면, 해당 형식 매개변수에 대해 불변을 의미
- 위 예시에서는 `Nil`이 `List[Nothing]`을 `extends`하는 형태
  - `Nothing`은 모든 형식의 하위 형식이고, `A`가 공변이므로
  - `Nil`을 `List[Int]`와 같이 그 어떤 형식의 목록으로 간주 가능

### 자료 생성자의 예시
```scala
// Nil이지만, `List[Double]`로 인스턴스화 가능
// 빈 목록에는 아무런 요소가 없으므로, 어떤 형식의 요소로 간주할 수 없기 때문
val ex1: List[Double] = Nil 
val ex2: List[Int] = Cons(1, Nil)
val ex3: List[String] = Cons("a", Cons("b", Nil))
```

## 3.2. 패턴 부합
- `List`의 동반 객체(`companion object`)라고 부름
  - 예시 중 `sum`, `product`
  - 코드
    ```scala
    def sum(ints: List[Int]):Int = ints match { // 패턴 부합을 이용해서 목록의 정수들을 합하는 함수
      case Nil => 0
      case Cons(x, xs) => x + sum(xs) // x로 시작하는 목록의 합은, x + 나머지 목록
    }

    def product(ds: List[Double]): Double = ds match {
      case Nil => 1.0
      case Cons(0.0, _) => 0.0
      case Cons(x,xs) => x + product(xs)
    }
    ```
- 네이밍
  - 목록 같은 순차열(`sequence`)를 나타내는 변수의 이름으로는
    - `xs, ys, as, bs`등을 사용하고
    - 순차열의 개별 요소를 나타내는 변수명으로 `x, y, z, a, b` 등을 사용하는 것이 관례
  - 첫 요소는 `h`(head)
  - 마지막 요소는 `t`(tail)
  - 목록 전체는 `l`(list)
- `List`와 같은 재귀적인 자료 형식(`Cons` 자료 생성자에서 자신을 재귀적으로 참조함을 주목)을
  - 다루는 함수들을 작성할 때는
  - 이처럼 재귀적인 정의를 사용하는 경우가 많음

### 스칼라의 동반 객체
- `자료 형식`과 `자료 생성자`와 함게
  - **동반 객체**를 선언하는 경우가 많음
- **동반 객체**는 자료형식과 같은 이름의 `object`로
  - 자료 형식의 값들을 생성하거나,
  - 조작하는 여러 **편의용 함수**를 담는 목적으로 쓰임
- 예시
  - 요소 `a`의 복사본 `n`개로 이루어진 `List`를 생성하는 함수
    ```scala
    def fill[A](n: Int, a: A):List[A]
    ```
  - 이 모듈이 목록을 다루는데 관련된 함수를 담고 있음이, 보다 명확하기 때문

### 패턴 부합과 switch
- **패턴 부합**은 표현식의 구조를 따라 내려가면서
  - 그 구조의 부분 표현식을 추출하는 **복잡합 switch**구만과 비슷하게 동작
- 패턴 부합은 `ds`같은 표현식으로 시작하여
  - `match` 키워드가 오고
  - `case`구문과 그에 해당하는 결과가 작성됨
- 매칭되는 대상이 여러 패턴에 해당된다면
  - **처음으로 부합한 구문**을 선택함
- 예시
  ```scala
  // List(1,2,3) => 1
  List(1,2,3) match {
    case Cons(h, _) => h
  }

  // List(1,2,3) => 3
  List(1,2,3) match {
    case Cons(_, t) => t
  }

  // List(1,2,3) => MatchError, 부합하는 것이 하나도 없음
  List(1,2,3) match {
    case Nil => 42
  }
  ```
- 패턴의 변수들을 대상의 **부분 표현식**들에 배정한 결과가
  - 대상과 **구조적으로 동등**(strictirally equivalent)하다면 패턴은 대상과 **부합**

### 스칼라의 가변 인수 함수
- `object List`의 `apply` 함수는 **가변 인수 함수**(variadic function)
- 이 함수가 `A` 형식의 인수를 `0`개 이상 받음(즉, 하나도 받지 않거나, 하나 또는 여러개를 받음) 수 있음을 뜻함
- 코드
  ```scala
  def apply[A](as: A+): List[A] = // 가변인수 함수 구문
      if (as.isEmpty) Nil
      else Cons(as.head, apply(as.tail: _*))
  ```
- 자료 형식을 만들 때에는,
  - 자료 형식의 **인스턴스**를 편리하게 생성하기 위해
  - 가변 인수 `apply` 메서드를 자료 형식의 **동반 객체**에 집어넣는 관례가 흔히 쓰임
- 그런 생성 함수의 이름을 `apply`로 지정후 동반객체에 두면
  - **목록 리터럴**(list literal)또는 **리터럴 구문** 으로 함수 호출이 가능
    - `e.g. List(1,2,3,4), List("hi", "bye")`
- 가변인수 함수는 요소들의 **순차열**을 나타내는 `Seq`를 생성하고 전달하기 위한 구문
- `Seq`는 스칼라의 컬렉션 라이브러리에 있는 하나의 인터페이스로
  - **목록**이나 **대기열**, **백터**와 같은 순차열 비슷한 자료구조들이 구현되도록 마련
- `apply` 안에서 인수 `as`는 `Seq[A]`에 묶인다
- `Seq[A]`는 `head`, `tail`
- 특별한 형식 주해인 `_*`s는, `Seq`를 **가변 인수 메서드**에 전달할 수 있도록 함

## 3.3. 함수적 자료구조의 자료 공유
- 자료가 **불변**이라면,
  - 목록에 요소를 추가하거나, 제거하려면?
- 추가시,
  - 기존 목록 앞에 `1`이라는 원소를 추가하려면
  - `Cons(1, xs)`라는 새 목록을 만들면 됨
  - 목록은 불변이므로, `xs`를 **복사할 필요는 없음** => **재사용**
  - 이를 **자료 공유**(data sharing)이라고 함
- **불변인 자료**를 공유하면, **함수**를 좀 더 효율적으로 활용 가능
- 이후의 코드가 **지금** 이 자료를 언제 어떻게 수정할지 걱정 x
  - 항상 **불변인 자료구조**를 반환하면 됨
  - 자료가 변하거나 깨지지 않도록 **방어적 복사**가 필요 없음
- 제거시,
  - 목록의 꼬리인 `xs`를 돌려주면 됨
  - **실질적인 제거**는 일어나지 않음
- 원래의 목록은 여전히 **사용 가능한 상태**
- 이를 두고 **함수적 자료 구조**는 **영속적**(persistent)라고 함
  - 자료구조에 연산이 가해져도,
    - **기존의 참조가 결코 변하지 않음**

### 3.3.1. 자료 공유의 효율성
- 예시
  ```scala
  def append[A](a1: List[A], a2: List[A]) : List[A] =
    a1 match {
      case Nil => a2
      case Cons(h,t) => Cons(h, append(t, a2))
    }
  ```
  - 오직 **첫 목록**이 다 소진될 때까지만, **값들을 복사**
  - 따라서 이 함수의 **실행 시간**과 **메모리 사용량**은 `a1의 길이에만 의존`
  - 목록의 나머지는 그냥 `a2`를 가리키는 상태
- 만약 위 함수를 **배열 두개**로 구현한다면,
  - 두 배열의 모든 요소를 **결과 배열에 복사**해야 함
  - 이 경우, **불변**이 **연결 목록의 배열**보다 훨씬 효율적
- **단일 연결 목록**의 구조 때문에
  - `Cons`의 `tail`을 치환할 때마다
  - 반드시 이전의 모든 `Cons` 객체를 복사해야함(`tail`이 목록의 마지막 `Cons`임에도 해야함)
- **순수 함수적 자료구조**를 작성할 때
  - **자료 공유**를 현명하게 사용해야 함
- scala library의 `Vector`

### 3.3.2. 고차 함수를 위한 형식 추론 개선
- 예시
  ```scala
  def dropWhile[A](l: List[A], f: A => Boolean): List[A]

  val xs: List[Int] = List(1,2,3,4,5)
  val ex1 = dropWhile(xs, (x: Int) => x < 4) // x의 타입을 명시해야 함
  ```
- 스칼라가 추론할 수 있도록 변경
  ```scala
  def dropWhile[A](as: List[A])(f: A => Boolean): List[A] = 
    as match {
      case Cons(h,t) if f(h) => dropWhile(t)(f)
      case _ => as
    }
  
  val xs: List[Int] = List(1,2,3,4,5)
  val ex1 = dropWhile(xs)(x => x < 4) // x의 타입을 따로 명시하지 않음
  ```
  - `dropWhile(xs)(f)` 형태
  - `dropWhile(xs)`는 하나의 함수를 돌려주며,
    - 그 함수를 인수 `f`로 호출함
    - `dropWhile`이 **currying**된 상황
  - 인수들을 이런식으로 **묶게 되면**
    - 스칼라의 **형식 추론**을 도울 수 있음
- 조금 더 일반화하면,
  - **함수 정의**에 **여러 개의 인수 그룹**이 존재하는 경우,
  - **형식 정보**는 그 **인수 그룹**들을 따라 **왼쪽 -> 오른쪽**으로 흘러감
- 예시에서는
  - **첫 인수그룹**은 `dropWhile`의 형식 매개변수 `A`를 `Int`로 고정시키므로
  - `x => x < 4`에 대한 **형식 주해가 필요하지 않음**
- 이처럼
  - **형식 추론**이 최대로 일어나도록
  - **함수 인수**들을 적절한 순서의 **여러 인수 목록**들로 묶는 경우가 많음

## 3.4. 목록에 대한 재귀와 고차 함수로의 일반화
- `sum`과 `product`의 구현
  ```scala

  def sum(ints: List[Int]):Int = ints match {
    case Nil => 0
    case Cons(x, xs) => x + sum(xs)
  }

  def product(ds: List[Double]) : Double = ds match {
    case Nil => 1.0
    case Cons(x, xs) => x * product(xs)
  }
  ```
  - `product`의 경우 `0.0` 점검을 위한 `short-circuiting` 논리를 포함하지 않음
  - 정의는 매우 비슷함
- `+, *` 연산 및 `0, 1.0`인 부분만 제외하면, 통합이 가능함
- 개선 예시
  ```scala
  def foldRight[A,B](as:List[A], z:B)(f:(A,B) => B) : B = {
    as match {
      case Nil => z
      case Cons(x, xs) => f(x, foldRight(xs, z)(f))
    }
  }

  def sum2(ns: List[Int]) = foldRight(ns, 0)((x,y) => x+ y)
  def product2(ns: List[Double]) = foldRight(ns, 1.0)(_ * _) // `(x, y) => x * y` 를 간결하게 표시
  ```

#### 익명 함수를 위한 밑줄 표기법
- 스칼라가 `x, y`의 형식을 추론할 수 있다면
  - `(x, y) => x + y`를 `_ + _`로 표기 가능
- 이는 **함수 매개변수**들이, 본문 내에서 **한 번씩만 언급 될 떄** 유용함
- 예시
  ```scala
  _ + _ // (x, y) => x + y
  _ * 2 // x => x * 2
  _.head // xs => xs.head
  _ drop _ // (xs, n) =>  xs.drop(n)
  ```
- 단, 적당히 사용해야 함
- 스칼라에서는 **밑줄 기반 익명 함수**의 범위에 적용하는 **정밀한 규칙**이 있으나,
  - 그런 규칙을 따질 정도라면, 그냥 보통의 **이름 붙인 매개변수**를 사용하는게 좋음

#### foldRight의 특징
- 하나의 **요소 형식**에만 특화되지 않음
- 함수가 돌려주는 값이 **목록의 요소**와 같은 형식일 필요도 없음
- 원리
  ```scala
  Cons(1, Cons(2, Nil))
  // 위 식을, 아래와 같이 변경
  f(1, f(2,z))
  ```
- 구체적인 예시
  ```scala
  foldRight(Cons(1, Cons(2, Cons(3, Nil))), 0)((x,y) => x + y)

  1 + foldRight(Cons(2, Cons(3, Nil)), 0)((x,y) => x + y)
  1 + (2 + foldRight(Cons(3, Nil), 0)((x,y) => x + y))
  1 + (2 + (3 + (foldRight(Nil, 0)((x,y) => x + y))))
  6
  ```
- `foldRight`가 하나로 축약(`collapsing`)되려면,
  - 반드시 몰고의 끝까지 `traversal` 해야 함

### 3.4.1. 그 외의 목록 조작 함수들
#### 표준 라이브러리의 목록들
- `::`은 **오른쪽으로 연관됨**
  - 스칼라에서 이름이 `:`로 끝나는 모든 메서드는, 오른쪽으로 연관
  - `x :: xs`는 `xs.::(x)`와 동일하며, `::(x,xs)` 자료생성자를 호출
- `1 :: 2 :: Nil`은 `(1 :: (2 :: Nil))`과 동일
  - 이는 `List(1,2)`와 같음
- `case Cons(h, t)`는 `case h :: t`와 같음
  - 여러 요소를 아우르기 위해, 괄호가 필요하지 않음(`case h :: h2 :: t`)
- 유용한 함수
  ```scala
  def take(n: Int): List[A]
  def takeWhile(f:A => Boolean): List[A]
  def forAll(f:A => Boolean): Boolean
  def exists(f: A => Boolean): Boolean
  scanLeft , scanRight // foldLeft, foldRight와 유사하나, 최종적으로 누적된 값만이 아닌, 부분결과의 List 반환
  ```

### 3.4.2. 단순 구성요소들로 목록 함수를 조립할 때의 효율성 손실
- `List`의 문제는
  - 범용적인 함수로 표현은 가능하나,
  - 그 결과로 만들어진 구현이 **항상 효율적이지 않음**
- 같은 입력을 **여러번 읽는** 구현이 만들어질 수 있으며,
  - 이른 종료를 위해서는 **명시적인 재귀 루프**를 작성해야 할 수 있음