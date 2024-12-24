# Week1 - Introducing SwiftUI: Building Your First App
- SwiftUI는 UIKit과는 다르게 Canvas, Code기반 UI 개발 둘 다 자유롭게 전환 가능
- canvas, code에서 command-click으로 다양한 작업 가능
- 큰 변화가 있는 경우 Xcode가 preview를 pause한다
- command-click, extract subview를 사용해 subview 곧바로 빼기 가능
- SwiftUI에서 뷰는 굉장히 가볍기 때문에 로직을 감싸거나 분리하기 위해 뷰를 새로 생성하는 데 거리낌 없어도 된다.

### SwiftUI에서 뷰가 동작하는 방식
- SwiftUI에서 뷰는 View 프로토콜을 준수하는 구조체이다. 
	- UIView처럼 베이스 클래스로부터 상속받는 클래스가 아님
- 이는 뷰가 저장 프로퍼티를 상속받지 않고, 스택에 할당되며 값 타입이라는 의미이다.
```swift
struct RoomDetail: View {
	let room: Room

	var body: some View {
		Image(room.imageName)
			.resizable()
			.aspectRatio(contentMode: .fit)
	}
}
```
- RoomDetail은 구조체인 Room만을 저장하고 있기 때문에 추가적인 할당이나 레퍼런스 카운팅 오버헤드가 없다.
- SwiftUI는 뷰 계층을 효율적인 자료구조로 적극적으로 축소시켜 렌더링한다.
- 따라서 단일 목적의 작은 뷰를 자유롭게 많이 사용하는 것이 좋다.
- SubView를 추출해 더 작은 구성 요소로 리팩토링하는 것은 런타임 오버헤드가 거의 없다. Apple이 권장하는 방식.

### 뷰는 UI의 일부분을 정의한다
- SwiftUI의 View는 UI의 일부분을 정의
- View 프로토콜은 body 프로퍼티만을 정의함
- 이 body 프로토콜은 또한 View 프로토콜을 준수함
- 작은 뷰들을 합성하여 더 큰 뷰를 만든다. 
- 조립하는 뷰의 렌더링은 해당 뷰의 body의 렌더링일 뿐이다.
- 따라서 body에 breakpoint를 걸었는데 실행이 중단된다면 SwiftUI 프레임워크가 뷰의 새로운 렌더링이 필요하다고 판단한 시점이다.

### 뷰는 뷰의 의존성을 정의한다.
- SwiftUI 프레임워크는 뷰의 새로운 렌더링을 언제 가져와야 하는지 알고 있다. 어떻게?
- SwiftUI의 뷰는 UI를 정의하는 것 뿐만 아니라 뷰의 의존성도 정의한다.
- @State 변수를 가지는 뷰에서는 해당 변수를 위해 SwiftUI가 뷰를 대신해 메모리를 할당한다.
- zoomed 변수의 값은 SwiftUI가 책임지고 관리한다.
```swift
struct RoomDetail: View {
	let room: Room
	@State private var zoomed = false

	var body: some View {
		Image(room.imageName)
			.resizable()
			.aspectRatio(contentMode: zoomed ? .fill: .fit)
			.tapAction { self.zoomed.toggle() }
	}
}
```
<img width="378" alt="Screenshot 2024-06-09 at 9 22 50 PM" src="https://github.com/user-attachments/assets/522b986b-384a-4bf8-a091-027fc215662d" />

- 여기서 zoomed 변수가 변경되면 무슨 일이 일어나는 것일까?
- @State 변수의 특별한 속성 중 하나는 SwiftUI가 그것들이 읽히거나 쓰일 때를 관찰(observe)할 수 있다는 것이다. SwiftUI가 body에서 zoomed가 읽혔다는 것을 알기 때문에, 뷰의 렌더링이 이 변수에 의존하고 있다는 것도 알고 있다. 
- 따라서 변수가 변경되면 SwiftUI가 body에게 새로운 @State 값을 사용해 렌더링할 것을 요청한다.

### Truth는 어디에?
- 전통적인 UI 프레임워크에서는 @State 변수(상태 변수)와 일반 변수와의 구분이 없다.
- 그러나 SwiftUI에서는 이 구분이 명확함
- SwiftUI에서는 사용자가 UI에서 마주칠 수 있는 모든 상태(State), 예를 들어 스크롤 뷰의 오프셋, 버튼의 하이라이트 상태, 네비게이션 스택의 내용 등은 권위 있는 데이터 조각, 즉 Source of Truth에서 파생된다. 
- 일반적으로 이 SoT는 State 변수 또는 모델로 구성되고, 이것들이 합쳐져 앱의 전체 상태를 관리하는 SoT를 형성한다. 
- 각각의 변수들은 SoT 또는 파생된 값(derived value)으로 깔끔하게 분류할 수 있다.

```swift
struct RoomDetail: View {
	let room: Room
	@State private var zoomed = false // Source of Truth

	var body: some View {
		Image(room.imageName)
			.resizable()
			.aspectRatio(contentMode: zoomed ? .fill: .fit)
			.tapAction { self.zoomed.toggle() }
	}
}

struct AspectRatioView: View {
	let contentMode: ContentMode // derived value
	var body: some View { ... }
}
```

- AspectRatioView 타입은 aspectRatio의 정의이다. 
- SoT인 zoomed와 그로부터 파생된 값인 일반 변수 contentMode를 확인 가능
- SwiftUI는 @State 변수가 읽히거나 쓰여질 때를 알 수 있기 때문에, 변경이 있다면 어떤 렌더링을 새로고침 해야 하는지 알고 있다.
- zoomed가 변경되면, SwiftUI는 새로운 AspectRatioView를 밑바닥부터 만들며 contentMode 속성 및 다른 저장 속성들을 덮어쓰며 새로운 body를 요청해 렌더링을 새로고침한다. 
- 이러한 방법을 통해 SwiftUI의 모든 파생 값(derived value)들이 최신 상태를 유지하게 된다.
- 이제 @State 변수를 사용해 SoT를 선언하는 방법을 배웠고, 모든 일반 속성들이 파생 값이라는 것을 이해했다. 
- 또한 Swift는 @Binding이라는 도구를 제공해 읽고 쓸 수 있는 파생 값을 전달할 수 있다.
- 기술적으로는 상수도 완벽하게 읽기 전용 SoT가 될 수 있다. ex) 테스트 데이터
- BindableObject를 사용해 모델의 변경사항을 관찰할 수 있다.

<img width="862" alt="Screenshot 2024-06-09 at 9 47 38 PM" src="https://github.com/user-attachments/assets/e1c93247-36a9-4f1e-9f41-257ea1fcaba4" />


### 의존성 관리는 어려워
- 뷰 자체가 영구적이고 개발자가 뷰를 최신화하기 위해 노력하던 이전의 UI 프레임워크와는 개념이 다르다.
- SwitUI는 뷰가 데이터를 읽을 때 마다 암시적 의존성이 생성된다.
	- 이는 데이터가 변경되면 뷰도 새로운 값을 반영하기 위해 업데이트되어야 하기 때문이다.
	- 그렇제 못하면 버그가 발생한 것!
- SwiftUI는 이런 버그가 발생하지 않도록 적절한 파생 값을 재계산하여 자동으로 의존성을 관리한다.
- 그러나 실제 앱은 의존성이 복잡하게 얽혀있다.

<img width="1114" alt="Screenshot 2024-06-09 at 10 00 51 PM" src="https://github.com/user-attachments/assets/4949be22-c98b-406b-a226-30deae022e72" />


- 또한 그러한 의존성 관계에서 여러 콜백 함수들을 여러 순서로 실행하게 되면 예상치 못한 UI 버그가 발생할 수 있다. ex) 애니메이션 콜백이 예상치 못한 상황에 발생함
- 특히 멀티스레딩 코드에서 이러한 부분을 고려하기 쉽지 않다.
- 따라서 상태가 많아질수록 복잡도는 급격하게 높아진다.

<img width="1015" alt="Screenshot 2024-06-09 at 10 01 25 PM" src="https://github.com/user-attachments/assets/95e4a620-8dab-4843-99dd-0cdeca0866e8" />


- 이런 상황에서는 버그를 피할 수 없다.
- 그래서 UIKit에서 이러한 복잡성을 처리하기 위해 모든 뷰 업데이트 코드를 하나의 메소드에 모으고 이벤트 핸들러 콜백에서 그 하나의 메소드를 호출하는 방법을 선택하는 경우가 있다. 이는 복잡성을 관리하기 좋은 방법이다.
- SwiftUI는 이 방법론에 영감을 받았다.
- UIKit에서 이러한 방법을 구현하려면 뷰 계층 구조에 뷰를 추가하거나 삭제하는 것, 네비게이션 스택을 푸시하거나 팝 하는 것, 또는 테이블 뷰를 업데이트하는 등의 까다로운 경우들을 고려해야 한다. 
- 그러나 SwiftUI의 View 프로토콜에는 body라는 단 하나의 요구사항만 있다. 
- 이는 프레임워크가 호출하는 단일 진입점이고, 호출될 수 있는 순서가 하나뿐이라는 의미이다.
- UI의 변경된 부분에 대해 단순히 새로운 뷰를 가져오는 이 패턴 덕분에 SwiftUI는 UI 불일치를 거의 제거해주고, 우리가 이해 가능한 범위에서 확장될 수 있다. 

### SwiftUI의 4가지 디자인 철학
- 선언적 문법
- 합성 가능한 조각
- 자동적 행동
- 지속적인 상태
- interrupt 가능한 애니메이션
