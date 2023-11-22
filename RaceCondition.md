## Race Condition
Race Condition(경쟁 상태)에 대해 위키백과에서는 다음과 같이 정의되어있다.
> 경쟁 상태란 공유 자원에 대해 여러 개의 프로세스가 동시에 접근을 시도할 때 접근의 타이밍이나 순서 등이 결과값에 영향을 줄 수 있는 상태를 말한다.
> 동시에 접근할 때 자료의 일관성을 해치는 결과가 나타날 수 있다.

<br>

예시 코드를 살펴보자면
```swift
import Foundation

var cards = [1, 2, 3, 4, 5, 6, 7, 8, 9]

DispatchQueue.global().async {
    for _ in 1...3 {
        let card = cards.removeFirst()
        print("쥬봉이: \(card) 카드를 뽑았습니다!")
    }
}

DispatchQueue.global().async {
    for _ in 1...3 {
        let card = cards.removeFirst()
        print("시루: \(card) 카드를 뽑았습니다!")
    }
}

DispatchQueue.global().async {
    for _ in 1...3 {
        let card = cards.removeFirst()
        print("마루: \(card) 카드를 뽑았습니다!")
    }
}

/*
Swift/RangeReplaceableCollection.swift:622: Fatal error: Can't remove first element from an empty collection
쥬봉이: 1 카드를 뽑았습니다!
마루: 1 카드를 뽑았습니다!
쥬봉이: 3 카드를 뽑았습니다!
쥬봉이: 4 카드를 뽑았습니다!
마루: 5 카드를 뽑았습니다!
마루: 6 카드를 뽑았습니다!
*/
```

위의 예시처럼 Race Condition은 `스레드가 여러개인 상황에서 코드가 동시에 실행되어 하나의 값에 동시에 접근하는 경우`에 발생할 수 있다.   
Race condition이 발생하는 이유는 `Swift의 배열이 Thread Safe하지않기 때문`이다.   

- `Thread Safe 하다` : 여러 스레드에서 동시에 접근이 불가능하다 -> 즉, Thread Unsafe는 여러 스레드에서 동시에 접근이 가능하다는 말!
- Swift의 대부분의 타입들은 `Thread Unsafe`하다

<br>

그렇다면 Race Condition을 해결하는 방법들이 어떤 것들이 있는지 살펴보자

<br>

### DispatchSemaphore
semaphore에 대해 위키백과에서는 다음과 같이 정의되어 있다.
> 정수 변수로서, 멀티프로그래밍 환경에서 **공유 자원에 대한 접근을 제한하는 방법**으로 사용된다.   
> 스레드가 **공유 자원의 배타적인 사용**을 보장받기 위해서 **임계 구역**에 들어가거나 나올때는 **세마포어** 같은 **동기화 매커니즘**이 사용된다.
  
세마포어는 모든 교착 상태에 대한 해답을 제시해 줄 수 없으나 교착 상태 해법에 대한 고전적인 해법으로 많이 언급이 되고 있다.   

<br>

``` swift
class DispatchSemaphore : DispatchObject
```
애플 공식문서에서 DispatchSemaphore는 `전통적인 counting semaphore를 사용하여 여러 실행 컨텍스트에서 리소스에 대한 액세스를 제어하는 개체`로 설명이 되어있다.   
DispatchSemaphore는 공유 자원에 접근할 수 있는 스레드의 수를 제어해주는 역할을 하기때문에 race condition을 해결할 수 있다.   

<br>

```swift
let semaphore = DispatchSemaphore(value: 1) // count = 1

DispatchQueue.global().async {
    semaphore.wait() // count -= 1

    semaphore.signal() // count += 1
}
```

- init(value:) : 접근 가능 스레드 갯수
- signal() : count up (count += 1)
- wait() : count down (count -= 1)
- wait()는 값에 접근했다고 알리는 메서드, signal()은 볼 일 다봤다는 메서드인 셈!

<br>

예시 코드
```swift
import Foundation

var cards = [1, 2, 3, 4, 5, 6, 7, 8, 9]
let semaphore = DispatchSemaphore(value: 1)

DispatchQueue.global().async {
    for _ in 1...3 {
        semaphore.wait()
        let card = cards.removeFirst()
        print("쥬봉이: \(card) 카드를 뽑았습니다!")
        semaphore.signal()
    }
}

DispatchQueue.global().async {
    for _ in 1...3 {
        semaphore.wait()
        let card = cards.removeFirst()
        print("시루: \(card) 카드를 뽑았습니다!")
        semaphore.signal()
    }
}

DispatchQueue.global().async {
    for _ in 1...3 {
        semaphore.wait()
        let card = cards.removeFirst()
        print("마루: \(card) 카드를 뽑았습니다!")
        semaphore.signal()
    }
}

/*
쥬봉이: 1 카드를 뽑았습니다!
시루: 2 카드를 뽑았습니다!
마루: 3 카드를 뽑았습니다!
쥬봉이: 4 카드를 뽑았습니다!
시루: 5 카드를 뽑았습니다!
마루: 6 카드를 뽑았습니다!
쥬봉이: 7 카드를 뽑았습니다!
시루: 8 카드를 뽑았습니다!
마루: 9 카드를 뽑았습니다!
*/
```

<br>

### SerialQueue (엄격한 Thread-safe)
한 번에 하나의 작업만 처리할 수 있는 작업 큐(Serial Queue)를 만들면 큐를 사용하는 구성 요소에 스레드 안전성을 간접적으로 도입하게 된다.

```swift
DispatchQueue.global().async {
    for _ in 1...3 {
        pickCardsSerialQueue.sync {
            let card = cards.removeFirst()
            print("쥬봉이: \(card) 카드를 뽑았습니다!")
        }
    }
}

DispatchQueue.global().async {
    for _ in 1...3 {
        pickCardsSerialQueue.sync {
            let card = cards.removeFirst()
            print("시루: \(card) 카드를 뽑았습니다!")
        }
    }
}

DispatchQueue.global().async {
    for _ in 1...3 {
        pickCardsSerialQueue.sync {
            let card = cards.removeFirst()
            print("마루: \(card) 카드를 뽑았습니다!")
        }
    }
}

/*
시루: 1 카드를 뽑았습니다!
마루: 2 카드를 뽑았습니다!
쥬봉이: 3 카드를 뽑았습니다!
시루: 4 카드를 뽑았습니다!
마루: 5 카드를 뽑았습니다!
쥬봉이: 6 카드를 뽑았습니다!
시루: 7 카드를 뽑았습니다!
마루: 8 카드를 뽑았습니다!
쥬봉이: 9 카드를 뽑았습니다!
*/
```
SerialQueue에 sync를 사용하는 이유는 serial queue로 보낸 작업을 기다림으로써 공유 자원의 제대로 된 값을 얻기 위함이다.     

<br>

### Concurrent queue + Dispatch Barrier 활용
`dispatch barrier`은 concurrent queue가 사용하고 있는 여러 개의 thread 중에서, 
barrier block의 실행을 위한 하나의 스레드만 제외하고는 모든 스레드 사용을 막는 것으로 
해당 시점에서는 스레드를 하나만 사용하기 때문에 serial로 순서 있는 실행이 가능하다.   
공식문서에는 `concurrent dispatch queue에서 실행되고 있는 task 들에 대한 동기화 지점`으로 설명되어 있다.   
``` swift
concurrentQueue.async(flags: .barrier, execute: { })
```
read 작업은 일반적인 task로 넣고, write 작업을 barrier task로 넣는 식으로 응용가능하다.

<br>
Barrier은 조금 더 공부를 해봐야겠다.😱

---
#### 참고 문서
[야곰 닷넷 - 동시성 프로그래밍](https://yagom.net/courses/%eb%8f%99%ec%8b%9c%ec%84%b1-%ed%94%84%eb%a1%9c%ea%b7%b8%eb%9e%98%eb%b0%8d-concurrency-programming/)   
[Thread safety in swift](https://swiftrocks.com/thread-safety-in-swift)   
[차근차근 시작하는 gcd](https://sujinnaljin.medium.com/ios-%EC%B0%A8%EA%B7%BC%EC%B0%A8%EA%B7%BC-%EC%8B%9C%EC%9E%91%ED%95%98%EB%8A%94-gcd-13-723d33e157e)   
