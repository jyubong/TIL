해당 TIL은 swift programming language를 바탕으로 작성하였습니다.   

<br>

## 캡처리스트(Capture List)
캡처리스트를 알아보기에 앞서 클로저의 캡처에 대해서 알아보자

### 클로저의 캡처란?    
**클로저 내부에서 외부에 있는 값에 접근하면 값에 대한 참조를 획득**를 하는데 이것을 `캡처`라고 부른다.   
캡처를 하게 되면 상수와 변수를 정의한 원래의 범위가 더 이상 존재하지 않더라도 클로저 본문 내에서 해당 상수와 변수의 값을 참조하고 수정가능하게 된다.
``` swift
func makeIncrementer(forIncrement amount: Int) -> () -> Int {    
	var runningTotal = 0    
	func incrementer() -> Int {        
		runningTotal += amount        
		return runningTotal    
		}    
	return incrementer
}
```
위의 예제는 공식문서에 나와있는 예제이다.    
`incrementer()`은 `runningTotal`과 `amount`를 캡처하고 있는 것을 알 수 있다.
```swift
let incrementByTen = makeIncrementer(forIncrement: 10)

incrementByTen()
// returns a value of 10
incrementByTen()
// returns a value of 20
incrementByTen()
// returns a value of 30
```

<br>

클래스 인스턴스의 속성에 클로저를 할당하고 클로저가 인스턴스 또는 해당 멤버를 참조하여 해당 인스턴스를 캡처하는 경우 클로저와 인스턴스 사이에 강한 참조 순환이 만들어진다. 
강한 참조 사이클이 형성되면 인스턴스가 정상적으로 해제되지 않는 일이 발생할 수 있다.   
Swift는 이러한 문제를 해결하기위한 우아한 방법을 제시하였는데, 그것이 바로 `캡처리스트`이다.

<br>

### 클로저의 강한 참조 순환
강한 참조 순환은 클래스 인스턴스의 속성에 클로저를 할당하고 해당 클로저의 본문이 인스턴스를 캡처하는 경우에도 발생할 수 있다.   

<br>

이 경우 왜 강한 참조 순환이 발생할까?🤔   
`클로저가 클래스와 같은 참조 형식이기 때문`에 발생한다!   
1. 프로퍼티에 클로저를 할당하면
2. 프로퍼티에 해당 클로저에 대한 참조가 할당된다.
3. 본질적으로 두 개의 강한 참조가 서로를 살리고 있는 것 -> 서로를 유지시키는 클래스 인스턴스와 클로저인 샘!    

공식문서 예제로 살펴보도록 하자
```swift
class HTMLElement {
    let name: String
    let text: String?

    lazy var asHTML: () -> String = {
        if let text = self.text {
            return "<\(self.name)>\(text)</\(self.name)>"
        } else {
            return "<\(self.name) />"
        }
    }

    init(name: String, text: String? = nil) {
        self.name = name
        self.text = text
    }

    deinit {
        print("\(name) is being deinitialized")
    }
}
```
`asHTML은 name 과 text 을 HTML 문자열로 결합하는 클로저를 참조`하고 있다.   
```swift
var paragraph: HTMLElement? = HTMLElement(name: "p", text: "hello, world")
print(paragraph!.asHTML())
// Prints "<p>hello, world</p>"
```

![](https://docs.swift.org/swift-book/images/closureReferenceCycle01@2x.png)   
1. 인스턴스의 asHTML 속성은 클로저에 대한 강한 참조를 보유하고
2. 클로저는 본문 내에서 self를 참조하고 있다.
3. 클로저는 self.name, self.text를 캡처하는데 -> HTMLElement 인스턴스에 대한 강력한 참조를 다시 보유한다는 것을 의미한다.
4. 즉, 둘 사이에 강한 참조 주기가 생성된다.
5. paragraph 변수를 nil로 설정하고 HTMLElement 인스턴스에 대한 강한 참조를 끊으면 강력한 참조 주기로 인해 HTMLElement 인스턴스와 클로저의 할당이 모두 취소되지 않게 된다.

```swift
paragraph = nil
// deinit이 호출되지 않음...
```

<br>

클로저와 클래스 인스턴스 사이의 강력한 참조 주기는 클로저 정의의 일부로 캡처 리스트를 정의하여 해결 가능하다. 

<br>

### 캡처리스트
`캡처 리스트`는 **클로저 본문 내에서 하나 이상의 참조 형식을 캡처할 때 사용할 규칙을 정의하는 것**으로   
캡처된 각 참조를 강한 참조가 아닌 `약한 참조(weak)나 미소유 참조(unowned)`로 선언한다.    
쉽게 말하면, 캡처리스트는 `클래스 인스턴스(예: self) 또는 일부 값으로 초기화된 변수(예: delegate = self.delegate)에 대한 참조 + weak 또는 unowned`의 쌍이다. (전혀 쉽지 않은게 함정...)
```swift
lazy var someClosure = {
        [unowned self, weak delegate = self.delegate]
        (index: Int, stringToProcess: String) -> String in
    // closure body goes here
}
```
`[unowned self, weak delegate = self.delegate]` 쉼표로 구분된 한 쌍의 대괄호 안에 기록하며,   
클로저의 매개 변수 리스트 앞에 캡처 리스트와 제공된 리턴 타입을 배치한다.   

<br>

만약 클로저가 매개변수 목록 또는 반환 형식을 지정하지 않은 경우   
클로저의 맨 처음에 캡처리스트를 배치하고 in 키워드를 배치한다.
``` swift
lazy var someClosure = {
        [unowned self, weak delegate = self.delegate] in
    // closure body goes here
}
```
<br>

### weak vs unowned
**unowned**   
- 클로저의 캡처를 미소유 참조로 정의하면 `클로저와 클로저가 캡처하는 인스턴스가 항상 서로를 참조하고 항상 동시에 할당이 취소`된다.
- 다른 인스턴스의 수명이 같거나 수명이 더 긴 경우 소유되지 않은 참조를 사용한다.
- 값을 unowned로 표시해도 옵셔널이 되지 않으며 ARC는 unowned 참조의 값을 nil로 설정하지 않는다.
- 해당 인스턴스의 할당이 해제된 후 미소유 참조의 값에 액세스하려고 하면 런타임 오류가 발생한다. -> 참조가 `항상 할당이 해제되지 않은 인스턴스를 참조한다고 확신하는 경우에만 사용`한다.
- `캡처된 참조가 nil이 되지 않는 경우 항상 약한 참조가 아닌 소유되지 않은 참조로 캡처해야 한다고 공식문서에 나와있다.`

**weak**   
- `캡처된 참조가 미래의 어느 시점에서 nil이 될 수 있는 경우` 캡처를 약한 참조로 정의한다.
- 다른 인스턴스의 수명이 더 짧은 경우, 즉 다른 인스턴스를 먼저 할당 해제될 수 있는 경우 약한 참조를 사용한다.   
- `약한 참조는 항상 옵셔널 형식이며, 참조하는 인스턴스의 할당이 취소되면 자동으로 nil`이 된다. -> 이를 통해 클로저 본문 내에서 존재를 확인 가능하다.

```swift
class HTMLElement {
    let name: String
    let text: String?

    lazy var asHTML: () -> String = {
            [unowned self] in
        if let text = self.text {
            return "<\(self.name)>\(text)</\(self.name)>"
        } else {
            return "<\(self.name) />"
        }
    }

    init(name: String, text: String? = nil) {
        self.name = name
        self.text = text
    }

    deinit {
        print("\(name) is being deinitialized")
    }
}

var paragraph: HTMLElement? = HTMLElement(name: "p", text: "hello, world")
print(paragraph!.asHTML())
// Prints "<p>hello, world</p>"
```
위의 예제에 캡처리스트를 적용한 예제이다.   
캡처 리스트는 `[unowned self]이며, 이는 "강력한 참조가 아닌 소유되지 않은 참조로 자신을 캡처"`를 의미한다.   

![](https://docs.swift.org/swift-book/images/closureReferenceCycle02@2x.png)
- 클로저에 의한 self 캡처는 미소유 참조이며 -> 캡처한 HTMLElement 인스턴스를 강력하게 유지하지 않는다.

paragraph 변수의 강력한 참조를 nil로 설정하면 아래 예제의 초기화 해제 메시지 인쇄에서 볼 수 있듯이 HTMLElement 인스턴스의 할당이 해제된다.
```swift
paragraph = nil
// Prints "p is being deinitialized"
```

---
#### 참고 문서
[애플 공식 문서 - closure](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/closures/#Capturing-Values)   
[애플 공식 문서 - ARC](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/automaticreferencecounting/#Strong-Reference-Cycles-for-Closures)   
