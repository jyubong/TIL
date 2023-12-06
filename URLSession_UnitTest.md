## URLSession Unit Test

### URLSession 네트워킹
URLSession API는 iOS에서 네트워킹을 구현하는 가장 기본적인 방법이다.
> URLSession : An object that coordinates a group of related, network data transfer tasks.

![image](https://github.com/jyubong/TIL/assets/126065608/183d46a3-7f47-4fe7-8514-ee4ecf852a4b)
- URLSession 인스턴스를 만들고
- URLSession의 [dataTask(with:)](https://developer.apple.com/documentation/foundation/urlsession/1411554-datatask) 메서드를 호출해 URLSessionDataTask를 생성한다.
> `URLSessionDataTask`는    
URLSessionTask의 구체적인 하위 클래스로 `다운로드한 데이터를 메모리의 앱에 직접 반환하는 URL 세션 작업`이다.
- Task는 일시 중단된 상태로 만들어지므로 `resume()`을 호출하여 시작한다.

<br>

다음은 영화 API를 활용해 데이터를 가져오도록 구현한 코드이다.
```swift
func fetchData<T: Decodable>(url: String, completion: @escaping (T?, Error?) -> Void) {
    guard let url = URL(string: url) else {
        completion(nil, FetchError.invalidURL)
        return
    }
    
    URLSession.shared.dataTask(with: url) { data, response, error in
        if let error = error {
            completion(nil, error)
            return
        }
        
        guard let httpResponse = response as? HTTPURLResponse,
            (200...299).contains(httpResponse.statusCode) else {
            completion(nil, FetchError.invalidResponse)
            return
        }
        
        guard let data = data, let movie = try? JSONDecoder().decode(T.self, from: data) else {
            completion(nil, FetchError.invalidData)
            return
        }
        
        completion(movie, nil)
    }.resume()
}
```
데이터를 가져오는 가장 간단한 방법은 `completion handler`를 사용하는 데이터 작업을 만드는 것이다.   
Task는 서버의 응답, 데이터 및 가능한 오류를 사용자가 제공하는 completion handler 블록으로 전달한다.   

<br>

```swift
func dataTask(
    with url: URL,
    completionHandler: @escaping @Sendable (Data?, URLResponse?, Error?) -> Void
) -> URLSessionDataTask
```
dataTask(with:completionHandler:)는 data, response, error를 completionHandler로 전달하고 있다.

1. `error` 매개 변수가 `nil`인지 확인한다. 그렇지 않은 경우 전송 오류가 발생한 것이기때문에 오류를 처리하고 종료한다.
2. `response` 매개 변수를 검사하여 상태 코드가 성공을 나타내고 그렇지 않은 경우 서버 오류를 처리하고 종료한다.(MIME 형식이 예상 값인지 확인도 가능함)
3. 필요에 따라 `data` 인스턴스를 사용한다.

```swift
URLSession.shared.dataTask(with: url) { data, response, error in
    if let error = error {
        completion(nil, error)
        return
    }
    
    guard let httpResponse = response as? HTTPURLResponse,
        (200...299).contains(httpResponse.statusCode) else {
        completion(nil, FetchError.invalidResponse)
        return
    }
    
    guard let data = data, let movie = try? JSONDecoder().decode(T.self, from: data) else {
        completion(nil, FetchError.invalidData)
        return
    }
    
    completion(movie, nil)
}.resume()
```
구현한 코드에서는 error와 response, data 오류를 completion으로 전달하고 있다.

이외에도 아래와 같이 result type을 활용해 오류를 처리하는 방법도 있으며,    
이러한 방법 활용시 발생하는 문제점을 해결하기 위한 concurrency를 활용하는 방법도 있다. (해당 내용에 대해서는 다음에 다루는 것으로🤔)
```swift
URLSession.shared.dataTask(with: url) { data, response, error in
    if let error = error {
        completion(.failure(error))
        return
    }
    
    guard let httpResponse = response as? HTTPURLResponse,
          (200...299).contains(httpResponse.statusCode) else {
        completion(.failure(FetchError.invalidResponse))
        return
    }
    
    guard let data = data, let movie = try? JSONDecoder().decode(Movie.self, from: data) else {
        completion(.failure(FetchError.invalidData))
        return
    }
    
    completion(.success(movie))
}.resume()
```

<br>

여기서의 핵심은 네트워킹을 처리하는 방법보다는 이렇게 구현된 `URLSession 네트워킹을 어떻게 Test를 할것인가`에 중점을 두고자 한다.
`fetchData(url:completion:)`을 테스트하는 방법은 여러가지가 있을 수 있는데, 여기에서는
1. protocol을 이용한 Stub Test Double
2. URLProtocol subclassing    

두 가지를 다루어보았다.


<br>

### protocol을 이용한 Stub Test Double
**Stub란?**   
Test Doubles 중에 하나로 함수가 항상 미리 정의된 데이터 세트를 반환하는 객체이다.(테스트에서 호출된 요청에 대해 미리 준비해둔 결과를 제공)   
stub는 서버 오류나 네트워크 연결 오류 등 실제 생활에서 달성하기 어려운 특정 조건을 확인하려는 경우 특히 유용하다.

<br>

여기서의 핵심은 `미리 정의된 데이터 세트를 반환`한다는 점이다.    
미리 예상되는 결과를 만들어 놓고 test하는 대상이 반환하는 결과가 예상과 동일한지를 비교하는 원리!    
stub를 활용해서 test를 위한 객체를 만들어 보자.

<br>

```swift
typealias DataTaskCompletionHandler = (Data?, URLResponse?, Error?) -> Void

protocol URLSessionProtocol {
    func dataTask(with url: URL,  completionHandler: @escaping DataTaskCompletionHandler) -> URLSessionDataTask
}

extension URLSession: URLSessionProtocol { }
```
- 의존성 주입을 위해 URLSessionProtocol을 정의하고 extension을 이용하여 URLSession에 채택시켜준다.
- 이때 URLSession에는 이미 dataTask 메서드가 구현되어 있기 때문에 따로 구현해야 할 내용은 없다.

URLSessionProtocol을 채택한 StubURLSession을 만들어보자.
```swift
struct DummyData {
    let data: Data?
    let response: URLResponse?
    let error: Error?
    var completionHandler: DataTaskCompletionHandler? = nil

    func completion() {
        completionHandler?(data, response, error)
    }
}
```
먼저 `우리가 미리 정의할 데이터 세트를 가지고 있을 DummyData`를 만들었다.

```swift
class StubURLSessionDataTask: URLSessionDataTask {
    var dummyData: DummyData?

    init(dummy: DummyData?, completionHandler: DataTaskCompletionHandler?) {
        self.dummyData = dummy
        self.dummyData?.completionHandler = completionHandler
    }

    override func resume() {
        dummyData?.completion()
    }
}
```
- URLSessionDataTask를 subclassing한 StubURLSessionDataTask를 만들었다.   
- StubURLSessionDataTask는 위에서 정의한 DummyData를 가지고 있으며, resume()을 override하여 해당 메서드 실행시 completionHandler가 실행되도록 하였다.   

```swift
class StubURLSession: URLSessionProtocol {
    var dummyData: DummyData?

    init(dummy: DummyData) {
        self.dummyData = dummy
    }

    func dataTask(with url: URL, completionHandler: @escaping DataTaskCompletionHandler) -> URLSessionDataTask {
        return StubURLSessionDataTask(dummy: dummyData, completionHandler: completionHandler)
    }
}
```
URLSessionProtocol을 채탁하는 StubURLSession을 만들었다.   
필수 구현 메서드인 `dataTask(with url:completionHandler:r)`을 구현하였다. 이 메서드에서는 우리가 만들어준 StubURLSessionDataTask를 반환하도록 한다.

<details>
<summary>StubURLSession 전체코드</summary>

```swift
import Foundation

struct DummyData {
    let data: Data?
    let response: URLResponse?
    let error: Error?
    var completionHandler: DataTaskCompletionHandler? = nil

    func completion() {
        completionHandler?(data, response, error)
    }
}

class StubURLSessionDataTask: URLSessionDataTask {
    var dummyData: DummyData?

    init(dummy: DummyData?, completionHandler: DataTaskCompletionHandler?) {
        self.dummyData = dummy
        self.dummyData?.completionHandler = completionHandler
    }

    override func resume() {
        dummyData?.completion()
    }
}

class StubURLSession: URLSessionProtocol {
    var dummyData: DummyData?

    init(dummy: DummyData) {
        self.dummyData = dummy
    }

    func dataTask(with url: URL, completionHandler: @escaping DataTaskCompletionHandler) -> URLSessionDataTask {
        return StubURLSessionDataTask(dummy: dummyData, completionHandler: completionHandler)
    }
}

```
</details>

<br>

그리고 테스트할 NetWorkManager에서 urlSession에 의존성주입을 할 수 있도록 init을 추가 구현해주었다.  
```swift
struct NetworkManager {
    private let urlSession: URLSessionProtocol
    
    init(urlSession: URLSessionProtocol = URLSession.shared) {
        self.urlSession = urlSession
    }

    func fetchData<T: Decodable>(url: String, completion: @escaping (T?, Error?) -> Void) {
//(생략)
        
        urlSession.dataTask(with: url) { data, response, error in
            if let error = error {
                completion(nil, error)
                return
            }
//(생략)
        }.resume()
    }
}
```

<br>

**이제 URLSession을 대체할 StubURLSession을 만들어 주었으니 본격적으로 test를 해보도록 하자!**

<br>

```swift
  let api = "https://kobis.or.kr/kobisopenapi/webservice/rest/boxoffice/searchDailyBoxOfficeList.json?key=f5eef3421c602c6cb7ea224104795888&targetDt=20220105"
  
  guard let url = URL(string: api) else {
      XCTFail()
      return
  }
  
  guard let data = TestMovieJsonData.json.data(using: .utf8) else {
      XCTFail()
      return
  }
  
  let response = HTTPURLResponse(url: url, statusCode: 200, httpVersion: nil, headerFields: nil)
  let dummy = DummyData(data: data, response: response, error: nil)
  let stubURLSession = StubURLSession(dummy: dummy)
```
위의 코드는 데이터를 미리 정의하여 세트를 만들어주는 과정이다.   
- 미리 만들어서 저장해둔 json 데이터(TestMovieJsonData.json)를 `data`로 변환 시켜주고
- 200 staus를 반환하는 `response`를 만들어 주었다.
- data와 response, error를 가진 `dummy 데이터 세트`를 만들어주고
- 이 데이터 세트를 가진 `stubURLSession`을 최종적으로 만들었다!

<br>

```swift
sut = .init(urlSession: stubURLSession)

let promise = expectation(description: "")
let expectation: Movie? = try? JSONDecoder().decode(Movie.self, from: data)
var result: Movie?

// when
sut.fetchData<Movie>(url: api) { movie, error  in
    result = movie
    // then
    XCTAssertEqual(result, expectation)

    promise.fulfill()
}
wait(for: [promise], timeout: 10)
```
- 우리가 미리 정의한 데이터를 가진 `stubURLSession을 networkManager에 주입`을 해준다.
- 그리고 `fetchData에서 만들어진 Movie와 기대되는 Movie 값이 같은지 확인`하면 test는 끝!

<details>
<summary>Test 전체 코드</summary>

```swift
import XCTest
@testable import BoxOffice

final class NetworkManagerTests: XCTestCase {
    var sut: NetworkManager!

    func test_fetchDailyBoxOffice() {
        // given
        let api = "https://kobis.or.kr/kobisopenapi/webservice/rest/boxoffice/searchDailyBoxOfficeList.json?key=f5eef3421c602c6cb7ea224104795888&targetDt=20220105"
        
        guard let url = URL(string: api) else {
            XCTFail()
            return
        }
        
        guard let data = TestMovieJsonData.json.data(using: .utf8) else {
            XCTFail()
            return
        }
        
        let response = HTTPURLResponse(url: url, statusCode: 200, httpVersion: nil, headerFields: nil)
        let dummy = DummyData(data: data, response: response, error: nil)
        let stubURLSession = StubURLSession(dummy: dummy)
        
        sut = .init(urlSession: stubURLSession)

        let promise = expectation(description: "")
        let expectation: Movie? = try? JSONDecoder().decode(Movie.self, from: data)
        var result: Movie?
        
        // when
        sut.fetchData<Movie>(url: api) { movie, error  in
            result = movie
            // then
            XCTAssertEqual(result, expectation)
        
            promise.fulfill()
        }
        wait(for: [promise], timeout: 10)
    }
}
```
</details>

<br>

일단 테스트를 구현하기에 성공을 하였다.   
하지만! 여기에는 문제점이 한가지 있었다.   
![image](https://github.com/jyubong/TIL/assets/126065608/e589fa45-9f20-4cd8-97b9-496885059ca3)
바로 `URLSessionDataTask의 init()이 deprecated` 되었다는 것!!!!
실제로 URLSessionTask와 그 하위클래스들의 init()이 모드 deprecated 되었다.   
> 이 초기화 프로그램을 직접 사용하지 마세요. 대신 URLSession에서 작업을 생성하세요.

<br>

히히.. 그러면 다른 방법을 찾아봐야지 하면서 공부하게 된 방법이 바로 URLProtocol subclassing이다.

<br>

### URLProtocol subClassing
**URLProtocol이란?**   
프로토콜로 오해할 수 있지만! 프로토콜별 URL 데이터 로드를 처리하는 추상 클래스이다.    
즉, 여기서 Protocol은 Swift에서 채택하여 사용할 수 있는 그 프로토콜이 아니니 헷갈리지 말것(사실 나도 헷갈렸었음...)   

<br>

> 추상클래스란?    
> 기본적인 행동을 정의하고, 그 행동을 어떻게 수행할지는 하위 클래스에게 맡기는 클래스

<br>

```swift
class URLProtocol : NSObject
```
공식문서에는 URLProtocol로 직접 인스턴스를 만들지말고, 서브클래스를 만들라고 나와있다.   

<br>

보통 우리가 알고 있는 URLSession으로 networkRequest를 처리하는 과정은 다음과 같다.   
1. URLSessionConfigruation(생략하기도) -> URLSession -> URLSessionDataTask 생성   
2. Request   
3. Request에 맞는 Response   

![image](https://github.com/jyubong/TIL/assets/126065608/72c3fa10-8484-4b27-bb95-886806e7c688)   

WWDC에 따르면 위의 과정 뒤에 좀더 Low-Level API가 하나 더 있다고 하는데, 그것이 바로 `URLProtocol`이다.   
URLProtocol은 **Openning Network Connection, Writing the request, Reading Back Response** 등의 역할을 한다.   
그리고 우리가 Custom한 URLProtocol의 subclass를 URLSessionConfiguration에 넣을 수 있다.   

```swift
configuration.protocolClasses = [TestURLProtocol.self]
```

`protocolClasses`는 **세션에서 요청을 처리하는 추가 프로토콜 서브 클래스의 배열**로 사용자지정 URLProtocol 하위 클래스를 담을 수 있다.   
URLSession은 기본적으로 여러 가지 일반적인 네트워킹 프로토콜을 지원하는데, 이 배열을 사용하여 사용자가 정의한 URLProtocol로 확장할 수 있다.   
URLSession은 request를 처리하기 전에 먼저 기본 프로토콜을 검색한 다음 지정된 요청을 처리할 수 있는 프로토콜을 찾을 때까지 사용자 지정 프로토콜을 확인하는 과정을 거치게 되는데, 이러한 flow를 활용하여 test를 하게 된다.   

<br>

#### Custom URLProtocol
```swift
final class TestURLProtocol: URLProtocol {
    
    override class func canInit(with request: URLRequest) -> Bool {
        return true
    }
    
    override class func canonicalRequest(for request: URLRequest) -> URLRequest {
        return request
    }
    
    override func startLoading() {

    }
    
    override func stopLoading() {}
}
```

URLProtocol을 상속받는 TestURLProtocol을 만든다.   
그리고 아래의 4가지 메서드를 override 해야한다.   
1. `canInit(with:)` : 프로토콜 서브클래스가 지정된 요청을 처리할 수 있는지 여부를 결정한다.
-> `return true`로 해야 활용이 가능하다.
2. `canonicalRequest(for:)` : request를 canonical(표준,기준?)하게 바꾸어 준다.
-> 보통 request를 반환하면 된다고 한다.
3. `startLoading()` : request가 시작되는 메서드로 이 안에서 URLProtocolClient 프로토콜을 통해 URL Load system을 제공한다.
4. `stopLoading()` : request가 끝났을 때 혹은 중지되었을때 처리할 행동을 정의한다.

<br>

공부한 내용을 토대로 프로젝트에 Test를 적용시켜보았다.

<br>

#### TestURLProtocol
```swift
final class TestURLProtocol: URLProtocol {
    static var loadingHandler: ((URLRequest) -> (Data?, URLResponse?, Error?))?
    
    override class func canInit(with request: URLRequest) -> Bool {
        return true
    }
    
    override class func canonicalRequest(for request: URLRequest) -> URLRequest {
        return request
    }
    
    override func startLoading() {
        guard let handler = TestURLProtocol.loadingHandler else { return }
        
        let (data, response, error) = handler(request)
        
        if let data = data, let response = response {
            client?.urlProtocol(self, didReceive: response, cacheStoragePolicy: .notAllowed)
            client?.urlProtocol(self, didLoad: data)
            client?.urlProtocolDidFinishLoading(self)
        } else if let error = error {
            client?.urlProtocol(self, didFailWithError: error)
        }
    }
    
    override func stopLoading() {}
}
```
1. **request를 받아서 (Data?, URLResponse?, Error?)를 반환**하는 loadingHandler 클로저를 생성했다.
2. `startLoading`
   - request를 이용해 loadingHandler를 실행하고 response, data, error를 변수에 저장했다.
   - 받은 response, data, error를 client에게 알려주었다.

#### Unit Test 
``` swift
let url = try XCTUnwrap(URL(string: api))
let data = try XCTUnwrap(TestMovieJsonData.json.data(using: .utf8))

TestURLProtocol.loadingHandler = { request in
    let response = HTTPURLResponse(url: url, statusCode:200, httpVersion: nil, headerFields: nil)
    return (data, response, nil)
}
```
- dummy url과 data를 만들었다.
- TestURLProtocol의 loadingHandler에서 request를 받아 `원하는 data, response, error를 반환`하도록 클로저를 구현하였다.

```swift
let configuration = URLSessionConfiguration.ephemeral
configuration.protocolClasses = [TestURLProtocol.self]

sut = .init(urlSession: URLSession(configuration: configuration))
```
- `URLSessionConfiguration`을 정의하고
- **configuration의 하위 프로토콜 클래스로 TestURLProtocol**을 넣어주었다.
- **Stub URLSession을 networkmanager에 주입**해주었다.

```swift
let expectation: Movie? = try? JSONDecoder().decode(Movie.self, from: data)
var result: Movie?

sut.fetchData(url: api, dataType: Movie.self) { movie, error  in
    result = movie
    //then
    XCTAssertEqual(result, expectation)
}
```
- fetchData(network 처리하는 메서드)를 불러와 handler로 전달된 movie data와 주입했던 데이터가 맞는지 확인해주면 끝!
- error case도 구현된 값들을 달리해주면 가능하다.

<br>

---
### 참고자료
[우아한 형제들 기술블로그](https://techblog.woowahan.com/2704/)   
[포프리님 블로그](https://seob-p.tistory.com/m/18)   
[Youngwoo Lee님 블로그](https://velog.io/@leeyoungwoozz/iOS-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC%EC%97%90-%EC%9D%98%EC%A1%B4%ED%95%98%EC%A7%80-%EC%95%8A%EB%8A%94-Test)   
[애플 공식 문서 - URLProtocol](https://developer.apple.com/documentation/foundation/urlprotocol)   
[야곰 닷넷 - Unit Test](https://yagom.net/courses/unit-test-%ec%9e%91%ec%84%b1%ed%95%98%ea%b8%b0/)   
