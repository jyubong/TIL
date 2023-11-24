이번 문서는 Responder와 Gesture Event들을 Handle하는 방법과 관련된 apple 공식문서들을 정리해보았다.   
🍎 [Using responders and the responder chain to handle events](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/using_responders_and_the_responder_chain_to_handle_events)   
🍏 [Touches, presses, and gestures](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures)   
🍎 [Handling UIKit gestures](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/handling_uikit_gestures)   
🍏 [Handling touches in your view](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/handling_touches_in_your_view)   

---
## Using Responders and the Responder Chain to Handle Events

### Overview

- 앱은 `responder 객체`를 사용하여 이벤트를 수신하고 처리
- responder 객체는 모두 `UIResponder` 클래스의 인스턴스 : UIView, UIViewController, UIApplication 등
- responder는 이벤트 데이터를 수신하면 이벤트를 처리하거나 다른 응답자 개체로 전달해야 함
- 앱이 이벤트를 수신하면 UIKit은 해당 이벤트를 `첫 번째 응답자`라고 하는 가장 적절한 responder 개체로 자동으로 전달
- 처리되지 않은 이벤트는 active responder chain을 통해 responder에서 responder로 전달

![image](https://github.com/jyubong/TIL/assets/126065608/874fff8f-3271-4219-9312-da4768077f63)


- 위 사진은 이벤트가 응답자 체인을 따라 한 responder에서 다음 responder로 이동하는 방법을 나타냄
    - TextField가 이벤트를 처리하지 않는 경우 UIKit은 TextField의 부모 `UIView` 로 보냄
    - 이벤트를 처리 하지 않는 경우 `UIWindow의 root view`로 이벤트를 보냄
    - root view에서 응답자 체인은 이벤트를 창으로 보내기 전에 `ViewController`로 전환
    - Window에서 이벤트를 처리할 수 없는 경우 UIKit은 `UIApplication` 개체에 이벤트를 전달
    - 만약 해당 객체가 `UIResponder`의 인스턴스이지만 아직 응답자 체인의 일부가 아닌 경우 `ApplicationDelegate`에 전달

<br>

### Determine an event’s first responder

UIKit은 해당 이벤트의 형식에 따라 객체를 이벤트에 대한 첫 번째 응답자로 지정

| 이벤트 유형 | 최초 대응자 |
| --- | --- |
| 터치 이벤트 | 터치가 발생한 뷰 |
| Press events | 포커스가 있는 개체 |
| Shake-motion 이벤트 | 사용자(또는 UIKit)가 지정하는 개체 |
| 원격 제어 이벤트 | 사용자(또는 UIKit)가 지정하는 개체 |
| 메뉴 메시지 편집 | 사용자(또는 UIKit)가 지정하는 개체 |
- 컨트롤(button, stepper 등) : 작업 메시지를 사용하여 연결된 대상 개체와 직접 통신
    - 사용자가 컨트롤과 상호 작용할 때 컨트롤은 대상 개체에 작업 메시지를 보냄
    - 작업 메시지는 이벤트가 아니지만 응답자 체인을 활용 가능
    - 컨트롤의 대상 개체가 `nil`이면 UIKit은 대상 개체에서 시작하여 적절한 작업 메서드를 구현하는 개체를 찾을 때까지 응답자 체인을 트래버스 (ex. UIKit 편집 메뉴는 이 동작을 사용하여 `paste(_:), cut(_:)copy(_:)`와 같은 이름의 메서드를 구현하는 응답자 개체를 검색)
- `제스처 인식기는 view보다 먼저 터치 및 누르기 이벤트를 수신`
- 제스처 인식기가 터치 시퀀스를 인식하지 못하는 경우 UIKit은 터치를 view로 보냄
- view가 터치를 처리하지 않는 경우 UIKit은 응답자 체인을 전달([Handling UIKit gestures](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/handling_uikit_gestures))

<br>

### Determine which responder contained a touch event

- UIKit은 뷰 기반 적중 테스트를 사용하여 터치 이벤트가 발생하는 위치를 확인
- 특히 UIKit은 터치 위치를 뷰 계층 구조의 뷰 개체 범위와 비교
- `UIView`의 [hitTest(_:with:)](https://developer.apple.com/documentation/uikit/uiview/1622469-hittest) 메서드는 뷰 계층 구조를 트래버스하여 터치 이벤트에 대한 첫 번째 응답자가 되는 지정된 터치를 포함하는 가장 깊은 하위 뷰를 찾음

> 터치 위치가 view의 범위를 벗어나는 경우 `hitTest(_:with:)` 메서드는 해당 보기와 모든 하위 보기를 무시   
> view의 [clipsToBounds](https://developer.apple.com/documentation/uikit/uiview/1622415-clipstobounds) 속성이 `true`이면 해당 보기의 경계를 벗어난 하위 보기는 터치가 포함되어 있더라도 반환되지 않음

- 터치가 발생하면 UIKit은 [UITouch](https://developer.apple.com/documentation/uikit/uitouch) 개체를 만들고 뷰와 연결
- 터치 위치 또는 기타 매개 변수가 변경되면 UIKit은 동일한 `UITouch` 개체를 새 정보로 업데이트
- 변경되지 않는 유일한 속성은 view (터치 위치가 원래 보기 외부로 이동하더라도 터치의 `view` 속성 값은 변경되지 않음)
- 터치가 끝나면 UIKit은 `UITouch` 개체를 해제

<br>

### Alter the responder chain

responder 개체의 [next](https://developer.apple.com/documentation/uikit/uiresponder/1621099-next) 속성을 재정의하여 responder 체인을 변경할 수 있음 → 이렇게 하면 다음 응답자는 next에서 반환되는 객체

많은 UIKit 클래스가 이미 이 속성을 재정의하고 다음과 같은 특정 객체를 반환

- `UIView`
    - view가 viewController의 root view인 경우 다음 응답자는 view controller
    - 그렇지 않으면 다음 응답자는 view의 super view
- `UIViewController`
    - ViewController의 view가 window의 root view인 경우 다음 응답자는 window 객체
    - ViewController가 다른 ViewController에 의해 제공된 경우 다음 응답자는 presentation ViewController
- `UIWindow` : window의 다음 응답자는 `UIApplication`
- `UIApplication` : 다음 응답자는 applicationDelegate이지만 appDelegate가 `UIResponder`의 인스턴스이고 뷰, 뷰 컨트롤러 또는 앱 개체 자체가 아닌 경우에만 해당

<br>

## Touches, presses, and gestures

- 표준 UIKit view 및 control을 사용하여 앱을 빌드하는 경우 UIKit은 터치 이벤트(멀티터치 이벤트 포함)를 자동으로 처리
- 사용자 지정 보기를 사용하여 콘텐츠를 표시하는 경우 view에서 발생하는 모든 터치 이벤트를 처리해야 함
- 터치 이벤트를 직접 처리하는 방법 두 가지
    - 제스처 인식기를 사용하여 터치를 추적 ([Handling UIKit gestures](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/handling_uikit_gestures))
    - `UIView` 하위 클래스에서 직접 터치를 추적 ([Handling touches in your view](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/handling_touches_in_your_view))

<br>

## Handling UIKit gestures

- 제스처 인식기(gesture recognizer)는 보기에서 터치 또는 누르기 이벤트를 처리하는 가장 간단한 방법
- 하나 이상의 제스처 인식기를 모든 뷰에 연결할 수 있음
- 제스처 인식기는 해당 뷰에 대해 들어오는 이벤트를 처리 및 해석하고 알려진 패턴과 일치시키는 데 필요한 모든 논리를 캡슐화 → 일치가 감지되면 제스처 인식기는 할당된 대상 개체(보기 컨트롤러, 보기 자체 또는 앱의 다른 개체일 수 있음)에 알림
- 제스처 인식기는 대상 작업 디자인 패턴을 사용하여 알림을 보냄
- ex. [UITapGestureRecognizer](https://developer.apple.com/documentation/uikit/uitapgesturerecognizer) 개체가 보기에서 한 손가락 탭을 감지하면, 응답을 제공하는 데 사용하는 view의 view controller의 작업 메서드를 호출

![](https://docs-assets.developer.apple.com/published/7c21d852b9/0c8c5e29-c846-4a16-988b-3d809eafbb6b.png)

- 제스처 인식기는 불연속 인식기와 연속형의 두 가지 유형으로 제공
    - `개별 제스처 인식기(불연속, discrete)`는 제스처가 인식된 후 작업 메서드를 정확히 한 번 호출
    - 초기 인식 조건이 충족되면 `연속 제스처 인식기`는 작업 메서드를 여러 번 호출하여 제스처 이벤트의 정보가 변경될 때마다 사용자에게 알림.
    - 예를 들어 `UIPanGestureRecognizer` 개체는 터치 위치가 변경될 때마다 작업 메서드를 호출
- Interface Builder에는 각 표준 UIKit 제스처 인식기에 대한 개체가 포함되어 있음
- 사용자 지정 `UIGestureRecognizer` 서브클래스를 나타내는 데 사용할 수 있는 사용자 지정 제스처 인식기 개체도 포함되어 있음
- 제스처 인식기는 view의 응답자 체인에 참여하지 않음

<br>

### Configuring a gesture recognizer
- 제스처 인식기를 구성하려면(스토리보드 방식)
    1. 스토리보드에서 제스처 인식기를 뷰로 끌어오기
    2. 제스처가 인식될 때 호출될 작업 메서드를 구현
    3. 동작 메서드를 제스처 인식기에 연결
        - 제스처 인식기를 마우스 오른쪽 단추로 클릭하고 인터페이스의 적절한 개체에 연결
        - 제스처 인식기의 [addTarget(_:action:)](https://developer.apple.com/documentation/uikit/uigesturerecognizer/1624230-addtarget) 메서드를 사용하여 프로그래밍 방식으로 작업 메서드를 구성할 수도 있음
- 제스처 인식기의 작업 메서드에 대한 일반 형식
    - `@IBAction func myActionMethod(_ sender: UIGestureRecognizer)`
    - 원하는 경우 특정 제스처 인식기 하위 클래스와 일치하도록 매개 변수 형식을 변경할 수 있음
- 제스처 인식기에는 하나 이상의 대상-작업 쌍이 연결 → 여러 대상-작업 쌍이 있는 경우 이산적(개별적, discrete)이며 누적되지 않음

### Responding to gestures
- 제스처 인식기와 연결된 작업 메서드는 해당 제스처에 대한 앱의 응답을 제공
    - 개별 제스처의 경우 동작 메서드는 단추의 동작 메서드와 비슷 : 작업 메서드가 호출되면 해당 제스처에 적합한 작업을 수행
    - 연속 제스처의 경우 작업 메서드는 제스처 인식에 응답할 수 있지만 제스처가 인식되기 전에 이벤트를 추적할 수도 있음 → 이벤트를 추적하면 보다 인터랙티브한 경험을 만들 수 있음
    - 예를 들어 [UIPanGestureRecognizer](https://developer.apple.com/documentation/uikit/uipangesturerecognizer) 개체의 업데이트를 사용하여 앱의 콘텐츠 위치를 변경할 수 있음
- 제스처 인식기의 [state](https://developer.apple.com/documentation/uikit/uigesturerecognizer/1619998-state) 속성은 개체의 현재 인식 상태를 전달
    - [UIGestureRecognizer.State.began](https://developer.apple.com/documentation/uikit/uigesturerecognizer/state/began)
    - [UIGestureRecognizer.State.changed](https://developer.apple.com/documentation/uikit/uigesturerecognizer/state/changed)
    - [UIGestureRecognizer.State.ended](https://developer.apple.com/documentation/uikit/uigesturerecognizer/state/ended)
    - [UIGestureRecognizer.State.cancelled](https://developer.apple.com/documentation/uikit/uigesturerecognizer/state/cancelled)
    - 이 속성들을 사용하여 적절한 작업 과정을 결정
    - 예를 들어, started 및 changed 상태를 사용하여 콘텐츠를 일시적으로 변경하고, ended 상태를 사용하여 해당 변경 사항을 영구적으로 적용하고, cancelled 상태를 사용하여 변경 사항을 취소할 수 있음
    - 작업을 수행하기 전에 항상 제스처 인식기의 `state` 속성 값을 확인
- 특정 유형의 제스처를 처리하는 방법
    - [Handling tap gestures](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/handling_uikit_gestures/handling_tap_gestures)
    - [Handling long-press gestures](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/handling_uikit_gestures/handling_long-press_gestures)
    - [Handling pan gestures](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/handling_uikit_gestures/handling_pan_gestures)
    - [Handling swipe gestures](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/handling_uikit_gestures/handling_swipe_gestures)
    - [Handling pinch gestures](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/handling_uikit_gestures/handling_pinch_gestures)
    - [Handling rotation gestures](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/handling_uikit_gestures/handling_rotation_gestures)
- 제스처 인식기 상태 및 코드에 미치는 영향에 대한 자세한 내용은 [사용자 지정 제스처 인식기 구현을](https://developer.apple.com/documentation/uikit/touches_presses_and_gestures/implementing_a_custom_gesture_recognizer) 참조

<br>

### [UIGestureRecognizer](https://developer.apple.com/documentation/uikit/uigesturerecognizer) 종류
- [UIHoverGestureRecognizer](https://developer.apple.com/documentation/uikit/uihovergesturerecognizer) : 뷰 위의 포인터 움직임을 해석하는 연속 제스처 인식기
- [UILongPressGestureRecognizer](https://developer.apple.com/documentation/uikit/uilongpressgesturerecognizer) : 길게 누르는 동작을 해석하는 연속 동작 인식기
- [UIPanGestureRecognizer](https://developer.apple.com/documentation/uikit/uipangesturerecognizer) : 패닝 제스처를 해석하는 연속 제스처 인식기
- [UIPinchGestureRecognizer](https://developer.apple.com/documentation/uikit/uipinchgesturerecognizer) : 두 번의 터치와 관련된 핀치 제스처를 해석하는 연속 제스처 인식기
- [UIRotationGestureRecognizer](https://developer.apple.com/documentation/uikit/uirotationgesturerecognizer) : 두 번의 터치가 포함된 회전 동작을 해석하는 연속 동작 인식기
- [UIScreenEdgePanGestureRecognizer](https://developer.apple.com/documentation/uikit/uiscreenedgepangesturerecognizer) : 화면 가장자리 근처에서 시작되는 패닝 제스처를 해석하는 연속 제스처 인식기
- [UISwipeGestureRecognizer](https://developer.apple.com/documentation/uikit/uiswipegesturerecognizer) : 하나 이상의 방향으로 스와이프 동작을 해석하는 개별 동작 인식기
- [UITapGestureRecognizer](https://developer.apple.com/documentation/uikit/uitapgesturerecognizer) : 단일 또는 다중 탭을 해석하는 개별 제스처 인식기

![image](https://github.com/jyubong/TIL/assets/126065608/54affada-13f6-4099-bcaa-933e148a61b6)

<br>

## Handling touches in your view

### Overview
- 사용자 지정 보기에서 제스처 인식기를 사용하지 않으려는 경우 보기 자체에서 직접 터치 이벤트를 처리 가능
- view는 응답자이기 때문에 Multi-Touch 이벤트 및 기타 여러 유형의 이벤트를 처리 가능
- 뷰에서 터치 이벤트가 발생했음을 확인하면 뷰의 [touchesMoved(_:with:)](https://developer.apple.com/documentation/uikit/uiresponder/1621107-touchesmoved) , [touchesBegan(_:with:)](https://developer.apple.com/documentation/uikit/uiresponder/1621142-touchesbegan), [touchesEnded(_:with:)](https://developer.apple.com/documentation/uikit/uiresponder/1621084-touchesended) 메서드 호출 → 사용자 지정 보기에서 이러한 메서드를 재정의하여 터치 이벤트에 대한 응답을 제공 가능
- 터치를 처리하기 위해 뷰(또는 응답자)에서 재정의하는 메서드는 터치 이벤트 처리 프로세스의 여러 단계에 해당
    - 손가락(또는 Apple Pencil)이 화면을 터치하면 UIKit은 UITouch 개체를 만들고 터치 위치를 적절한 지점으로 설정
    - [phase](https://developer.apple.com/documentation/uikit/uitouch/1618113-phase) 속성을 [UITouch.Phase.began](https://developer.apple.com/documentation/uikit/uitouch/phase/began) 로 설정
    - 같은 손가락이 움직이면 UIKit은 터치 위치를 업데이트하고 터치 개체의 `phase` 속성을 [UITouch.Phase.moved](https://developer.apple.com/documentation/uikit/uitouch/phase/moved) 로 변경
    - 사용자가 화면에서 손가락을 떼면 UIKit은 `phase` 속성을 [UITouch.Phase.ended](https://developer.apple.com/documentation/uikit/uitouch/phase/ended) 변경 및 터치 시퀀스가 종료

![](https://docs-assets.developer.apple.com/published/7c21d852b9/08b952fe-6f46-41eb-8b8a-4830c1d48842.png)

- 마찬가지로 시스템은 진행 중인 터치 시퀀스를 언제든지 취소할 수 있음
    - 예를 들어 수신 전화가 앱을 중단하는 경우, UIKit은 `touchesCancelled(_:with:)` 메서드를 호출하여 뷰에 알림 → 이 메서드를 사용하여 뷰의 데이터 구조에 필요한 정리를 수행
- UIKit은 화면을 터치하는 각 새 손가락에 대해 새 `UITouch` 개체를 만듬 → 터치 자체는 현재 [UIEvent](https://developer.apple.com/documentation/uikit/uievent) 개체와 함께 제공
- UIKit은 손가락에서 시작된 터치와 Apple Pencil에서 시작된 터치를 구분하며 각각을 다르게 처리 가능

> 기본 구성에서 뷰는 두 개 이상의 손가락이 뷰를 터치하는 경우에도 이벤트와 연결된 첫 번째 `UITouch` 개체만 받음
추가 터치를 받으려면 뷰의 [isMultipleTouchEnabled](https://developer.apple.com/documentation/uikit/uiview/1622519-ismultipletouchenabled) 프로퍼티를 true로 설정해야 함 → Interface Builder에서 Attributes 인스펙터를 사용하여 이 프로퍼티를 구성할 수도 있음
> 

<br>

---

### touchesBegan, touchesMoved, touchesEnded 재정의시 주의사항

- 많은 UIKit 클래스가 이 메서드를 재정의하고 이를 사용하여 해당 터치 이벤트를 처리 → 이 메서드들의 기본 구현은 메시지를 응답자 체인 위로 전달됨
- 고유한 하위 클래스를 만들 때 `super`를 호출하여 직접 처리하지 않는 이벤트를 전달할 수 있음
- `super`(일반적인 사용 패턴)를 호출하지 않고 이 메서드를 재정의하는 경우 구현에서 아무 작업도 수행하지 않더라도 터치 이벤트를 처리하기 위한 다른 메서드도 재정의해야 함
- **즉! 재정의하여 원하는 이벤트로 처리하려면 super를 미호출, 직접 처리하지 않으려면 super를 호출 → 다른 메서드들도 재정의 필요!**

<br>

### UITouch

화면에서 발생하는 터치의 위치, 크기, 움직임 및 힘을 나타내는 객체

```swift
@MainActor
class UITouch : NSObject
```

- 이벤트 처리를 위해 응답자 객체에 전달된 `UIEvent` 개체를 통해 Touch 객체에 액세스
- Touch의 프로퍼티, 메서드
    - [var view: UIView?](https://developer.apple.com/documentation/uikit/uitouch/1618109-view) , [var window: UIWindow?](https://developer.apple.com/documentation/uikit/uitouch/1618126-window) : 터치가 발생한 보기 또는 창
    - [func location(in: UIView?) -> CGPoint](https://developer.apple.com/documentation/uikit/uitouch/1618116-location) , [func preciseLocation(in: UIView?) -> CGPoint](https://developer.apple.com/documentation/uikit/uitouch/1618134-preciselocation) : 보기 또는 창 내에서 터치의 위치
    - [var majorRadius: CGFloat](https://developer.apple.com/documentation/uikit/uitouch/1618106-majorradius) : 터치의 대략적인 반경
    - [var force: CGFloat](https://developer.apple.com/documentation/uikit/uitouch/1618110-force) : 터치의 힘(3D Touch 또는 Apple Pencil을 지원하는 기기)
    - 터치가 발생한 시점을 나타내는 타임스탬프, 사용자가 화면을 탭한 횟수를 나타내는 정수, 터치가 시작, 이동 또는 종료되었는지 또는 시스템이 터치를 취소했는지 여부를 설명하는 상수 형식의 터치 단계가 포함

<br>

### addGestureRecognizer(_:)

제스처 인식기를 뷰에 연결

```swift
func addGestureRecognizer(_ gestureRecognizer: UIGestureRecognizer)
```

- 제스처 인식기를 뷰에 연결하면 표현된 제스처의 범위를 정의하여 해당 뷰 및 모든 하위 뷰에 대해 히트 테스트된 터치를 수신하게 됨
- 뷰는 제스처 인식기에 대한 강력한 참조를 설정
