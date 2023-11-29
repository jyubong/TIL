## 작업하는 스레드의 수 제한
`상황 : 예금 은행원 2명, 대출 은행원 1명이 비동기적으로 고객의 업무를 처리하는 상황`

<br>

### 문제 상황
GCD로 비동기를 구현하였고, 동시에 실행되는 작업의 개수를 제한하기위해 DispatchSemaphore를 사용했다.   
대출 은행원은 1명으로 serial queue로 구현해준 점은 생략하였다.
DispatchSemaphore에 대한 설명은 [RaceCondition.md](https://github.com/jyubong/TIL/blob/main/RaceCondition.md#dispatchsemaphore)에 작성해두었다.   

```swift
// (생략)
let group = DispatchGroup()
let semaphore = DispatchSemaphore(value: 2)
let depositQueue = DispatchQueue(label: "depositQueue", attributes: .concurrent)

while let customer = customerQueue.dequeue() as? BankClerk.Customer {
    let banking = customer.banking

    switch banking {
    case .deposit:
        depositQueue.async(group: group) {
            semaphore.wait()
            bankClerk[banking]?.receive(customer: customer)
            semaphore.signal()
        }
// (생략)
}
```
결론적으로 은행원 2명이 일을 처리하는 상황이되었지만... 문제가 있었다.

<br>

<img width="871" alt="스크린샷 2023-11-23 오후 5 27 14" src="https://github.com/jyubong/TIL/assets/126065608/65e7f216-de44-4d46-9f56-4d55bb67654d">   
<br>
은행원의 수보다 훨씬 많은 스레드가 생성이 된것이다.   
어떻게 된 일인고 하니

![gcd1 001](https://github.com/jyubong/TIL/assets/126065608/eb6c041e-0afb-428b-a4e5-dc02de9c1b9f)
`concurrent queue에 async로 고객을 보내고 있는 상황`으로 고객을 족족 새로운 은행원들(새로운 스레드)에게 보내지만 정작 2명의 은행원만(2개의 스레드만) 업무를 처리하고 있는 것이다.   
이것이 바로 GCD의 문제점으로 많이 언급되는 thread explosion 상황?!(폭발까지는 아니지만... 발생할 가능성이 다분한 상황)

<br>

그래서 은행원 수 만큼 스레드를 생성하도록 구현을 해보기로 했다.

<br>

### SerialQueue + Semaphore
위의 상황에서 `concurrentQueue 안에서 semaphore wait()을 걸어주는 상황이 문제`가된 것 같다.   
그래서 이번에는 `concurrentQueue 밖에서 semaphore wait()을 걸어주어 접근 수 제한이 걸렸을 경우 concurrent queue에 진입하지 못하고 기다리는 방향`으로 변경하였다.   
```swift
let semaphore = DispatchSemaphore(value: 2)
let depositQueue = DispatchQueue(label: "depositQueue", attributes: .concurrent)
let newQueue = DispatchQueue(label: "newQueue")

while let customer = customerQueue.dequeue() as? BankClerk.Customer {
    let banking = customer.banking

    switch banking {
    case .deposit:
        newQueue.async(group: group) {
            semaphore.wait()
            depositQueue.async(group: group) {
                bankClerk[banking]?.receive(customer: customer)
                semaphore.signal()
            }
        }
//(생략)
}
```
다만 **main thread에서 wait()을 호출할 경우 main thread가 block** 되어버리게 되므로 `새로운 serial queue 안에서 wait()이 호출되도록` 감싸주었다.

<br>

하지만 이 방법 또한 완전한 방법은 아니었다.
<img width="886" alt="스크린샷 2023-11-23 오후 5 26 20" src="https://github.com/jyubong/TIL/assets/126065608/817433e6-501a-42d0-809c-e35d7f19c9aa">
![스크린샷 2023-11-23 오후 6 43 36](https://github.com/jyubong/TIL/assets/126065608/46ae39e3-1feb-4684-85bd-f14799a16ca3)


<br>

감싸준 serial queue의 thread가 추가 생성되어, 은행원 수 + 1개의 thread가 생성되었다.

![image](https://github.com/jyubong/TIL/assets/126065608/0796d697-eaa8-46ae-98c6-0b7a2cb745c6)


풀어쓰자면 serial queue(thread 1개)는 안내원이 되어 concurrent queue 2명(thread 2개)에게 안내해주는 역할처럼 된것!

<br>

serial Queue를 위한 thread가 1개 추가된 점말고는 스레드 수를 제한하는데 성공을 했다.
사람의 욕심은 끝이없다고 완전히 은행원 수만큼 thread를 만들 수는 없을까?

<br>

### 모두 SerialQueue로 구현
serial queue는 thread를 1개만들어 작업을하니 모든 은행원을 serial queue로 만들면 되지 않을까?에서 나온 로직이다.
``` swift
var doneCustomers = Set<UInt>()
let group = DispatchGroup()
let semaphore = DispatchSemaphore(value: 1)
let depositQueue1 = DispatchQueue(label: "depositQueue1")
let depositQueue2 = DispatchQueue(label: "depositQueue2")
let loanQueue = DispatchQueue(label: "loanQueue")

while let customer = customerQueue.dequeue() as? BankClerk.Customer {
    let banking = customer.banking
    
    switch banking {
    case .deposit:
        let workItem = DispatchWorkItem {
            semaphore.wait()
            if !doneCustomers.contains(customer.number) {
                doneCustomers.insert(customer.number)
                semaphore.signal()
                bankClerk[banking]?.receive(customer: customer)
            }
            semaphore.signal()
        }
        
        depositQueue1.async(group: group, execute: workItem)
        depositQueue2.async(group: group, execute: workItem)
    case .loan:
        loanQueue.async(group: group) {
            bankClerk[banking]?.receive(customer: customer)
        }
    }
}
```
depositQueue 또한 serial queue로 만들었다.   
이때 추가된 로직은 예금업무를 처리한 customer를 따로 모아두는 것이다.  
비동기 작업이 시작되면 semaphore wait()을 걸어주고, 예금업무를 이미한 고객 리스트에 해당 고객이 있다면 바로 siagnal()을 소환하고 아니라면 insert 후 소환한다. 그리고 업무를 처리해준다.   
depositQueue 2개가 같은 고객을 갖는 것을 방지하기 위한 로직인 것이다.   

<br>

![스크린샷 2023-11-23 오후 7 07 45](https://github.com/jyubong/TIL/assets/126065608/8adc0b87-54bc-4cc0-84f0-eb90e3d26277)
![스크린샷 2023-11-23 오후 7 08 02](https://github.com/jyubong/TIL/assets/126065608/f4f6eaca-701d-4b65-9674-3f8f0663d921)   
결과적으로 thread 수가 은행원 수에 맞게 생성되었다.
![image](https://github.com/jyubong/TIL/assets/126065608/ee35ce0e-f53b-4331-a0e0-3146e077ad54)
하지만 고객이 2개의 대기줄에 서고, 이미 업무를 본 손님인지 확인후 업무를 봤으면 넘어가고 안봤으면 업무를 처리해주는 상황이다.   
빠른 쪽을 찾아 고객이 분신술을 써서 줄을 2개 선 상황갔달까... 그리고 큰 문제가 발생하고야 말았다.   

<br>

![스크린샷 2023-11-23 오후 7 08 26](https://github.com/jyubong/TIL/assets/126065608/1808239d-8289-4e4f-8dd4-8aa1389f689a)
contains나 insert를 할때 종종 다음과 같은 문제가 발생하는것...   
어떤때는 실행이 잘 되다가 어떤때는 실행이 안된다😱 아직 해결하지 못한게 함정...


<br>

### Operation
이번에는 Operation으로 비동기를 구현해주었다.
``` swift
private let depositCounter: OperationQueue = {
    let queue = OperationQueue()
    queue.maxConcurrentOperationCount = 2

    return queue
}()

private let loanBankCounter: OperationQueue = {
    let queue = OperationQueue()
    queue.maxConcurrentOperationCount = 1
    
    return queue
}()
```
```swift
//(생략)
while let customer = customerQueue.dequeue() as? BankClerk.Customer {
    let banking = customer.banking
    let operation = BlockOperation {
        bankClerks[banking]?.receive(customer: customer)
    }

    switch banking {
    case .deposit:
        depositCounter.addOperation(operation)
//(생략)
}
```
OperationQueue에 addOperation()으로 operation을 실행할 경우 operation을 각각 새로운 스레드를 만들어 작업하게 된다.   
semaphore 대신 Operation에는 `maxConcurrentOperationCount로 OperationQueue의 작업(Operation) 수를 제한`할 수 있다.   
여기서 maxConcurrentOperationCount를 2로 설정하면, 해당 OperationQueue는 2개까지만 작업을 할 수 있게된다.

<br>

![스크린샷 2023-11-23 오후 6 30 53](https://github.com/jyubong/TIL/assets/126065608/476c73df-69c6-40c8-9fcf-b0b8733d8730)
![스크린샷 2023-11-23 오후 6 47 37](https://github.com/jyubong/TIL/assets/126065608/114d0ee0-0c8a-43ac-bbf8-ce1a84128d70)   
스레드 수가 3개만 생길거라는 예상과는 다르게 추가적으로 4개의 thread가 생성되었다.   
아직 그 이유를 못 찾은 상황이다.😱

<br>

---
결론적으로 semaphore의 적절한 사용, operaionQueue를 활용하면 thread 수를 제한을 할 수는 있다.   
thread의 갯수를 줄였지만 원하는 만큼 줄이지는 못한 상황으로 더 공부를 해봐야겠다. 흑흑   
(swift cuncurrency는 아직입니다🌝)
