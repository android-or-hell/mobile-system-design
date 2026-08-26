좋아요. **볼드 처리한 내용만 중심으로 남기고**, 설명을 최소화하면 이렇게 정리할 수 있습니다.

# 8 거창한 프레임워크 없이 합리적으로 의존성 주입하기

## 8.1 단순한 해결책

`NetworkClient`를 `CourseService`에 전달하면, `CourseService`가 이를 이용해 `TutorAPI`, `TodoAPI`, `Calendar`를 직접 생성할 수 있습니다.

하지만 `CourseService`는 `NetworkClient`를 직접 사용하지 않습니다. **자신의 의존성을 만들기 위해 전이 의존성인 `NetworkClient`를 알게 된 것**입니다.

> **의존성을 구성하는 일이 정말 `CourseService`의 책임일까요?**
>

---

## 8.2 ABC 문제

ABC 문제는 **타입이 자신이 의존하는 대상의 의존성까지 알게 되는 문제**입니다.

```
CourseService
    ↓
TutorAPI
    ↓
NetworkClient
```

`CourseService`는 `TutorAPI`만 직접 사용하므로 `NetworkClient`까지 알 필요가 없습니다.

전이 의존성까지 알게 되면 하위 타입의 변경이 상위 타입까지 전파되어 코드가 서로 얽히고 경직됩니다.

따라서:

> **각 타입은 자신이 실제로 사용하는 직접 의존성만 알아야 합니다.**
>

---

## 8.3 계층 구조 뒤집기

ABC 문제를 피하기 위해 **의존성 계층을 실제 객체 생성 순서대로 뒤집어 구성**합니다.

의존 관계가 바뀌는 것은 아니며, 단지 생성 순서에 맞게 그래프를 바라보는 것입니다.

```
Transport
→ NetworkClient
→ TutorAPI / TodoAPI / Calendar
→ CourseService
```

의존성이 없는 가장 낮은 타입부터 생성하고, 마지막에 `CourseService`를 생성합니다.

### `AppSetup`

의존성 구성은 `AppSetup`처럼 **애플리케이션 진입점 근처의 명확한 위치**에서 담당합니다.

```swift
let transport = StagingTransport()
let networkClient = NetworkClient(transport: transport)

let tutorAPI = TutorAPI(networkClient: networkClient)
let todoAPI = TodoAPI(networkClient: networkClient)
let calendar = Calendar(networkClient: networkClient)

let courseService = CourseService(
    tutorAPI: tutorAPI,
    todoAPI: todoAPI,
    calendar: calendar
)
```

이제 `CourseService`는 `NetworkClient`를 모르고 **직접 의존성만 전달받습니다.**

---

## 8.4 테스트 환경

테스트에서도 동일한 구조를 사용하되 `MockTransport`만 전달합니다.

```swift
let transport = MockTransport()
```

나머지는 실제 `NetworkClient`, API, `CourseService`를 그대로 사용하므로 **더 많은 실제 배포 코드를 테스트할 수 있습니다.**

---

## 8.5 컴파일러 플래그

컴파일러 플래그는 가능한 한 **애플리케이션의 시작점에 모아둡니다.**

```swift
#if DEBUG
let transport = StagingTransport()
#else
let transport = ProductionTransport()
#endif
```

이렇게 하면 환경에 따른 차이가 여러 타입으로 퍼지지 않습니다.

다만 OS나 플랫폼에 직접 종속되는 조건까지 모두 한곳에 모을 수 있는 것은 아닙니다.

---

## 8.6 핵심 비결

`AppSetup`은 **모든 직접·전이 의존성에 접근하여 전체 구조를 조립**합니다.

```
Transport
→ NetworkClient
→ API
→ CourseService
```

즉, 모든 타입을 **아래에서 위로(bottom-up)** 연결합니다.

의존성을 구성하는 `AppSetup` 자체는 ABC 규칙을 위반하지만, 이것이 `AppSetup`의 역할입니다.

따라서 중요한 것은:

> **ABC 규칙을 깨는 위치를 없애는 것이 아니라, 의존성을 구성하는 명확한 위치로 제한하는 것입니다.**
>

---

## 8.7 더 큰 의존성 트리

앱이 커져도 원칙은 같습니다.

`Marketplace`가 `TutorAPI`, `NetworkClient` 같은 **전이 의존성을 직접 알지 않도록 유지**하고, 가장 깊은 의존성부터 생성하여 연결합니다.

필요하면 의존성 구성 자체도 나눌 수 있습니다.

```swift
setupMarketplace()
setupNetworkClient()
```

또한 인터페이스를 꼭 필요한 경계에만 두면 Mock보다 **실제로 배포되는 코드를 더 많이 테스트하는 방향**으로 유도할 수 있습니다.

다만 이 방식은 모든 의존성을 시작할 때 즉시 생성한다는 단점이 있습니다.

---

## 8.8 시작할 때 만들 수 없는 의존성

모든 의존성을 `AppSetup` 실행 시점에 준비할 수 있는 것은 아닙니다.

예를 들어 `Payments`에 사용자가 런타임에 선택하는 `PaymentProvider`가 필요하다면 앱 시작 시에는 `Payments`를 만들 수 없습니다.

또 사용자가 결제를 하지 않는다면 `Payments` 자체가 끝까지 필요하지 않을 수도 있습니다.

이런 경우 **지연 초기화**가 필요합니다.

---

## 8.9 지연 의존성과 Factory

**지연 의존성은 필요한 의존성을 나중에 만들어 반환하는 함수**입니다.

익명 함수를 직접 전달하기보다 `PaymentsFactory`처럼 명시적인 타입으로 만들 수 있습니다.

```
AppSetup
→ NetworkClient를 미리 Factory에 전달

Marketplace
→ 나중에 PaymentProvider를 전달

PaymentsFactory
→ 두 값을 합쳐 Payments 생성
```

미리 준비할 수 있는 값은 생성자로 전달합니다.

```swift
init(networkClient: NetworkClient)
```

런타임에만 알 수 있는 값은 팩토리 메서드로 전달합니다.

```swift
makePayments(provider: PaymentProvider)
```

따라서:

> **지금 준비할 수 있는 의존성은 생성자에 넣고, 나중에 알 수 있는 값은 팩토리 메서드의 인자로 전달합니다.**