# [CHAP.05] 엄격성과 나태성
- **CHAP.03**에서 **단일 연결 목록**의 예로
  - 순수 함수적 자료구조를 논의
- `map, filter, foldLeft, foldRight, zipWith` 등
  - 목록에 대한 일괄 연산 진행
- 이 연산들이 모두 
  - 주어진 **입력 목록 전체**를 훑고
  - 새로운 목록을 **연산의 결과**로 돌려줌
- 한번만 순회하기 => `traversal`
- 예시
  - 놀이용 카드 한 벌에서, 홀수 카드를 모두 제거하고, 퀸 카드를 뒤집기
  - 스칼라는 후자의 방식으로 수행(홀수 카드 모두 제거 후, 남은 카드에서 퀸을 찾음)
    ```scala
    List(1,2,3,4).map(_ + 10).filter(_ % 2 == 0).map(_ * 3)
    ```
    - 각 변환은 자신만의 새로운 목록을 생성함, 그리고 다음 변환의 입력으로만 쓰인후 폐기됨
      - `map(_ + 10)`은 그 자체의 새로운 목록을 생성
      - 이후 `filter`로 넘긴 후, 폐기
- 비엄격성(`non-strictness`), 나태성(`laziness`)를 이용하면
  - 자동적인 **루프 융합**이 가능해짐
- 비엄격성이 **함수적 프로그램**의 **효율성**과 **모듈성**을 개선하는 근본적인 기법

## 5.1. 엄격한 함수와 엄격하지 않은 함수
- **비엄격성**은 함수의 한 속성
- `함수가 엄격하지 않다`
  - 그 함수가 하나 이상의 인수들을 **평가하지 않을 수 있음**
- `함수가 엄격하다`
  - 자신의 **인수**들을 항상 **평가**
  - 대부분의 언어는, 인수들을 모두 평가하는 함수만 지원
  - 스칼라에서도 특별히 다르게 지정하지 않는 한, **모든 함수 정의는 엄격한 함수**
- 예시
  ```scala
  def square(x: Double): Double = x * x

  // 정상 호출
  square(41.0 + 1.0)

  // 비정상 호출, 표현식이 square 본문에 진입하기 전에 평가
  square(sys.error("failure"))
  ```
- 스칼라를 비롯한 여러 프로그래밍 언어에서 `||`, `&&`는 **비엄격**
  - 인수의 평가가 생략될 수도 있는 함수로 생각 가능
- `&&`함수는 `Boolean`인수 두 개를 받되,
  - 첫번째 인수가 `true`일때만 둘째 인수를 평가
- 스칼라 `if 제어 구조`의 비엄격성 예시
  ```scala
  val result = if (input.isEmpty) sys.error("empty input") else input
  ```
  - if의 경우 내장 언어 구조이긴 함
  - `if`함수는 자신의 **모든 인수를 평가하지 않음**적인 측면에서 **비엄격 함수**
  - `if` 함수는 **조건 매개변수**에 대해서는 엄격함
    - 참/거짓에 따른 두 분기를 선택하기 위해, 조건이 평가되어야 하기 때문
  - 단, `true/false` 분기에 대해서는 엄격하지 않음
    - 둘 중 하나만 조건에 따라 평가되기 때문 
- 인수 중, 일부가 평가되지 않아도 호출이 성립하는 비엄격 함수
  - 비엄격 `if`함수 예시
    ```scala
    def if2[A](cond:Boolean, onTrue: () => A, onFalse: () => A):A = 
        if (cond) onTrue() else onFalse()
    
    if2(a <22,
        () => println("a"), // () => A 를 생성하는 리터럴 구문
        () => println("b")
    )
    ```
    - `평가되지 않을 채 전달될 인수`에는 해당 형식 바로 앞에 `() =>`를 표기해준다
      - `() => A`는 인수를 받지 않고 `A`를 돌려주는 함수
    - 일반적으로 표현식의 **평가되지 않은 상태**를 `thunk`라고 함
    - 나중에 그 `thunk`의 표현식을 평가하여, 결과를 내보내도록 **강제** 가능
      - `onTrue`나 `onFalse`처럼 **빈 인수 목록**을 지정하여 함수를 호출하면 됨
    - `if2`의 호출자는 `thunk`를 명시적으로 생성해야 함
      - 함수 리터럴 구문의 것과 동일한 관례를 따름
- 비엄격 `if`를 더 깔끔하게 표현하기
  ```scala
  def if2[A](cond:Boolean, onTrue: => A, onFalse => A):A =
    if (cond) onTrue else onFalse

  if2(false, sys.error("fail"), 3)
  ```
  - 평가되지 않은 채로 전달할 인수에는, 그 형식 앞에 `=>`만 붙이는 형태
  - `=>`는 함수 본문에서 지정된 인수를 평가하는데 **어떤 특별한 구문도 필요하지 않음**
    - 평소대로 식별자만 참조하면 됨
  - 함수를 호출할때도, 별다른 구문이 필요하지 않으며
    - 보통의 함수 구문만 사용하면 `thunk`내의 표현식을 알아서 감싸줌
- 평가되지 않은 채로 전달되는 인수는
  - 함수의 본문에서 **참조된 장소마다 한 번씩 평가됨**
  - 스칼라는 인수 평가의 결과를 기본적으로 **캐싱하지 않음**
    ```scala
    def maybeTwice(b:Boolean, i: => Int) = if (b) i+i else 0
    val x = maybeTwice(true, { println("hi"); 1+41})
    // "hi"가 두 번 호출됨, 84 반환

    // 캐싱이 적용된, lazy하게 평가하기
    def maybeTwice2(b: Boolean, i:=> Int) = {
        lazy val j = i
        if (b) j+j else 0
    }
    val x = maybeTwice2(true, { println("hi"); 1+41})
    // "hi"가 한번 호출됨, 84 반환
    ```
- `lazy val`을 사용하게 되면,
  - 우변의 평가를, 우변이 **처음 참조될 때까지 지연**
  - **평가 결과를 캐시에** 담아주고, 이후 참조에서는 **평가를 되풀이 하지 않음**
- 스칼라에서의 **비엄격 함수**의 인수는
  - **값**(by value)으로 전달되는 것이 아닌 **이름**(by name)으로 전달 됨

### 엄격성의 공식적인 정의
- `어떤 표현식의 평가가 무한정 실행`되거나, `한정된 값을 돌려주지 않고 오류`를 던진다면,
  - 그러한 표현식을 `terminate`하지 않는 표현식 또는 `bottom`으로 평가되는 표현식이라 부름
- 만일 **바닥**으로 평가되는 모든 `x`에 대해 표현식 `f(x)`가 바닥으로 평가되면,
  - 그러한 함수 `f`는 **엄격한** 함수이다

## 5.2. 확장 예제: 게으른 목록
- 스칼라에서 나태성을 사용하는 예시
  - 함수적 프로그램의 **효율성**과 **모듈성**을 `lazy list` 또는 `stream`을 이용하여 계산

### Stream의 간단한 정의
```scala
sealed trait Stream[+A]
case object Empty extends Stream[Nothing]
case class Cons[+A](h:() => A, t:() => Stream[A]) extends Stream[A]
// 비지 않은 스트림은 하나의 머리와, 꼬리로 구성
// 둘 다 엄격하지 않은 값이며,
// 기술적인 한계로, 이는 이름으로 전달되는 인수가 아닌, 반드시 명시적으로 강제해아하는 성크

object Stream {
  // 비지 않은 스트림의 생성을 위한 똑똑한 생성자
  def cons[A](hd: => A, tl: => Stream[A]): Stream[A] = {
    lazy val head = hd // 평가 반복을 피하기 위해 head, tail을 값 파싱
    lazy val tail = tl
    Cons(() => head, () => tail)
  }
  def empty[A]: Stream[A] = Empty // 특정 형식의 빈 스트림을 생성하기 위한 똒똑한 생성자

  // 여러 요소로 이루어진 Stream의 생성을 위한 편의용 `가변 인수 메서드`
  def apply[A](as: A*): Stream[A] =
    if (as.isEmpty) empty else cons(as.head, apply(as.tail: _*))
}
```
- `List`와 유사하나, `Cons`의 자료 생성자가
  - 보통의 엄격한 값이 아닌, `thunk`(`() => A`와 `() => Stream[A]`)를 받는다는 것이 다름
  - `Stream`을 순회하려면, `if2`의 정의와 마찬가지로
    - `thunk`의 평가를 강제해야 함
- 예시 : `Stream`에서 head를 추출하기
  ```scala
  def headOption: Option[A] = this match {
    case Empty => None
    case Cons(h,t) => Some(h()) // h()를 통해, thunk h를 명시적으로 강제
  }
  ```
  - 강제하는 부분외, `List`와 동일하게 동작
  - 실제 요구된 부분만 평가(Cons의 꼬리는 평가하지 않음)하므로, `Stream`의 이 능력은 유용

### 5.2.1. 스트림의 메모화를 통한 재계산 피하기
- `Cons` 노드가 강제 되었다면, 그 값을 **캐싱**해두는 것이 바람직
- 다음과 같이 `Cons` 자료 생성자를 직접 사용한다면, `expensive(x)`가 두번 계산 됨
  ```scala
  val x = Cons(() => expensive(x), tl)
  val h1 = x.headOption
  val h2 = x.headOption
  ```
- 일반적으로 이런 문제는 추가적인 `invariant`(불변식)을 보장하거나
- **패턴 부합**에 쓰이는 생성자와는 조금 다른 **서명**을 제공하는
  - 자료 형식을 생성하는 함수인 똑똑한 생성자(smart)를 이용하여 피함
- **똑똑한 생성자**의 이름으로는
  - 해당 자료 생성자의 **첫 글자를 소문자로 바꾼 것을 사용하는 것이 관례**
- 위 `cons`는 `Cons`의 머리와 꼬리를 이름으로 전달 받아, `memoization`을 수행
  - 이는 흔히 쓰이는 기법으로,
  - 이렇게 하면 `thunk`는 오직 한번만(`처음으로 강제될 때`) 평가되고
  - 이후의 강제에서는 캐싱된 `lazy val`이 반환
- `cons` 예시
  ```scala
  def cons[A](hd: =>A, tl: => Stream[A]): Stream[A] = {
    lazy val head = hd
    lazy val tail = tl
    Cons(() => head, () => tail)
  }
  ```
- empty **똑똑한 생성자**는 `Empty`를 그냥 돌려주나,
  - `Empty`의 형식이 `Stream[A]`로 지정되어 있음
  - 경우에 따라 이 형식이 **형식 추론**에 더 적합함
- `apply` 예시
  ```scala
  def apply[A](as: A*): Stream[A] =
    if (as.isEmpty) empty
    else cons(as.head, apply(as.tail: _*))
  ```
- 인수들을 `cons`안에서 `thunk`로 감싸는 작업은 `scala`가 내부적으로 처리
- `as.head`와 `apply(as.tail: _*)` 표현식은
  - `Stream`이 강제할 때까지는 평가되지 않음