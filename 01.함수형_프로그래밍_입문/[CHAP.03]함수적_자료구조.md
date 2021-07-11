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