# Reusing Views Across Flows

해당 챕터는 뷰를 스택에 집어넣든, 다른 뷰 위에 올리든, 다른 뷰에 임베딩하는 등 어떠한 흐름에서도 뷰를 재사용할 수 있도록 하는 데 집중한다.

### Common Problems

- 만약 닫기 버튼의 액션을 구현하고자 하면, 부모 컴포넌트가 이를 어떻게 띄웠는지에 따라 동작이 달라진다.
  - 모달 형태라면 모달을 닫도록 하고, 네비게이션 스택에 있다면 pop 해야 한다.
- 하지만 이를 위해 ViewController가 네비게이션에 대한 부분까지 모두 책임지게 된다면 안티 패턴이 된다.
- 뷰가 네비게이션 스택을 직접 조작함으로써, 화면 전환의 주도권을 쥐게 되어 내부 사정을 너무 많이 알게 된다.
- 한 화면 흐름에 있는 모든 뷰가 각자 네비게이션 스택을 제어한다면 네비게이션의 SSOT가 사라진다.

#### A smarter, yet still problematic, approach

- 네비게이션 책임을 전담 객체인 Coordinator로 위임할 수 있다.
- 뷰가 화면 제어를 맡지 않고 Coordinator에게 그 역할을 넘겼기에, 뷰의 입장에서는 코디네이터가 어떻게 돌아가는지 알 필요 없다.

#### The view decides what the parent should be

- 하지만 이 뷰는 동작하기 위해 반드시 Coordinator를 요구한다는 거대한 전제를 깔고 있다.
- 부모 흐름이 단순 네비게이션 방식을 택하더라도, 단지 이 뷰를 하나 쓰기 위해 Coordinator를 강제해야 한다.
- 이는 하위 뷰가 부모 뷰의 아키텍처 형태를 건드리는 상황을 만든다.

#### The child orders a parent around

- 두 번째 문제는, 버튼을 누르면 무조건 설정 화면을 열어야 한다고 단정한다는 점이다. (Coordinator.openSettings())
- 만약 이 뷰를 온보딩 흐름에 그대로 배치한다면, 계정 생성 전이라면 설정 화면에 접근할 수 있는 권한이 없다.
- 결국 흐름마다 서로 다른 Coordinator 구현체를 넘기거나, Coordinator 내부에 분기문을 넣어야 한다.

### Preparing for a solution

- 앞서 우리는 CoruseView, CourseUI에 의도적으로 네비게이션 기능을 넣지 않음으로써, 여러 네비게이션 흐름에 넣거나 다른 뷰에 임베딩할 수 있도록 하였다.

#### What we need to support

- 뷰를 만들 때 joinCall 처럼 특정 흐름이 나오는 것을 가정하는 메서드명을 짓는 대신, didPressJoinCalll() 처럼 할일을 직접 정할 수 있도록 낮은 추상화를 적용해야 한다.

### Considering various solutions

- 호출자의 입장에서 CourseView를 설계해보자
- 해당 뷰는 탭 바에 들어갈 수도 있고, MarketplaceView에 들어갈 수도, CoursePresenter에 들어갈 수도 있다.
- 네비게이션 트리거에 대한 어떤 방법을 선택하던 간에, 뷰가 네비게이션 흐름에 대해 모르도록 하는 것이 키포인트다.

### Using closures

- 클로저 또는 람다를 사용하면 부모가 어떻게 화면을 이동할지 아는 상태에서 해당 뷰에 주입하는 형태이다.

### A closure implementation

- 전체론적인 개발론에서 진행했듯이 클로저와 람다에 플레이스홀더 메서드를 작성해놓고 하나씩 채워가면 된다.

#### When a parent is a coordinator

- 만약 부모가 Coordinator라면 클로저에 Coordinator의 이동 메서드를 하나씩 주입해줄 것이다.
- 어떻게 화면 전환을 할지는 Coordiantor가 알기 때문에, 자식인 CourseView는 화면 전환에 대한 로직을 몰라도 된다.

#### Cleaning up the call-site

- 호출 메서드를 클로저로 감싸면서 간접 참조 레이어가 추가된다. 이를 메서드를 클로저로 감싸지 않고 직접 넘겨줄 수 있다.

#### Gluing everything togetehr

- CourseView를 사용하는 부모는 어떤 동작을 할지 클로저를 CourseView에 전달하고
- 이는 CourseView 안의 CourseUI에 전달된다.

#### Inside CourseUI

- 이제 클로저를 넘기기 때문에 CourseUI 내에 있는 플레이스홀더는 교체 가능하다.

#### Cleaning up CourseUI

- CourseUI에서 전달받은 메서드를 별도의 메서드로 감싸지 않고 바로 전달 가능하다.

#### Trade-offs when using closures

- 선언형 UI를 사용한 중첩 뷰에서 클로저를 사용하면 보일러 플레이트 코드가 발생할 수 있다.
- 하지만 클로저의 최대 단점은 확장성이 부족하다는 것이다. 매번 액션이 생길때마다 클로저를 사용하면 12개의 액션이 있을 때는 12개의 클로저가 필요하다.
- 액션이 많을 수록 뷰 계층이 깊을수록 보일러 플레이트 코드는 늘어난다.

### Passing a navigation type

- 인터페이스, 클로저를 담은 구조체 처럼 네비게이션을 위한 타입을 도입하는 방법도 있다.

#### A navigation interface

- 앞서 Coordinator도 이에 해당한다.
- Coordinator의 구현체 클래스는 CourseView에 전달되어 네비게이션 흐름을 담당한다.
- 액션마다 클로저를 주입하는 것이 아닌 구현체 하나만 전달하면 된다.

#### The problem of passing a navigation type

- Coordinator와 같은 네비게이션 타입을 사용하면 네비게이션을 처리하기 위한 메서드를 알게 된다.
  - e.g., coordinator.joinCall(), coordinator.openScheduler()
- 그리고 어떤 네비게이션 동작이 있는지 뷰가 너무 많이 가정하게 된다.

#### The name sets expectations

- Coodinator가 아닌 Navigation이라고 이름 붙이면 가정을 완화시킬 수 있다.
- 하지만 여전히 뷰가 모든 액션을 가정한다는 문제점이 있다.
  - e.g., join call을 눌러도 화면 전환이 아니라 alert를 띄울 수도 있다.
- 네비게이션 중의적인 이름을 위해 CourseActions로 바꾸고
- 안의 메서드도 didPressJoinCall 처럼 중의적인 네이밍을 사용해서 해결할 수 있다.
- 적절한 네이밍으로 네비게이션 로직에 대해 뷰가 모르도록 할. ㅜ 있다.

### Dealing with optionality

- 이제 메서드들이 optional일 때는 어떻게 설계가 바뀔지 살펴보자.

#### Designing a declarative solution

- Builder 패턴처럼 필요한 메서드만 체이닝하여 구현이 가능하다.

#### Reasoning about optionality

- 이처럼 선언형 API 설계를 한다면, Optionality를 추론하기 쉬워진다.
  - e.g., .didPressJoinCall 만 체이닝 되어있다면 나머지 메서드는 미구현
- 반대로 구현해야할 메서드를 빼먹을 수도 있지만 다음 방법으로 완화가 가능하다:
  - 사용하지 않을 메서드가 있는 뷰를 렌더링하지 않는다.
  - 따로 메서드를 구현하지 않아도 호출한 디폴트 액션을 지정한다.
  - assetion 을 통해 구현되지 않았다면 크래시를 발생시킨다.
  - 유저가 액션을 실행할 때, 아직 구현되지 않은 메서드가 있으면 디버그 메시지를 노출한다.

#### Embracing optionality

- Compose UI 에서 Modifier 를 몇몇개 적용해도 유효한 것처럼 View에도 동일한 관점이 필요하다.

```swift
CourseView(courseService: courseService, id: id)
    // 해당 메서드 구현 시에만 통화 버튼 노출
    .didPressJoinCall(action: openCallFlow)
    // 해당 메서드 구현 시에만 일정 버튼 노출
    .didPressSchedule(action: openCalendarSchedule)
    // 해당 메서드 구현 시에만 메시지 버튼 노출
    .didPressMessageButton(action: openChat)
    // 해당 메서드 구현 시에만 할일 아이템 노출
    .didPressTodoItem(action: openTodoItemDetails)

```

### Implementing a declarative API

- 필요한 선언형 메서드를 호출하더라도 같은 CourseView 타입 객체를 반환해야 한다.
- didPressJoinCall() 을 구현한다면, 해당 액션을 가진 CourseView의 갱신된 사본을 반환

#### Gluing actions to new instances, Impelmenting all declarative methods

```swift
struct CourseView: View {
    // These actions are now optional.
    var didPressJoinCall: (() -> Void)?
    var didPressSchedule: (() -> Void)?
    var didPressMessageButton: (() -> Void)?
    var didPressTodoItem: ((TodoItem) -> Void)?

    // These methods now accept the action to call once they are triggered.
    func didPressJoinCall(action: @escaping () -> Void) -> CourseView {
        // We make a copy
        var copy = self
        // We wire the new action to this method.
        copy.didPressJoinCall = action
        // We return a copy
        return copy
    }

    func didPressSchedule(action: @escaping () -> Void) -> CourseView {
        var copy = self
        // We set the didPressSchedule action on the copy
        copy.didPressSchedule = action
        return copy
    }

    func didPressMessageButton(action: @escaping () -> Void) -> CourseView {
        var copy = self
        // We set the didPressMessageButton action on the copy
        copy.didPressMessageButton = action
        return copy
    }

    func didPressTodoItem(action: @escaping (TodoItem) -> Void) -> CourseView {
        var copy = self
        // We set the didPressTodoItem action on the copy
        copy.didPressTodoItem = action
        return copy
    }

    // ... rest omitted
}

```

- 필요한 액션들을 property로 두고 이를 구현하면 프로퍼티를 갱신하고 리턴하는 형태가 된다.

#### Connecting to CourseUI, Chaining forever

- 앞선 내용처럼 CourseUI는 클로저를 전달받아, 전달한 클로저에 따라 UI 요소를 표시하도록 한다.
- 동일한 메서드를 여러번 체이닝할 수도 있기 때문에, 오용을 막으려면 같은 메서드가 여러번 반복되면 에러를 발생시킨다거나 하는 처리가 필요하다.

### What we covered

- 뷰가 스스로 닫거나 사라지려면, 부모의 네비게이션 흐름에 관여할 수도 있다.
- 뷰가 직접 네비게이션 관련 메서드를 호출하면 재사용성을 해친다.
- 뷰가 네비게이션에 대해 너무 많이 알면, 다른 뷰안에 넣기가 어려워진다.
- 뷰가 네비게이션에 대한 정보를 모르도록 메서드명을 중의적으로 짓는 것이 중요하다.
- 네비게이션 로직이 담긴 이름을 짓기 보다는, 클릭한 버튼 관련으로 메서드명을 지어라
  - e.g., didPressCrossIcon
  - 부모가 액션에 대한 추상화를 가지도록 해라 e.g., dismissView

#### Closures as actions:

- 뷰로부터 네비게이션 로직을 분리하기 위해 클로저를 사용할 수 있다.
- 많은 클로저가 생기고, 계층이 깊어질수록 보일러 플레이트 코드가 늘어난다.

#### Passing around a navigation type:

- 네비게이션 액션이 늘어날수록, 관련 타입은 만드는 것이 다루기 쉽다.
- 인터페이스도 좋은 방법 중 하나이다.
- 인터페이스의 메서드명을 적절하게 지어서 네비게이션에 대해 너무 많은 것을 알지 않도록 해야 한다.

#### Optional actions:

- 뷰를 재재사용 가능하게 만들려면, 설정할 수 있는 옵션이 더 필요하다.
  - e.g., 특정 버튼을 없앤다던가
- Optional한 액션은 복잡성을 늘리지만 빌더 패턴을 통한 체이닝으로 완화할 수 있다.
- 필요한 Modifier만 설정하더라도 해당 액션을 쓰는 뷰만 렌더링하도록 해서 유효한 뷰를 만들어야한다.



# Crafting Robust and Reusable Navigation Flows

### An authorization flow

- 인증 흐름은 복잡해질 수 있고 다른 흐름 내에 재사용되는 경우도 많다.
- 재사용 가능하게 설계함으로써, 인증이 필요한 다른 기능들에 넣을 수 있다.

#### A flow is born

- 앞으로 이 흐름을 Authorizaiton Flow라고 할 것이다.
- 해당 흐름은 AuthrizationIntroView, PinpadView, AuthorizatinoView를 필요로 한다.
- 각 단계가 끝날 때마다 다음 뷰를 보여줄 수 있어야 한다.
- 앞선 챕터에서 보았듯이 안의 뷰는 네비게이션에 대해 모르되, 해당 뷰들을 사용하는 AuthorizationFlow는 네비게이션 로직에 대해 알도록 해야 한다.

#### Defining the Authorization domain

- AuthorizaitonFlow는 밖에서 보면 Authorizaiton 도메인에 해당한다.
- 호출하는 부모는 Authorization 도메인 밖에 위치하기 때문에, AuthorizationFlow는 호출하는 부모에 대해서는 모른다.

### Gathering requirements: deceptively complex

- 세 화면 밖에 없음에도 기능은 충분히 복잡하다.
- 더 나은 설계를 위해서는, 숨은 구체적인 요구사항들을 알아내야 한다.

#### List of requirements

- 로그인된 유저의 세션을 관리하기 위해 세션에 의존한다.
- 중간에 유저가 로그아웃할 수 있다.
- 핀코드를 입력할 때 브루트 포스 공격은 방지되어야 한다.
- 의심스러운 행위가 감지되고, 유저를 강제로 로그아웃시킬 수 있어야 한다.
- 호출하는 부모의 역할에 따라 완료 후 다른 엔트포인트로 이동해야 한다.
- 첫 화면과 마지막 화면의 문구는 수정 가능해야 한다.
- 미래의 업데이트를 위해 API 버저닝이 지원되어야 한다.

#### First attempt at handing complexities

- 앞서 말한 중간에 로그아웃이라던가 세션 관련 기능을 인터페이스로 분리해보자. 이를 AuthorizationHandler로 칭하겠다.
- 이 인터페이스를 구체화하기 위해 onAuthorizationFailure, onAuthorizationSucceeded 같은 메서드를 정의한다.
- 인터페이스를 제공함으로써, Authorizaiton 도메인 밖의 호출자가 직접 구현하도록 하였다.

### Reviewing the current design

- 인터페이스를 도입하는 것의 이점은 AuthorizationFlow를 간단하게 만들고 화면에서 화면으로 이동하는데 집중하게 만든다는 것이다.
- 가장 큰 이슈는 AuthorizationFlow가 책임져야 할 복잡성에 대해 모른다는 것이다.
- AuthorizationFlow가 관리해야할 복잡성을 부모에게 떠넘기기 때문에, 해당 플로우를 사용하려는 모든 부모 컴포넌트가 책임져야 한다.
- 이를 해결하기 위해 Authorization 도메인 내부에서 기본 구현체를 하나 제공할 수도 있지만, 이는 임시 방편이며 근본적인 해결책은 아니다.

### Why do we need navigation flows, anyway?

- 네비게이션 플로우의 목적은 단순히 여러 화면으로 나누는 것이 아니다.
- 유일한 정답을 쫓기보다는 본질적인 목적을 이해함으로써, 현재와 미래에 나은 설계를 찾는 것이 중요하다.

#### The role of views

- 앞선 단원에서 보았듯이 뷰는 view primitives, view component, feature-specific view로 나뉜다.
- 뒤로 갈수록 높은 추상화를 가지고 비즈니스 로직을 뷰 안에 지니기도 한다.

#### Views rarely are features

- 중요한 건은 뷰 자체로 완성된 기능이 되는 경우는 드물다는 것이다.
- 동영상 뷰를 만들기 위해 카메라 화면과 녹화 버튼을 조합하더라도 이는 화면에 불가하다.
- 기능 자체는 동영상 녹화에 관련된 전체 사용자 경험을 의미하기 때문에 필터나 다시 재생하는 화면 등 추가 화면이 포함될 수 있다.

#### Views in flows

- Feature View나 View component는 폰에서느 보통 화면 전체를 가득 채우고, 태블릿에서는 오버레이나 컨테이너에 표시되는 경향이 있다.
- 이렇게 화면을 가득 채우면 Screen으로 부르기도 하는데, 뷰가 화면 전체를 채우면 플로우를 구성하는 기본 블록 역할을 수행한다.

#### Navigation flows are often features

- 뷰와 플로우의 주요한 차이점은 플로우는 기능 전체를 나타내는 경우가 더 많다는 점이다.
- AuthorizationFlow도 인증 기능 이라고 할 수 있다.
- View Primitives → View Component → Feature views → Navigation flows
  - 뒤로 갈수록 추상화 수준이 높아짐

### A navigation flow is like a story

- 이야기처럼 플로우도 시작, 중간, 결과가 있다.
- 사용한 부모에 따라서 결과 화면도 달라야 하기 때문에 플로우의 성공과 실패 결과를 부모에게 알려야 한다.

#### Many flows can be reduced to a story

- 복잡한 플로우도 하나의 간단한 목적으로 축약할 수 있다.
- 이미지 피커 플로우를 만들면 네트워킹, 캐싱, 사진 선택 등 많은 기능이 필요하지만 목적인 유저가 선택한 이미지를 반환하는 것이다.
- 내부 구현은 복잡하더라도 결국에는 호출자 입장에서는 어떤 이미지를 골랐는지만 알면 된다.

#### Purpose matters; the rest is extra

- 다시 돌아와서 AuthorizoationFlow 사례를 살펴보자.
- 어떻게 인증이 이루어지는지는 호출자가 상관할 부분이 아니다. 알고 싶은 정보는 인증이 성공했냐 실패했냐 뿐이다.
- 목적만 만족한다면 이에 필요한 의존성을 넘겨주는 것은 부가적일 뿐이다.

### Designing the API

- 이제는 이전처럼 인터페이스를 두는 것이 아닌 실패했는지 안했는지 전달하는 콜백만 넘기는 단순한 API로 설계할 수 있다.

#### Re-adding complexities

- API는 단순하게 만들었지만 이제 복잡성을 어디서 다룰지 고민해야 한다.
- 이번에는 복잡성은 인증 도메인에서 다루도록 할 것이다.
- 호출자의 입장에서는 AuthorizationFlow가 복잡성을 다루는 것이고, 만약 호출자마다 다른 타입의 일을 시켜야 한다면
- AuthorizationFlow가 사용할 수 있는 다양한 서브 타입을 도입할 수 있다.

#### Defining internal types

- 복잡한 요구사항들을 다루기 위해 구체적인 타입들을 도입할 수 있다.
  - e.g., RateLimiter, SecureStorage
- 개발하는 과정에서 도메인에 종속된 컴포넌트가 아니라는 것을 깨달으면 도메인 밖으로 손쉽게 분리할 수 있다.
  - e.g., PinpadView (여러 목적에 따라 사용될 수 있어 UI 라이브러리에 편입)

#### Adding dependencies

- 목적에 집중하기 위해 API를 최대한 작게 유지했으나, 플로우가 작업을 완수하려면 Network나 User와 같은 의존성이 필요하다.
- 모든 의존성을 넘겨주는 것보다 핵심적인 의존성만 넘겨서 API를 작게 유지하는 것이 중요하다.

#### Adding configurations

- 또 다른 불가피한 복잡성은 다양한 설정은 지원하는 것이다.
- 인증의 첫 화면과 결과 화면의 문구를 커스텀해야 하는 등 설정을 추가하면 API 표면은 커지게 된다.
- 이를 효과적으로 관리하기 위해 설정을 Config 라는 타입으로 묶어 전달하면 효율적으로 관리할 수 있다.

### Reducing a flow to its purest form

- 플로우가 시작과 종료로만 단순화할 수도 있다면 콜백이나 클로저를 사용할 수도 있고
- 만약에 진행 사항을 공유받고 싶다면 구독 패턴을 사용할 수도 있다.
- API 표면을 단순하게 유지함으로써 호출자의 부담을 줄이고 복잡성을 숨길 수 있다.

### What we covered

- 플로우는 줄곧 전체 기능을 나타내기도 한다.
- 뷰는 다른 뷰의 빌딩 블록이 되는 등 여러 역할을 가지고 있다.
- 보통 플로우는 값을 반환하는 엔티티로 추상화할 수 있다.
- 플로우의 API를 설계하기 전에 목적을 생각하고 무엇을 반환할지 생각해야 한다.
- 플로우를 하나의 이야기로 생각하는 것이 좋다.
  - 시작, 중간, 끝이라면
  - 호출자는 시작과 끝에 관심있고, 플로우가 작동하는 중간 과정은 신경쓰지 않는다.
- 시간이 지나면서 요구사항이 복잡해져도 API 를 작게 유지하기 위해 노력해야 한다.
  - 생성자를 작게 유지하거나, Config 타입을 도입해서