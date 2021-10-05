# [CHAP.06] 순수 함수적 상태
- `state`를 다루는 순수 함수적 프로그램 작성 방법
  - 난수 발생 분야의 예시를 통해 확인
- 임의의 상태 있는(stateful) API를 **순수 함수적**으로 만드는 데 쓰이는 기본 패턴을 익히기

## 6.1. 부후 효과를 이용한 난수 발생
- `scala`에서 난수 생성시 `scala.util.Random` 클래스를 이용
- 위 클래스는 **부수 효과**에 의존하는 상당히 전형적인 **명령식**(imperastive) API를 제공
- 예시
  ```scala
  // 시스템 시간을 seed로 하여 새 난수 발생기 생성
  val rng = new scala.util.Random
  rng.nextDouble
  rng.nextInt
  rng.nextInt(10) // 0 ~ 9 이하의 정수 난수
  ```
- 상태 갱신은 **부수 효과**로서 수행되므로
  - 이 메서서들은 **참조**에 투명하지 않음
- 참조에 투명하지 않은 함수는 `그렇지 않은 함수`에 비해
  - `검사, 합성, 모듈화가 어려움`
  - `병렬화 어려움`
- **검사성**을 확인해보기
- **무작위성**을 활용하는 메서드를 작성할 때는
  - 그런 메서드의 **재현성**(repoducibility)를 검사할 필요가 있음
- 예시 : 육면체 주사위, `1~6`의 정수를 돌려주는 메서드
  ```scala
  def rollDie: Int = {
    val rng = new scala.util.Random
    rng.nextInt(6) // 0 ~ 5
  }
  ```
  - `off-by-one error`가 존재
  - `1 ~ 6`의 값을 돌려주어야 하나, 실제로는 `0 ~ 5`의 값을 반환
  - 명세대로 작동하지 않지만, `5/6`의 확률로 통과 함
  - 검사에 **실패**했을 때는, 실패 상황을 신뢰성 있게 재현할 수 있다면 **이상적**
- 위와 같은 간단한 예시가 아닌, 메서드가 훨씬 복잡하고, 버그가 훨씬 미묘한 상황도 얼마든지 상상 가능
- 위 예시의 해결책은, 난수 발생기를 인수로 전달하는 방법
  - 그렇게 되면, **실패한 검사 재현**시, 그 때 쓰였던 동일한 **난수 발생기**를 전달하면 됨
    ```scala
    val rollDie(rng: scal.util.Random):Int = rng.nextInt(6)
    ```
  - 단 **동일한** 발생기의 경우 `seed`와 기타 **내부 상태**가 동일해야 함
    - **상태가 동일**하다는 것은
      - 발생기를 만든 이후, 그 메서드들이 **원래의 발생기 메서드 호출 횟수**와 동일하게 호출되었음을 의미
    - 하지만 이를 보장하기는 매우 어려움
      - `nextInt`를 호출할 때마다, 난수 발생기의 **이전 상태가 파괴됨**
    - `Random` 메서드가 몇 번 호출되었는지 추적하는 **개별적인 매커니즘**을 마련해야 할까?
      - 그렇지 않음, 단 **부수 효과를 피해야하 함**

## 6.2. 순수 함수적 난수 발생
- **참조 투명성**을 되찾는 관건은
  - **상태 갱신**을 **명시적**으로 드러냄
- 상태를 **부수 효과**로서 갱신하지 말고
  - 그냥 새 상태를 **발생한 난수**와 함께 돌려주면 됨
- 예시 : 난수 발생기의 인터페이스
  ```scala
  trait RNG {
    def nextInt: (Int, RNG)
  }
  ``` 
  - 위 메서드는 무작위 `Int`를 생성해야 함
  - 이 `nextInt`를 이용해서, 다른 함수도 정의해야 함
  - 발생한 난수만 돌려주고, 내부 상태는 **제자리**(in-place)변이를 통해서 갱신하는 대신,
    - 이 인터페이스는 **난수**와 **새 상태**를 돌려주고, 기존 상태는 수정하지 않음
- 다음 상태를 **계산**하는 관심사와
  - 새 상태를 프로그램 나머지 부분에 **알려주는** 관심사를 분리함
- **전역 변이 가능 메모리**는 전혀 쓰이지 않음
- 그냥 다음 상태를 호출자에게 돌려줄 뿐
- 이 API의 사용자가
  - **난수 발생기의 자체 구현**에 대해서는 아무것도 모른다는 점에서,
  - 여전히 발생기 안에 **캡슐화**되어 있음을 의미
- 난수 발생기의 구현 방법
  - 기존에는 여러 방법이 있으나,
  - `scala.util.Random`과 같은 알고리즘을 사용하는 간단한 난수 발생기
  - 이는 **선형 합동 발생기**(Linear congruential generator)
  - `nextInt`가 발생된 난수와, 새 `RNG`객체를 돌려줌
    - `RNG`객체 : 다음 값을 생성하는데 사용

### CODE.6.2. 순수 함수적 난수 발생기
- 코드
  ```scala
  case class SimpleRNG(seed: Long) extends RNG {
    def nextInt: (Int, RNG) = {
      val newSeed = (seed * 0x5DEECE66DL + 0xBL) & 0xFFFFFFFFFFFFL // 비트곱 논리 연산(&), 현재 seed를 기준으로 새로운 seed 생성
      val nextRNG = SimpleRNG(newSeed) // 다음 상태(새 seed로 생성한 RNG Instance)
      val n (newSeed >>> 16).nextInt // >>>는 빈자리를 0으로 채우는, 이진 오른쪽 자리 이동. n은 새 의사난수 정수
      (n, nextRNG) // 반환값은 의사난수 정수와 다음 발생기 상태를 담은 튜플
    }
  }
  ```
- `SimpleRNG` 실행 예시
  ```scala
  val rng = SimpleRNG(42) // 임의의 seed=42
  val (n1, rng2) = rng.nextInt
  val (n2, rng3) = rng2.nextInt
  ```
  - 위 예시를 여러번 되풀이해서 실행해도 **항상 같은 값**을 반환
    - 이 API는 **순수**하다

## 6.3. 상태 있는 API를 순수하게 만들기
- 순수 함수를 사용하여 다음 상태를 갱신하면, **효율성**이 손실됨
  - 자료를 제자리에서 변이할 수 없기 때문
- 순수함수적 자료구조를 효율적으로 사용한다면
  - 그런 효율성 손실을 줄일 수 있음
- 때에 따라 자료를 **제자리 변이**하더라도 **참조 투명성**이 깨지지 않는 경우도 존재
  - 4부에서 다를 내용
- 예시 클래스
  ```scala
  class Foo {
    private var s: FooState = ...
    def bar: Bar
    def baz: Int
  }
  ```
- `bar`, `baz`가 각각 나름의 방식대로 `s`를 변이
- **한 상태**에서 **다음 상태**로의 **전이**를 명시적으로 드러내는 과정을
  - 기계적으로 진행해서, 이 API를 **순수 함수적 API**로 변환할 수 있음
  ```scala
  trait Foo {
    def bar: (Bar, Foo)
    def baz: (Int, Foo)
  }
  ```
- 위 패턴을 적용하는 것은
  - 계산된 **다음 상태의 프로그램**의 나머지 부분에 전달하는 책임을
    - **호출자**에게 지우는 것에 해당
  - 앞에서 본 순수 `RNG Interface`는
    - 만일 이전의 `RNG`를 재사용한다면, 이전에 발생한 것과 같은 값을 반환
- 다음 코드에서 `i1 = i2`를 만족
  ```scala
  def randomPair(rng: RNG): (Int, Int) = {
    val (i1,_) = rng.nextInt
    val (i2,_) = rng.nextInt
    (i1, i2)
  }
  ```
- 서로 다른 두 수를 만들려면
  - 첫 `nextInt`가 돌려준 `RNG`를 이용하여, 둘째 `Int`를 발생시켜야 함
  ```scala
  def randomPair(rng: RNG): ((Int, Int), RNG) = {
    val (i1, rng2) = rng.nextInt
    val (i2, rng3) = rng2.nextInt // rng2를 사용
    ((i1, i2), rng3) // 난수 2개를 발생한 후, 최종 상태 반환
    // 위와 같이 진행할 경우, 호출자는 새 상태를 이용해서, 난수를 더 많이 발생시킬 수 있음
  }
  ```
  
#### 함수형 프로그래밍의 어색함 극복
- 순수성을 추구하는 것이
  - 마치 영어 소설 한 권을 `E`없이 쓰는 것일까?
- 물론 아님
  - 대부분 위와 같은 어색함은, 무언가를 **덜 추상화 했기 때문**에 비롯된 상황
- 조금 더 코드를 살펴보면서 **추출할 수 있는 공통 패턴**을 찾아보기

## 6.4. 상태 동작을 위한 더 나은 API
- 모든 함수가, 어떤 형식 `A`에 대해 `RNG => (A, RNG)` 형태의 형식을 가지는 **공통의 패턴**을 가짐
- 한 `RNG` 상태를 다른 `RNG` 상태로 변환한다는 점에서,
  - 이런 종류의 함수를 **상태 동작**(state action) / **상태 전이**(state transition)라고 부름
- 상태를 **호출자**가 직접 전달하는 것은
  - **반복적**임
- **조합기**가 자동으로 한 동작에서 다른 동작으로
  - **상태를 넘겨주는** 것이 바람직
- **상태 동작**의 형식을 편하게 지칭하고
  - **형식**에 대한 고찰을 단순화 하기 위해
  - 먼저 `RNG`의 상태 동작 자료 형식에 대한 `alias`를 만들기
    ```scala
    type Rand[+A] = RNG => (A, RNG)
    ```
- `Rand[A]` 형식의 값을 `무작위로 생성된 A`로 생각해도 되지만
  - 이는 하나의 **상태 동작**, 어떤 `RNG`에 의존하며
  - 그것을 이용하여 `A`를 생성하며
  - 또한 `RNG`를 다른 동작이 이후에 사용할 수 있는 **새로운 상태로 전이**하는 **하나의 프로그램**
- `RNG`의 `nextInt` 같은 메서드를 새로운 형식의 값으로 변환 가능
  ```scala
  val int: Rand[Int] = _.nextInt
  ```
- `Rand` 동작들을 조합하되,
  - `RNG` 상태들을 명시적으로 전달하지 않아도 되는 **조합기**를 작성하는 것이 가능
- 이렇게 발전해나가다 보면, 그런 전달을 자동으로 처리하는
  - 일종의 **영역 국한 언어**(domain-specific language, DSL)에 도달
- 예시로, 주어진 `RNG`를 사용하지 않고, 그대로 전달하는, 가장 간단한 형태의 `RNG` 상태 전이인 `unit`의 동작
  ```scala
  def unit[A](a: A): Rand[A] = 
    rng => (a, rng)
  ```
- 한 **상태 동작**의 출력을 변환하되, 상태 자체는 수정하지 않는 `map`
- `Rnad[A]`가 함수 형식 `RNG => (A, RNG)`의 별칭
  - `map`은 단순히 함수 합성
    ```scala
    def map[A,B](s: Rand[A])(f: A => B): Rand[B] =
      rng => {
        val (a, rng2) = s(rng)
        (f(a), rng2)
      }
    ```
- `map`의 예시로, `nonNegativeInt`를 재사용하여 `x>=0 && x%2==0`인 `Int`를 발생하는 함수 만들기
  ```scala
  def nonNegativeEven: Rand[Int] =
    map(nonNegativeInt)(i => i - i % 2)
  ```

### 6.4.1. 상태 동작들의 조합
- 두 `RNG` 동작을 단항 함수가 아니라 **이항 함수**로 조합하는 새로운 조합기 `map2`가 필요
  ```scala
  def map2[A,B,C](ra: Rand[A], rb:Rand[B])(f:(A, B) => C): Rand[C]
  ```
- `map2`를 한 번만 작성해 두면, 이를 활용하여 임의의 `RNG` 상태 동작들을 조합할 수 있음
- 예시로, `A`형식의 값을 발생하는 동작과, `B` 형식의 값을 발생하는 동작이 있다면,
  - 이 둘을 조합하여 `A, B`쌍을 발생하는 동작을 만들 수 있음
    ```scala
    def both[A,B](ra: Rand[A], rb: Rand[B]): Rand[(A,B)] = map2(ra, rb)((_, ))

    // intDouble
    val randIntDouble: Rand[(Int, Double)] = both(int, double)eere
    val randDoubleInt: Rand[(Double, Int)] = both(double, int)
    ```

### 6.4.2. 내포된 상태 동작
- 하나의 패턴
- 이를 잘 추출한다면 `RNG` 값을 명시적으로 언급하거나, 전달하지 않는 구현이 가능함
- `map, map2` 덕분에, 이전에는 작성하기 번거로웠던 함수들을
  - 우아하게 구현 가능
- 하지만 `map, map2`로 구현하지 못하는 함수도 존재
  - `0`이상 `n`미만의 정수 난수를 발생하는 `nonNegativeLessThan`
    ```scala
    def nonNegativeLessThan(n: Int): Rand[Int]

    // 음이 아닌 정수 난수를 `n`으로 나눈 나머지를 돌려주기
    def nonNegativeLessThan(n: Int): Rand[Int] =
      map(nonNegativeInt) { _ % n}
    ```
    - 요구된 범위의 난수가 발생하긴 하나, `Int.MaxValue`가 `n`으로 나누어 떨어지지 않을 수 있음
      - 전체적인 난수의 `치우침`이 발생
    - 나눗셈의 나머지보다 **작은 수**들이 좀 더 자주 나타나게 됨
- `nonNegativeInt`가 `32비트 정수`를 벗어나지 않는 `n`의 최대 배수보다 큰 수를 발생했다면
  - 더 작은 수가 나오길 바라면서 **발생하기를 재시도 해야함**
    ```scala
    def nonNegativeLessThan(n: Int): Rand[Int] =
      map(nonNegativeInt) { i => 
        val mod = i % n
        if (i + (n-1) - mod >= 0) mod else nonNegativeLessThan(n)(???) // Int가 32비트 Int를 벗어나지 않는 n의 최대 배수보다 크면 난수를 재귀적으로 다시 발생
      }
    ```
- 올바른 방향이긴 하나, `nonNegativeLessThan(n)`의 형식이 그 자리에 맞지 않는 것이 문제
  - 이 함수는 `Rand[Int]`를 돌려주어야 하며,
  - 그것은 `RNG`하나를 인수로 받는 함수
- `nonNegativeInt`가 돌려준 `RNG`가 `nonNegativeLessThan`에 대한 **재귀적 호출**에 전달되도록
  - 어떤 식으로는 호출들을 연결해야 함
- `map`을 사용하지 않고 명시적으로 전달하는 방법
  ```scala
  def nonNegativeLessThan(n: Int): Rand[Int] = {rng =>
    val (i, rng2) = nonNegativeInt(rng)
    val mod = i % n
    if (i + (n-1) - mod >= 0)
      (mod, rng2)
    else nonNegativeLessThan(n)(rng2)
  }
  ```
- 하지만 이러한 전달을 처리해주는 조합기가 있다면 좋을 것 => `flatMap`
  ```scala
  def flatMap[A,B](f: Rand[A])(g: A => Rand[B]): Rand[B]
  ```
- `flatMap`을 이용하면, `Rand[A]`로 무작위 `A`를 발생하고
  - 그 `A`값에 기초해서 `Rand[B]`를 선택할 수 있음
- `nonNegativeLessThan`에서는 `nonNegatvieInt`가 생성한 값에 기초해서
  - 재시도 여부를 선택하는데 `flatMap`을 이용
- 예시 : 검사성이 좋은 주사위 굴림 함수 만들기
  - `nonNegativeLessThan`을 이용한 `rollDie`의 구현
    ```scala
    // '더 모자라는 오류'는 여전히 존재
    def rollDie: Rand[Int] = nonNegativeLessThan(6)

    // 함수의 반환값이 0이 되는 RNG를 빠르게 확인 가능
    val zero = rollDie(SimpleRNG(5))._1

    // 동일한 SimpleRNG(5) 난수 발생기를 이용하면
    // 이러한 실패 상황을 신뢰성 있게 재현 가능, 상태가 사용후 파괴되는 것을 걱정할 필요 없음
    // 버그가 간단히 교정됨
    def rollDie: Rand[Int] = map(nonNegativeLessThan(6))(_ + 1)
    ```

## 6.5. 일반적 상태 동작 자료 형식
- `unit, map, map2, flatMap, sequence`는 난수 발생에만 국한된 것이 아님
- 이들은 **상태 동작**에 작용하는 **범용 함수**들로
  - 상태의 **구체적인 종류**는 신경쓰지 않음
- `map`은 자신이 `RNG` 상태 동작을 상관하지 않으며, 이 함수에 보다 일반적인 **서명** 부여 가능
  ```scala
  def map[S,A,B](a: S => (A,S))(f: A => B): S => (B,S)
  ```
- 임의의 상태를 처리할 수 있는 ,`Rand`보다 일반적인 형식
  ```scala
  type State[S,+A] = S => (A,S)

  // 독립적인 클래스 형태 정의
  case class State[S,+A](run : S => (A,S))

  // Rand를 State를 이용해 정의
  type Rand[A] = State[RNG, A]
  ```
- `State`는 **어떤 상태를 유지하는 계산**
  - 즉, **상태 동작** 또는 **상태 전이**를 대표함
- 심지어 `Statement`를 대표한다고 설명이 가능

## 6.6. 순수 함수적 명령식 프로그래밍
- 특정한 패턴을 따르는 함수
  - 상태 동작을 실행하고
  - 그 결과를 `val`에 배정하고
  - 그 `val`을 사용하는 또 다른 **상태 동작**을 실행
  - 그 결과를 다른 `val`로 배정
- 이런 방식은 **명령식 프로그래밍**(imperative programming)과 유사
- **명령식 프로그래밍 패러다임**에서
  - 하나의 프로그램은 **명령문**(statement; 또는 문장)으로 이루어지며,
  - 각 명령문은 **프로그램 상태 수정** 가능
- **함수**로서의 상태 동작은
  - 그냥 **인수**를 받음으로서, 현재 프로그램 **상태**를 읽고,
  - 그냥 값을 돌려줌으로써, **프로그램 상태 수정**

#### 명령식 프로그래밍 vs 함수형 프로그래밍
- 절대 상극이 아님
- 함수형 프로그래밍
  - 단순히 **부수 효과가 없는 프로그래밍**을 의미
- 명령식 프로그래밍
  - 프로그램 **상태를 수정**하는 명령문들로 프로그램을 만드는 것
  - 부수 효과 없이 상태 유지/관리도 가능
- **함수형 프로그래밍**은 **명령식 프로그램의 작성**을 아주 잘 지원함
- **참조 투평석** 덕분에,
  - 그런 프로그램을 **동시적으로 추론**할 수 있다는 장점도 존재
- **동시적 추론**은 **제2부**
- **명령식 프로그램**은 **제3,4부**

### 명령식 프로그램의 성격
- `map, map2`같은 **조합기**를 구현했고,
  - 결국 한 명령문에서, 다른 명령문으로 상태 전파를 처리하는 `flatMap`까지 도달
- 단, 그 과정에서 **명령식 프로그램의 성격은 많이 사라짐**
- 예시 : `Rand[A]`는 `State[RNG, A]`의 형식 별칭
  ```scala
  val ns: Rand[List[Int]] =
      int.flatMap(x => // int는 하나의 정수 난수를 발생하는 `Rand[Int]`형식의 값
        int.flatMap(y =>
          ints(x).map(xs => // `ints(x)`는 길이가 `x인 목록 생성
            ex.map(_ % y)))) // 목록의 모든 요소를 `y`로 나눈 나머지로 치환
  ```
  - 위 코드는 작동방식을 이해하기 어려움
  - `map`과 `flatMap`을 사용하여 `for-comprehension`을 이용하여 명령식 스타일 복구
    ```scala
    val ns:Rand[List[Int]] = for {
      x <- Int, // 정수 난수 x
      y <- Int, // 정수 난수 y
      xs <- ints(x) // 길이가 x인 목록 xs 생성
    } yield xs.map(_ % y) // xs의 각 요소를 y로 나눈 나머지로 치환한 목록 반환
    ``` 
  - 위 코드는 읽기/쓰기 쉬움
  - 이 코드가 어떤 **상태**를 유지하는 **명령식 프로그램**이라는 점이
    - 코드의 형태 자체에 잘 반영되어 있기 때문
  - 그러나 이전과 같은 코드
- `for-함축`(`flatMap`)을 활용하는 이런 종류의 **명령식 프로그래밍**을 지원하는 데 필요한 것은
  - 기본적인 `State` 조합기 두 개뿐
- **상태를 읽는 조합기**와 **상태를 쓰는 조합기**만 있으면 됨
- 현재 상태를 얻는 조합기는 `get`이며
  - 새 상태를 설정하는 조합기가 `set`이라 할때,
  - 상태를 임의의 방식으로 수정하는 조합기를 다음과 같이 구현
    ```scala
    def modify[S](f:S => S): State[S, Unit] = for {
      s <- get // 현재 상태를 얻어서 s에 배정
      _ <- set(f(s)) // 새 상태를 s에 f를 적용한 결과로 설정
    } yield ()
    ```
    - 이 메서드는 주어진 상태를 `f`로 수정하는 `State`동작을 돌려줌 
    - 그리고 상태 이외의 반환값을 없음을 알리기 위해 `Unit`을 산출
  - 위 코드를 활용한 `get, set`의 모습
    ```scala
    // 입력 상태를 전달하고, 이를 반환값으로 반환
    def get[S]: State[S, S] = State(s => (s, s))

    // 새 상태 s를 받음
    // 결과로 나오는 동작은 입력 상태를 무시하며, 그 것을 새 상태로 치환
    // 의미 있는 값 대신 ()를 반환
    def set[S](s:S): State[S, Unit] = State(_ => ((), s))
    ```
- 이 간단한 두 동작과 이전에 작성한 `State` 조합기(`unit, map, map2, flatMap`)만 있으면
  - 그 어떤 종류의 **상태기계**(state machine)이나, **상태 있는 프로그램**이라도
  - **순수 함수적 방식**으로 구현 가능
  
## 6.7. 요약
- **상태를 가진 프로그램**을 **순수 함수적 방식으로 작성**
- `상태`를 인수로 받고
  - `새 상태`를 `결과`와 함께 돌려주는 **순수 함수** 사용
- 나중에 **부수 효과**에 의존하는 **명령식 API**를 만나게 되면
  - 그것을 **순수 함수적 버전**으로 바꿀 수 있는지 시도해 볼 것
  - 이번 장에서 작성한 함수들을 활용하면, 그러한 변환이 더 쉬울 것