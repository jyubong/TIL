## Swift Concurrency

### Swift Concurrency 적용해보기
기존 방식 : Completion Handler를 사용 → 중첩 클로저 작성
```swift
listPhotos(inGallery: "Summer Vacation") { photoNames in
    let sortedNames = photoNames.sorted()
    let name = sortedNames[0]
    downloadPhoto(named: name) { photo in
        show(photo)
    }
}
```

이 코드를 swift concurrency로 변환해보자!   

<br>

#### Step1 : listPhotos(inGallery:)에서 completionHandler 없애기
```swift
func listPhotos(inGallery name: String) async -> [String] {
    let result = // ... some asynchronous networking code ...
    return result
}
```
- 매개 변수 뒤, return 화살표 앞에 `async 키워드` 작성 -> '나는 비동기 함수야'
- throws가 있다면 throws 앞에 작성

<br>

#### Step2 : 비동기함수 호출시 await 키워드 사용하기
비동기 메서드는 해당 메서드가 반환될 때까지 실행이 일시 중단하기때문에   
호출 앞에 `await을 작성하여 일시중단점 표시`⭐️   

<br>
   
> 여기서 잠깐!   
> 비동기 함수는 실행 중인 thread를 포기하고 해당 thread에서 다른 비동기 함수를 실행할 수 있도록 하는 특징이 있다.   
> 즉, thread 제어권을 시스템에게 넘기면서 원래 async 작업에 대한 스케줄도 시스템에게 넘어가게 된다.   
> 이로 인해 비동기 함수가 재개될 때, 해당 함수가 어떤 thread로에서 실행될지 모른다.   
> 이는 이전 thread로 돌아가지 않을 수 있다는 말!   

<br>
   
```swift
let photoNames = await listPhotos(inGallery: "Summer Vacation")
let sortedNames = photoNames.sorted()
let name = sortedNames[0]
let photo = await downloadPhoto(named: name)
show(photo)
```
1. 첫번째 await 실행 → listPhotos(inGallery:) 함수 호출 → 해당 함수가 반환되기를 기다리는 동안 실행을 일시 중단
2. 위 코드가 일시 중단되는 동안 동일한 프로그램의 다른 동시 코드가 실행
    - 장기 실행 백그라운드 작업이 새 사진 갤러리 목록을 계속 업데이트
    - await로 표시된 다음 일시 중단 지점까지 또는 완료될 때까지 코드 실행
3. `listPhotos(inGallery:)`가 반환된 후 이 코드는 해당 시점부터 계속 실행 → photoNames에 반환 값 할당
4. `sortedNames` 및 `name` 은 동기 코드
5. 다음 `await`는 `downloadPhoto(named:)` 함수에 대한 호출을 표시 → 해당 함수가 반환될 때까지 실행을 다시 일시 중지하여 다른 동시 코드를 실행할 수 있는 기회를 제공
6. `downloadPhoto(named:)`가 반환된 후 반환 값이 `photo`에 할당된 다음 `show(_:)`를 호출할 때 인수로 전달

<br>

### await 키워드 알아보자
- await은 `일시 중단 지점`을 나타냄 : 현재 코드가 비동기 함수가 반환되기를 기다리는 동안 실행을 일시 중지할 수 있음을 나타냄
- `스레드 양보(yielding the thread)` 한다 : 현재 스레드에서 코드 실행을 일시 중단하고 대신 해당 스레드에서 다른 코드를 실행
- 실행을 일시 중단할 수 있어야 하므로 프로그램의 특정 위치에서만 비동기 함수를 호출 가능!
    - 비동기 함수, 메서드 또는 속성의 본문에 있는 코드
    - 구조체, 클래스 또는 열거형의 정적 `main()` 메서드에 있는 코드로, `@main`로 표시된 구조체
    - unstructured child task의 코드 → Task {}

 <br>

### 비동기 함수를 병렬로 호출

#### await 사용
`await`를 사용하여 비동기 함수를 호출하면 한 번에 하나의 비동기 함수만 실행된다.   
따라서, 비동기 코드가 실행되는 동안 해당 코드가 완료될 때까지 기다렸다가 다음 코드 줄을 실행된다.

```swift
let firstPhoto = await downloadPhoto(named: photoNames[0])
let secondPhoto = await downloadPhoto(named: photoNames[1])
let thirdPhoto = await downloadPhoto(named: photoNames[2])

let photos = [firstPhoto, secondPhoto, thirdPhoto]
show(photos)
```
- 다운로드가 비동기식이고 진행되는 동안 다른 작업이 수행되도록 하지만 `downloadPhoto(named:)`은 한 번에 하나만 실행
  
<br>

#### async-let 사용
let 앞에 async를 작성한 다음, 상수를 사용할 때마다 await를 작성하면, 각 사진을 독립적으로 또는 동시에 다운로드 가능하다.

```swift
async let firstPhoto = downloadPhoto(named: photoNames[0])
async let secondPhoto = downloadPhoto(named: photoNames[1])
async let thirdPhoto = downloadPhoto(named: photoNames[2])

let photos = await [firstPhoto, secondPhoto, thirdPhoto]
show(photos)
```
- `downloadPhoto(named:)`하기 위한 세 번의 호출이 모두 이전 호출이 완료될 때까지 기다리지 않고 시작
- 코드가 함수의 결과를 기다리기 위해 일시 중단되지 않기 때문에 이러한 함수 호출은 `await`로 표시하지 않음
- 비동기 호출의 결과를 필요로 하므로 세 장의 사진이 모두 다운로드를 마칠 때까지 실행을 일시 중지하기 위해 `await` 사용

<br>

### async-let과 await 적절히 사용하기
- 다음 줄의 코드가 해당 함수의 결과에 따라 달라지는 경우 `await`를 사용하여 비동기 함수를 호출 → 순차적으로 수행되는 작업이 생성
- 나중 코드에서 결과가 필요하지 않는 다면 `async-let`을 사용하여 비동기함수를 호출 → 병렬로 수행할 수 있는 작업 생성
- `await` 및 `async-let` 모두 일시 중단된 동안 다른 코드를 실행할 수 있도록 함
- 일시 중단 지점을 `await`로 표시하여 필요한 경우 비동기 함수가 반환될 때까지 실행이 일시 중지됨을 나타냄

---
### 참고 문헌
[Swift Programming Language Guide - Concurrency](https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/)
