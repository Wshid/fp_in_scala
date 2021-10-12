# [CHAP.07] 순수 함수적 병렬성
- 함수적 프로그래밍의 가정들이 **라이브러리 설계**에 어떤 영향을 미치는지
- 함수적 라이브러리 설계에 정답은 없음
  - 각자 `trade-off` 또는 장단점이 다른 `design choice`가 있음
- **병렬** 및 **비동기** 계산의 생성을 위한 **순수 함수적 라이브러리** 만들기
- 병렬적 프로그램에 내재하는 **복잡성**을
  - 순수 함수만으로 서술하기
- **치환 모형**을 이용하여 단순화 가능
  - 동시적 계산을 쉽게 할 수 있게 됨
- 계산의 **서술**이라는 관심사를
  - 계산의 실제 **실행**이라는 관심사와 분리하기
- 라이브러리 사용자가 라이브러리르 활용하여
  - 프로그램의 구체적인 실행 방식에 관한 세부사항과는 **격리된 수준**에서
  - 프로그램을 작성하기
  - 예시
    - `parMap`이라는 조합기를 개발
    - `f`를 한 컬렉션의 모든 요소에 동시 적용 가능
      ```scala
      val outputList = parMap(inputList)(f)
      ```
- 항상 **대수적 추론**에 역점을 두고
  - API를 특정 **법칙**들을 따르는, 하나의 **대수**(algebra)로 서술할 수 있다는 착안을 소개

## 7.1. 자료 형식과 함수의 선택
- 라이브러리 개발하기 : 병렬 계산을 생성할 수 있어야 함
- 병렬화할 계산 : 정수들의 합 구하기
  ```scala
  // foldLeft를 활용한 코드
  def sum(ints: Seq[Int]): Int =
    ints.foldLeft(0)((a,b) => a + b)
  ```
  - `Seq`에 `foldLeft` 메서드 존재
- 정수들은 순차적으로 접는 대신, `divide-and-conquer` 알고리즘 적용 가능

#### CODE.7.1. 분할정복 알고리즘을 이용한 목록 합산
```scala
// IndexedSeq; Vector와 비슷한 임의 접근 순차열의 상위 클래스
// 목록과는 달리, 순차령릉 특정 지점에서 두 부분으로 분할하는 효율적인 `splitAt` 메서드 제공
def sum(ints: IndexedSeq[Int]): Int =
  if (ints.size <= 1)
    ints.headOption getOrElse 0 // headOption, 모든 컬렉션에 정의되는 함수
  else {
    val (l, r) = ints.splitAt(ints.length/2) // splitAt 함수, 순차열을 반으로 나눔
    sum(l) + sum(r) // 재귀적으로 두 절반을 합치기
  }
```
- `splitAt`을 이용하여 **절반으로 분할**
  - 재귀적으로 두 절반을 합하여 결과를 합침
  - `foldLeft` 기반 구현과는 달리, 이 구현은 **병렬화** 가능
- 두 절반을 **병렬**로 합할 수 있다는 의미
- **병렬성**을 구현하기 위해
  - `Java.lang.Thread / Java.util.concurrent` 라이브러리 사용이 아닌,
  - 간단한 예제에서 영감을 얻어
    - 이상적인 API를 설계
    - 이후 구현으로 되돌아가는 접근방식

### 7.1.1. 병렬 계산을 위한 자료 형식 하나
- `sum(l) + sum(r)`
  - 두 절반에 대해 재귀적으로 `sum` 호출
  - 병렬 계산을 나타내는 자료 형식이, **하나의 결과**를 담아야 함
  - 그 결과는 의미 있는 형식 이어야 하며
    - 그 결과를 **추출하는 수단**도 갖추어야 함
- 결과를 담을 컨테이너 형식 `Par[A]`를 새로 invent
  - 이 형식에 필요한 함수
    - `def unit[A](a: => A): Par[A]`
      - 평가되지 않은 `A`를 받음
      - 개별적인 스레드에서 평가할 수 있는 계산을 반환
      - `unit`의 의미
        - 하나의 값을 감싸는 병렬성의 한 단위
    - `def get[A](a: Par[A]): A`
      - 병렬 계산에서 값을 추출

#### CODE.7.2. 새 커스텀 자료 형식을 이용하는 sum 함수
```scala
def sum(ints: IndexedSeq[Int]): Int =
  if (ints.size <= 1)
    ints headOption getOrElse 0
  else {
    val (l, r) = ints.splitAt(ints.length/2)
    val sumL: Par[Int] = Par.unit(sum(l)) // 왼쪽 절반 병렬 계산
    val sumR: Par[Int] Par.unit(sum(r)) // 오른쪽 절반 병렬 계산
    Par.get(sumL) + Par.get(sumR) // 두 결과를 합치기
  }
```
- 두 재귀적 `sum`호출을 `unit`으로 감싸고
  - 두 부분의 계산 결과를 `get`을 이용해 추출
- `unit`
  - 주어진 인수를 **개별적인 스레드**에서 즉시 평가 가능
  - 인수를 가지고 있다가, `get`이 호출되면 평가 가능
  - 병렬성의 이점을 위해서는
    - `unit`이 동시적 평가를 시작한 후, **즉시 반환**해야 함
- `unit`의 인수들이 평가를 동시에 시작한다면
  - `get`의 **참조 투명성**이 깨질 수 있음
  - 물론, `sumL`과 `sumR`을 해당 정의로 동시에 치환해보면, 이점이 명백해짐
  - 치환 결과가 나오긴 하나, 프로그램이 병렬로 실행되지 않음
    ```scala
    Par.get(Par.unit(sum(l))) + Par.get(Par.unit(sum(r)))
    ```
- `unit`이 자신의 인수를 **즉시 평가**하기 시작한다면,
  - 그 다음은 `get`이 그 평가의 완료를 기다리는 것
- `sumL`과 `sumR` 변수를 단순히 나열하면,
  - `+`의 양변은 **병렬로 실행되지 않음**
- 이는 `unit`에 한정적인 **부수 효과 존재**를 의미
  - 단, 그 부수 효과는 `get`에만 관련된 것
- 즉, `unit`은 **비동기 계산**을 나타내는 `Par[Int]`를 돌려줌
- `Par`를 `get`으로 넘겨주는 즉시,
  - `get`의 완료까지 실행이 차단된다는 **부수 효과**가 드러남
  - `get`을 호출하지 않거나, 호출을 ㅗ치대한 미루어야 함
- 비동기 계산들을 `그 완료를 기다리지 않고, 조합할 수 있어야 함`

#### 동시성 기본 수단을 직접 사용하는 것의 문제점
- `java.lang.Thread / Runnable`을 사용한다면?
  ```scala
  trait Runnable { def run: Unit}

  class Thread(r: Runnable) {
    def start: Unit // 개별 스레드에서 `r`의 실행을 시작
    def join: Unit // 호출한 스레드의 실행은 `r`의 실행이 끝날때까지 차단
  }
  ```
- 두 메서드 모두 **의미 있는 값을 돌려주지 않음**
- 즉, `Runnable`에서 어떤 정보를 얻으려면
  - 조사 가능한 어떤 상태를 변경하는 등의, **부수 효과**가 발생함
  - 합성 능력에 해가 됨
- `Runnable` 객체를 다룰 때는,
  - 항상 그 **내부 행동 방식**에 대해 무언가를 알아야 하기 때문에
  - 이를 `generic` 스타일로 조작 불가
- `Thread` 역시
  - **운영체제의 스레드**에 직접 대응된다는 단점 존재
- 바람직한 방식은,
  - **논리적 스레드**를 주어진 문제에 필요한 만큼, 얼마든지 생성하게 하고
  - 그것을 실제 `OS thread`에 대응시키는 것을 **나중에 처리**
- 이런 종류의 문제점을 `java.util.concurrent.Future / ExecutorService`로 처리 가능
  ```scala
  class ExecutorService {
    def submit[A](a: Callable[A]) : Future[A]
  }

  trait Future[A] {
    def get:A
  }
  ```
- 위와 같은 기본 수단이 **물리적 스레드**의 추상화에 도움이 되긴 하나,
  - 훨씬 낮은 수준의 추상
- `Future.get`을 호출하면
  - 호출한 스레드는 `ExecutorService`의 실행이 끝날때까지 차단됨
- 그리고 `API`는 `Future`를 **합성**하는 수단을 전혀 제공하지 않음
- 라이브러리 구현을 이 기본수단에 기초해서 구축할 수는 있음
- 그러나, 함수적 프로그램에서 직접 사용할 만한
  - 모듈적이고, 합성 능력이 있는 API는 제공하지 않음