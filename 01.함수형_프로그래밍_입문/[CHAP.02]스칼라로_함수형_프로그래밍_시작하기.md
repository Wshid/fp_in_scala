## [CHAP.02] 스칼라로 함수형 프로그래밍 시작하기
## 2.1. 간단한 스칼라 예시
- 코드
  ```scala
  // 단일체 객체 선언, 클래스와 클래스의 유일한 인스턴스 동시 선언
  object MyModule {
    def abs(n: Int): Int = // input Type : Int, return Type : Int
        if (n < 0) -n
        else n
  }

  // private method
  private def formatAbs(x: Int) = {
    val msg = "The absolute value of %d is %d"
    // %d에 해당하는 문자를 , x와 abs(x)로 치환
    msg.format(x, abs(x))
  }

  // unit = void at (java and C)
  def main(args: Array[String]): Unit =
    println(formatAbs(-42))
  ```
- `MyModule`이라는 객체(object)를 선언
  - object를 `module`이라고도 함

### object 키워드
- 새로운 `singleton` 형식을 생성
- `class`와 비슷하되
  - 명명된 인스턴스가 **단 하나**라는 점이 다름
- `java`에서의
  - **익명 클래스**의 새 인스턴스를 생성하는 것과 동일
- `scala`에서는
  - java의 `static` 키워드에 해당되는 내용이 없음
  - `java`에서 **정적 멤버를 가진 클래스**를 사용해야할 상황일 때
    - 스칼라에서 `object`를 사용

### 예제 코드 - `abs`
- 객체나 클래스 안에, `def` 키워드로 정의된
  - 함수 또는 필드를 **메서드**라고 함
- abs 메서드
  - 정수 하나를 받아, 절대값을 돌려주는 **순수 함수**
- 메서드의 `equal` 등호
  - 등호 왼쪽 : left-hand side, signature
  - 등호 오른쪽 : right-hand side, definition
- `return`을 사용하는 것이 아닌,
  - `right-hand-side`의 값이 결국 리턴 값

### 예제 코드 - `formatAbs`
- `private`로 선언되어, 자체 object 내부에서만 사용 가능
- 여기서는 반환 형식을 명시하지 않았으나, 명시해주는 것이 좋음
  - 생략이 가능함
- 메서드 본문이 **여러 줄**로 구성될 경우,
  - 중괄호 쌍으로 감싼다
- block
  - `statement`(문장)들을 담은 중괄호 쌍(`{...}`)
- 문장과 문장은 `new line`이나, 세미 콜론(`;`)으로 구분 된다.
- `val`의 사용
  - 불변(immutable)
  - 값을 재정의할 경우, compile error
- 단순히 `right-hand side`의 경우
  - 하나의 `block`이며
  - 다른 `return` 없이 사용 가능
- `formatAbs`의 메서드 결과는
  - `msg.format(x, abs(x))`

### 예제 코드 - `main`
- 순수 함수적 핵심부 호출하고, 결과를 콘솔에 출력하는 **외부 계층**(`shell`)
- **부수 효과**가 발생함을 강조하기 위해
  - 이런 메서드를 절차(`procedure`)함수 또는, 불순 함수(`impure function`)이라고 함
- `main` 키워드
  - 프로그램 실행 시, scala가 특정한 서명을 가진 `main`을 찾게 됨
  - 반드시 `Array[String]`을 받아, `Unit` 형태로 반환해야 함
- `args`에서는 `command-line` 지정한 인수가 들어옴
  - 현재 이 예시에서는 사용하지 않음
- `Unit`의 사용
  - `main`에서는 딱히 의미 있는 값을 반환하지 않음
  - 특별한 형식으로써 사용 됨
    - 위와 같은 메서드의 반환 형식
  - 값이 단순히 **하나**뿐이며,
    - 값은 `()`라는 **빈 괄호 쌍 리터럴**로 표기하며
    - 유닛 또는 단위로 읽는다.
  - 일반적으로, `반환형식 = Unit`일 경우
    - **그 메서드에 부수효과가 존재**함을 암시함

## 2.2. 스칼라 프로그램의 실행
- 컴파일 후 실행
  - `command line`에서 직접 스칼라 컴파일러 호출
    ```bash
    scalac MyModule.scala
    scala MyModule # 컴파일된 코드 실행 가능
    ```
  - `scalac`를 활용하여, 프로그램을 `java bytecode`로 변환
  - 해당 결과 값으로 `.class`의 파일이 생성 되며,
    - 이 파일들에는 **JVM**이 실행 가능한 컴파일 된 코드 존재
- 간단한 프로그램의 경우, `scalac` 컴파일 없이 실행 가능
  ```bash
  scala MyModule.scala
  ```
  - `Interpreter`에서 적절한 signiture가 있는 `main`을 찾아 실행
- REPL(read-evaluate-print loop)
  - 스칼라 해석기의 대화식 모드
  ```
  scala
  scala> :load MyModule.scala
  ```
  - `:load`
    - 스칼라 소스 파일을 적재하여 해석
    - 패키지 선언이 있는 소스 파일은 제대로 적재하지 못함
  - 개별 줄을 복사하여, `REPL`에 붙여넣을 수 있으며,
    - 여러줄로 된 코드를 붙여넣을 수 있는 `paste mode`로 실행 가능
      - `:paste` 명령으로 실행
  
## 2.3. 모듈, 객체, 이름공간
- `MyModule.abs`
  - `MyModule` 객체 내부 `abs` 참조
- `MyModule`은 `abs`가 속한 `namespace`
- 스칼라의 모든 값은 `object`이며,
  - 각 객체는 0개 이상의 `member`를 가질 수 있음
- 자신의 member들에게, `namespace`를 제공하는게
  - 주된 목적인 객체를 `module`이라고 한다
- member
  - `def` 키워드로 선언된 메서드
  - `val`이나 `object`선언된 또 다른 객체
  - 이외에도 여러 종류의 member가 존재
- 객체 멤버 접근
  - `.`로 접근 가능
- 멤버 구현
  - 객체의 다른 멤버를 한정 없이(앞에 객체 이름을 붙이지 않음) 지정 가능하나,
  - 필요할경우, `this`로 객체 지칭 가능
- 예시
  - `2 + 1` 역시 객체의 맴버 호출
  - `2.+(1)`
    - 2의 `+` 메서드에
    - 1을 **인수**로 전달
- 인수 하나인 메서드의 생략
  - **마침표**와 **괄호**를 생략한 중위(infix) 표기법 사용 가능
  ```scala
  // 아래는 동일한 식
  MyModule.abs(42)
  MyModule abs 42
  ```
- `import` 키워드
  - 객체의 맴버를 현재 위치로 `importing`하게 되면
  - **객체 이름 생략**이 가능
  ```scala
  import MyModule.abs
  abs(-42)
  ```
- 밑줄 표기법 활용
  - 객체의 **모든 비전용(nonprivate) 멤버** 도입 가능
  ```scala
  import MyModule._
  ```

## 2.4. 고차 함수: 함수를 함수에 전달
- 다른 함수를 인수로 받는 함수
  - 고차 함수(higher-order function, HOF; 고계 함수)

### 2.4.1. 함수적으로 루프 작성하기
- 재귀함수 사용
```scala
def factorial(n: Int): Int = {
  def go(n: Int, acc: Int): Int =
    if (n <= 0) acc
    else go(n-1, n*acc)
  go(n, 1)
}
```
- 루프형 보조 함수에는, 보통 `go`나 `loop`같은 이름을 붙이는 것이 관례
- 재귀 호출이 `tail position`에서 일어난다면,
  - `while 루프`를 사용했을 때와 동일한 **바이트 코드**로 컴파일
- 재귀 호출의 **반환 이후**에, 특별히 더 하는 일이 없다면
  - 이런 종류의 최적화(**꼬리 호출 제거(tail call elimination)**)가 적용됨

#### scala의 tail call
- 재귀 호출의 결과를 그대로 돌려주는 것 외에 아무런 일도 하지 않을 때,
  - 그런 호출을 **꼬리 위치**에서 하는 호출
  - **꼬리 호출**이라고 함
- 위 예시에서 `go(n-1, n*acc)`는 꼬리 위치에서 일어남
  - 단, `1 + go(n-1, n*acc)`는 꼬리 호출이 아님
  - `go`의 결과에 다른 어떤일(`+1`)을 수행해야 하기 때문
- 스칼라에서는, 한 함수가 수행하는 모든 재귀 호출이 **꼬리 호출**일 경우
  - 스칼라는 해당 재귀를 매 반복마다 호출 스택을 소비하지 않는 **반복 루프 형태**로 컴파일

### 2.4.2. 첫 번째 고차 함수 작성
- factorial 함수를 사용하는 간단한 예제 프로그램
  ```scala
  object MyModule {
    ... 
    private def formatAbs(x:Int) = { ... }

    private def formatFactorial(n:Int) = {
    ...
    }
  }
  ```
- 위 두 함수는 거의 유사하므로, `formatResult`로 일반화
  ```scala
  def formatResult(name:String, n:Int, f:Int => Int) = { 
    ...
    msg.format(name, n, f(n)) // 고차 함수 f
  }
  ```

#### 고차 함수의 인수
- 고차 함수의 인수에는 `f, g, h` 같은 이름을 사용하는 것이 관례
  - fp에서는 아주 짧은 변수 이름을 사용하는 경향이 있다고 함
  - 심지어 한글자짜리 이름도 많음
- 일반적으로 고차함수가 매우 일반적이기 때문에, 인수가 실제로 **수행하는 일**에 대해 구체적으로 알지 못하기 때문
  - 단순히 인수의 **형식**만 알 뿐

## 2.5. 다형적 함수: 형식에 대한 추상
- 단형적 함수(monomorphic function)
  - 한 형식의 자료에만 작용하는 함수
- 임의의 형식에 대해 동작할 수 있도록 하기

### 2.5.1. 다형적 함수의 예
- 배열에서 한 요소를 찾는 다형적 함수
  ```scala
  def findFirst[A](as : Array[A], p:A => Boolean):Int = {
    @annotation.tailrec
    def loop(n:Int):Int = 
      if (n>= as.length) -1
      else if (p(as(n))) n // p가 현재 요소와 부합한다면, 원하는 요소를 찾을 것
      else loop (n + 1)
    
    loop(0)
  }
  ```
- 형식에 대한 추상(abstracting over the type)
- 다형적 함수를 **하나의 메서드**로서 작성할 때에는
  - **쉼표**로 구분된 형식 매개변수(type parameter)를 대괄호로 감싸고
  - 함수에 이름을 지정하는 형식
- **형식 매개변수**의 이름은 어느것이든 사용 가능
  - 보통은 `[A, B, C]`와 같이, 한 글자짜리 대문자 형식 매개변수를 사용하는 것이 관례
- 형식 매개변수 목록은 **형식 서명**의 나머지 부분에서 참고할 수 있는
  - **형식 변수**(type variable)들을 도입한다

### 2.5.2. 익명 함수로 고차 함수 호출
- **고차 함수**를 호출할 때, 이름 붙은 함수가 아닌
  - **익명 함수**(anonymous function) 또는 **함수 리터럴**(function literal)을 지정해서 호출하는 것이 편리한 경우가 많음
    ```scala
    (x: Int) => x == 9 // 함수 리터럴, 익명 함수
    ```

#### 스칼라에서 값으로서의 함수
- 스칼라에서 **함수 리터럴**을 정의할 때
  - 실제로 정의되는 것은 `apply` 메서드를 가진 하나의 **객체**이다
- `apply` 메서드를 가진 객체는, 그 자체를 **메서드**인 것처럼 호출 가능
  ```scala
  // (a, b) => a < b 의 실제 구현
  val lessThan = new Function2[Int, Int, Boolean] {
    def apply(a:Int, b:Int) = a < b
  }
  ```
- `Function2` 인터페이스(`trait, 특질`)
- `Function1`, `Function3`와 같은 함수들도 **first-class**(일급) 값이라고 부름
- 이 책에서는 **일급 함수**와 **메서드**를 둘 다 **함수**로 지칭
- 일급 함수
  - 함수 자체를 객체로 취급
  - 함수 호출의 인자로 사용될 수 있음
  - 함수의 결과 값으로 활용 가능
  - 대입 연산을 통해, 변수가 가리키는 대상으로 지정 가능

## 2.6. 형식에서 도출된 구현
- **다형적 함수**를 구현할 때는
  - 가능한 구현들의 공간이 **크게 줄어듦**
- 함수가 어떤 형식 `A`에 대해 다형적이면,
  - `A`에 대해서는 
  - 오직 인수들로서 함수에 전달된 연산들(또는 그 연산으로 정의되는 연산들)만 수행할 수 있음
- 다형적 형식에 대해 오직 **단 하나의 구현**만 가능해질 정도로, 가능성의 공간이 축도 되는 경우 존재
- `단 한 가지 방식으로만 구현할 수 있는 함수 서명`의 예시
  - 부분 적용(partial application) - 고차 함수
  - 코드
    ```scala
    // 값 하나와, 인자 두개를 받는 함수를 받음
    // 인수가 하나인 함수를 리턴하는 함수
    // 함수명의 partial은, 함수가 주어진 인수들 모두가 아니라, 일부에만 적용된다는 의미
    def partial1[A,B,C](a: A, f: (A,B) => C):B => C
    ```
    - 이 고차 함수를 구현하려면?
      - 컴파일이 가능한 구현은 **단 한 가지**
      - **함수의 형식 서명**에서 논리적으로 도 출됨
    ```scala
    // B 형식의 인수를 받는 함수 리터럴을, 함수 본문에 추가
    def partial1[A,B,C](a:A, f:(A,B) => C): B => C =
     (b:B) => ??? // B형식의 b를 받는 함수를 돌려줌
    ```
    - C를 얻는 유일한 방법은, A와 B를 f에 넘겨주는 일
    ```scala
    def partial1[A,B,C](a:A, f:(A,B) => C): B => C =
     (b:B) => f(a,b)
    ```
  - 위 예시에서 `b`에 대한 형식은 지정할 필요가 없음
    - 반환 형식을 `B => C`로 명시했기 때문에
    - `b => f(a,b)`로만 명시해도, 스칼라가 문맥상 파악함
  - 스칼라가 **추론할 수 있을 때**에는
    - **함수 리터럴**에서 형식 주해를 생략할 수 있음

### 함수의 커링(currying)
- 인수가 두개인 `f`를 받고, 그것으로 `f`를 부분적용하는 함수로 변환하기
  ```scala
  def curry[A,B,C](f:(A,B) => C):A => (B => C)
  ```

### 함수의 uncurrying
- `curry`의 변환을 역으로 수행
  - `=>`는 오른쪽으로 묶이므로, `A => (B => C)`는 `A=> B => C`와 동일
  ```scala
  def uncurry[A,B,C](f:A => B => C) : (A, B) => C
  ```

### function composition
- 한 함수의 출력을 **다른 함수의 입력**으로 사용
  ```scala
  def compose[A,B,C](f:B => C, g: A => B) : A => C
  ```
- `Function1`(인수 하나를 받는 함수들에 대한 **인터페이스**)의 `compose` 메서드 기본적으로 제공
- `f`와 `g`를 합성하려면, `f compose g`로 표현
- 또한 해당 인터페이스는 `andThen(g compose f)`을 지원
  ```scala
  val f = (x:Double) => math.Pi / 2 - x
  val cos = f andThen math.sin
  ```
- 한 줄 짜리가 아닌, 여러줄을 합성하더라도
  - 위 예제들과 동일함
  - `compose`와 같은 **고차 함수**는, 거대한 함수든 간단한 함수든 구분하지 않음
