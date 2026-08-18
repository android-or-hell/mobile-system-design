# 08 Sane DI 정리 — 그래프를 뒤집어 ABC problem을 봉인하고, 위반은 AppSetup 한 곳에 가둔다

> Mobile System Design 시리즈 Book 0 (Fundamentals) 의 여덟 번째 챕터.
> 핵심 질문 — *"타입이 자신의 transitive dependency(의존성의 의존성)를 몰라도 되게 값을 전달하려면, 무엇을 어떤 순서로 조립해야 하는가?"*

---

## 도입 — 지루함이 미덕이다

챕터 에피그래프가 이 챕터 전체의 미학을 한 줄로 요약한다.

> *"훌륭한 dependency 설정은 지루하게 느껴진다 — 그것이 작동하고 있다는 것을 아는 방법이다."*

**이 챕터의 위치.** §7(DI Foundations)의 도입부는 *"다음 챕터에서 구현을 시작하기 전에, 여기서 기초를 다룰 것이다"* 라고 예고했다. §7이 이론편(왜 값을 전달해야 하는가, 무엇을 피해야 하는가)이었다면, **§8은 그 약속을 갚는 구현편이다.** §7.4가 결론만 내고 미룬 "컴파일러 플래그를 중앙화하자"와 §7.5.5가 결론만 내고 미룬 "처음부터 기능에서 DI를 지원하자"가 여기서 실제 코드로 완성된다.

> **In this chapter** — "바닐라 코드를 사용한 dependency injection 기법" · "dependency tree를 설정하는 방법" · "코드를 분리된 상태로 유지하면서 transitive dependency를 다루는 방법" · "인터페이스 수를 제한하면서도 코드를 분리된 상태로 유지하는 방법" · "깊이 중첩된 의존성을 다루는 방법" · "lazy loading 기법과 factory를 활용해 모든 의존성을 미리 생성하는 것을 피하는 방법" · "여러 위치에서 주입되는 의존성을 다루는 방법" · "미리 만들 수 없는 의존성을 설정하는 방법"

> *"이전 챕터에서 dependency injection의 use case와 왜 그것이 필요한지 다뤘다. 이 챕터에서는 우리 자신의 코드베이스, 구체적으로 Course 기능에 dependency injection을 적용하기 시작할 것이다."*
>
> *"값을 전달하는 단순한 바닐라 기법을 적용할 것이다. 그렇게 함으로써 하나하나 해결할 과속방지턱에 부딪힐 것이다."*

**챕터가 미리 보여주는 지도가 정확히 이 문서의 목차다.**

> *"가장 흔한 이슈 중 하나는 'ABC problem'이다. 클래스 A가 클래스 B를 소유하고, B가 클래스 C를 소유하는 상황. 조심하지 않으면 C를 A에 전달하게 되는데, A가 C를 인지하기를 원하지 않으므로 문제이다. (…) 팀이 이 시나리오에서 형식적인 해결책에 도달하려 할 수 있다. 그러나 현실에서는 구조를 뒤집어 이를 해결할 수 있으며, 이는 이 챕터에서 광범위하게 다룰 것이다."*
>
> *"우리가 직면할 또 다른 문제는 더 많은 수의 타입을 통해 의존성을 전달하는 것이며, 이 챕터가 그것을 다룰 것이다. 때때로 우리는 모든 의존성이 사용 가능하지 않을 수 있어 타입의 인스턴스를 항상 만들 수 없는 문제가 있다. 그 시나리오를 다루기 위해 factory 같은 lazy dependency 기법을 사용할 것이다."*

그리고 서드파티 프레임워크 없이 간다는 §7.1의 약속이 여기서 재확인된다 — *"몇 가지 기법으로 '제정신으로' 무언가를 '전달'하는 데 도움이 되는 또 다른 서드파티 솔루션을 필요로 하지 않을 수 있다는 것을 보게 될 것이다."* 챕터 제목 "Sane DI"의 "sane"이 정확히 이 지점을 가리킨다 — 마법 없이도 제정신을 유지할 수 있다는 것.

---

## § 8.1 순진한(naive) 해법 — 직관적이지만 함정이 있다

**전략.** 저자는 곧바로 정답으로 가지 않는다. *"더 나은 접근법으로 넘어가기 전에, 직면할 문제에 대한 더 깊은 이해를 얻기 위해 naive 해법으로 시작하자."* §7.5.1이 싱글턴의 편리함을 먼저 인정했던 것과 같은 화법 — **틀린 해법을 성실하게 보여줘야, 그것이 왜 틀렸는지가 다음 절에서 선명해진다.**

> *"naive 해법은 다음과 같다 — NetworkClient를 CourseService에 전달한다. 이는 CourseService가 어떤 서버에 연결되든 어떤 NetworkClient 인스턴스든 받을 수 있게 해준다. 그런 다음 CourseService는 NetworkClient를 사용해 자체 의존성 — 즉 TodoAPI, TutorAPI, Calendar — 의 인스턴스를 만들 수 있다."*

```swift
final class CourseService {

    private let tutorAPI: TutorAPI
    private let todoAPI: TodoAPI
    private let calendar: Calendar

    // NetworkClient가 전달된다.
    init(networkClient: NetworkClient) {
        // 이제 networkClient를 사용해 모든 속성을 인스턴스화한다
        self.tutorAPI = TutorAPI(networkClient: networkClient)
        self.todoAPI = TodoAPI(networkClient: networkClient)
        self.calendar = Calendar(networkClient: networkClient)
    }

    // ... snip
}
```
*(Listing 8.1 — transitive dependency인 NetworkClient를 CourseService에 전달)*

> *"NetworkClient 인스턴스를 전달하는 이유는 그것이 production, staging, 또는 테스팅용 mock transport이든 transport와 함께 미리 구성되어 오기 때문이다."*
>
> *"이 접근법의 이점은 직관적이라는 점이다. NetworkClient를 전달하므로 CourseService가 자체 의존성을 설정할 수 있다."*
>
> *"함정은 CourseService가 이제 NetworkClient에 대해 안다는 점이다. 그러나 CourseService가 작동하기 위해 NetworkClient를 사용하지 않는다는 점에 주목하라. 그것은 단지 transitive dependency를 설정하기 위해 NetworkClient를 사용하고 있을 뿐이다."*

마지막 문장이 이 절 전체의 핵심이다. **NetworkClient가 나쁜 게 아니라, CourseService가 그것을 "쓰지도 않으면서 안다"는 게 문제다.** 그래서 저자는 코드가 아니라 질문으로 절을 닫는다.

> *"생각해보자 — 의존성을 설정하는 것이 정말 CourseService의 책임인가?"*

이 질문에 "아니오"라고 답하는 것이 챕터 전체의 방향이다. **CourseService의 책임은 자기 의존성을 만드는 게 아니라 그저 받는 것**이어야 한다. 그 책임을 누가 대신 지는지가 §8.3~§8.6에서 밝혀진다.

---

## § 8.2 깊이 중첩된 의존성 — ABC problem에 이름 붙이기

> *"방금 마주친 문제는 우리가 'ABC problem'이라고 부를 수 있는 것이다. 그것은 클래스 A가 클래스 B에 의존하고, 클래스 B가 클래스 C에 의존하지만, 우리가 클래스 A가 클래스 C에 대해 알기를 원하지 않는 시나리오이다."*

```
   추상        A ──depends on──▶ B ──depends on──▶ C
                                                    ▲
                                                    │
                                        A가 알아서는 안 되는 관계

   구체        CourseService ──▶ TodoAPI ──▶ NetworkClient
                     ▲
              여기서 멈춰야 한다 — NetworkClient까지 넘어가면 안 된다
```
*(Figure 8.1·8.2를 하나로 합쳐 다시 그림)*

> *"CourseService는 NetworkClient를 직접 사용하지 않는데, 왜 그것에 대해 알아야 하는가? DI에 더 깊이 생각하지 않으면, NetworkClient를 CourseService에 단순히 전달함으로써 ABC problem을 도입하게 될 것이며, 이것이 이전 섹션의 상황이었다."*
>
> *"타입이 transitive dependency — 의존성의 의존성이라고도 알려진 — 를 인지하는 상황을 피해야 한다."*

**연쇄가 어떻게 자란는지 예시로 보여준다.**

> *"예를 들어, TutorAPI가 이제 initializer에 NetworkClient와 User가 필요하다고 하자. 이것을 naive 해법으로 작동시키려면, User를 CourseService에 전달해야 하고, CourseService의 initializer 본문을 업데이트해서 User를 TutorAPI에 전달하도록 해야 할 것이다."*
>
> *"CourseService는 이제 그 의존성과 함께 업데이트되어야 한다. 말하자면 그것이 자신의 속성의 속성을 너무 잘 인지하고 있다."*

**정도의 문제라는 것도 인정한다.**

> *"이것이 일어나도 세상의 끝은 아니다. 사실 코드베이스에서 여기저기 일어나는 것이 정상이지만, 기본적으로 모든 곳에서 일어나면 코드 냄새로 간주할 수 있다. 그러나 항상 이 접근법을 계속 취한다면, 시간이 지남에 따라 모든 의존성이 필요한 곳 어디에나 전달되게 될 것이다. 이는 우리가 코드베이스 전체에 걸쳐 모든 타입을 엮게 만들 것이다."*

여기서 §7.3.1의 "스위스 치즈"·§7.4의 "타입마다 플래그"와 **완전히 같은 논증 형태**가 세 번째로 반복된다는 것을 짚어두면 챕터를 관통하는 축이 하나로 잡힌다.

```
§7.3.1  인터페이스 하나 = 괜찮다   →  전부 인터페이스 = 스위스 치즈
§7.4    플래그 하나 = 괜찮다       →  타입마다 플래그 = 추론 불가
§8.2    transitive 하나 = 괜찮다   →  전부 transitive = 전체가 얽힘
```

**대가는 유연성이다.**

> *"또 다른 예 — NetworkClient를 서드파티 회사의 새 StorageSDK로 대체한다고 하자. 우리의 naive 해법으로는 CourseService가 변경에 불필요하게 끌려들어간다. 그러나 CourseService가 NetworkClient를 인지하지 않는다면 완전히 영향받지 않는다."*
>
> *"이에 대한 해법은 타입이 자신의 직접적인 의존성만 인지하도록 보장하는 것이다 — 그것이 그것들이 실제로 사용하는 것이기 때문이다."*

> **(p.170) Note** — *"일부 플랫폼은 transitive dependency에 대한 특정 솔루션을 제공한다. 예를 들어, SwiftUI는 깊이 중첩된 subview에 의존성을 전달하기 위해 EnvironmentObject를 제공한다. 그러나 이 접근법은 Apple 플랫폼과 SwiftUI에 제한되며, 잘못 사용되면 크래시할 수 있다."*

> **여기가 이 챕터에서 처음 걸렸던 지점이다.** 첫 출제에서 "ABC problem이 정확히 무엇을 금지하는가"를 묻자 "몰라"로 응답했다. 재교육 후 걸린 정의의 폭이 문제였다 — ABC problem은 "A가 C의 *존재를 아예 모른다*"는 요구가 아니라, **"A의 타입 시그니처에 C가 등장해서는 안 된다"** 는 더 좁은 규칙이다. CourseService의 initializer가 `networkClient: NetworkClient`를 받는 순간 그 코드는 위반이고, `tutorAPI: TutorAPI` 만 받으면 통과다. 이 좁은 정의를 정확히 붙잡아 두는 것이 §8.7.3에서 세 번 반복될 오해(뒤에서 다룸)를 막는 열쇠가 된다.

---

## § 8.3 계층을 뒤집어 안팎을 바꾸기 — 해법의 핵심 아이디어

> *"다행히 우리는 계층을 뒤집어 안팎을 바꿈으로써 ABC problem을 피할 수 있다. 어떻게 작동하는지 살펴보자."*

**§5·§7에서 미뤄뒀던 재료 — transport 계층을 확장한다.**

> *"기억한다면, system-wide testing 챕터에서 우리는 테스팅을 위해 MockTransport를 NetworkClient에 전달하거나, 실제 네트워크 호출을 위한 iOS 특정 방식으로 URLSession을 전달했다. 이 아이디어를 확장하자 — URLSession 대신 라이브 서버에 연결하는 ProductionTransport나 내부 서버에 연결하는 StagingTransport로 분리한다. MockTransport는 그대로 유지한다."*
>
> *"이 세 타입은 NetworkClient가 사용하는 NetworkTransport 인터페이스를 따르며, 이는 NetworkClient가 어떤 transport를 사용하는지 인지하지 못하게 한다."*

```
                       Course
                          │
              ┌───────────┼───────────┐
          TodoAPI      Calendar    TutorAPI
              └───────────┼───────────┘
                    NetworkClient
                          │
             ┌────────────┼────────────┐
      ProductionTransport │      MockTransport
                    StagingTransport
             (점선=NetworkTransport 인터페이스 구현)
```
*(Figure 8.3을 다시 그림 — production·staging·mock 세 transport가 같은 인터페이스를 구현하는 정상 방향 계층)*

**여기서 저자가 그래프를 뒤집는다 — 이 챕터의 결정적인 조작이다.**

> *"먼저, 그래프를 거꾸로 뒤집자. 잠시 후 보겠지만, 이는 그것을 코드로 변환하는 데 도움이 되기 때문에 그렇게 한다."*

```
   ProductionTransport   StagingTransport   MockTransport   ← leaf, 맨 위
             └────────────────┼────────────────┘
                        NetworkClient
              ┌────────────────┼────────────────┐
          TutorAPI          TodoAPI          Calendar
              └────────────────┼────────────────┘
                        CourseService                        ← root, 맨 아래
```
*(Figure 8.4를 다시 그림 — 뒤집힌 dependency 그래프)*

> *"타입은 정확히 같다는 점에 주의하라 — 단지 거꾸로 된 뷰일 뿐이다. 지금까지는 좋다. 다음으로, 이 그래프를 사용해 구현을 안내하자."*

> **여기가 이 챕터에서 가장 미묘하게 걸렸던 지점이다.** "그래프를 뒤집으면 ABC problem이 왜 사라지는가"를 묻는 질문에 "타입 관계 자체가 바뀌어서 문제가 원천적으로 사라진다"고 답해 오답이었다. 저자가 방금 명시한 것과 정반대다 — **타입 관계는 뒤집기 전과 완전히 동일하다.** CourseService가 TutorAPI에 의존한다는 사실도, TutorAPI가 NetworkClient에 의존한다는 사실도 그림을 뒤집는다고 바뀌지 않는다. 뒤집기가 하는 일은 오직 하나, **"어떤 순서로 만들 것인가"를 눈으로 읽기 쉽게 만드는 것**뿐이다. ABC problem을 실제로 푸는 것은 그래프의 방향이 아니라 §8.3.2에서 CourseService의 initializer 시그니처를 바꾸는 것이다 — 뒤집기는 그 작업의 순서를 알려주는 지도일 뿐이다.

### § 8.3.1 코드에서 계층 설정하기 — bottom-up 조립

> *"그래프를 거꾸로 뒤집은 이유는 이것이 우리가 타입을 초기화하는 순서를 더 잘 나타내기 때문이다."*
>
> *"그래프의 맨 위 행에는 leaf dependency가 있다. 즉 ProductionTransport와 StagingTransport. 테스팅을 위해서는 MockTransport가 있다. 그것들은 특별히 어떤 것에도 의존하지 않는다. 적어도 그것들에게 어떤 의존성도 전달할 필요가 없다. 이는 우리가 계층을 설정할 때 그것들로 시작할 수 있게 해준다."*
>
> *"맨 위 행을 먼저 만드는 것으로 시작한다 — 즉 CourseService를 마지막으로 만들 것이다."*

**조립을 담당하는 새 클래스가 여기서 처음 등장한다.**

> *"코드를 보면, 새 AppSetup 클래스 안에 setupCourseService()라는 새 메서드를 도입해 우리 앱의 의존성을 설정할 것이다. 그것이 앱을 부트스트래핑하는 일부이므로, 이 메서드는 Main 근처 같은 우리 앱의 진입점에 살 수 있다."*

네 단계에 걸쳐 코드가 뒤집힌 그래프를 그대로 따라간다.

```swift
final class AppSetup {

    static func setupCourseService() -> CourseService {
        // 1단계 — leaf: 어떤 의존성도 필요 없다
        let transport = StagingTransport()

        // 2단계 — transport를 주입해 NetworkClient
        let networkClient = NetworkClient(transport: transport)

        // 3단계 — NetworkClient로 세 타입 더
        let tutorAPI = TutorAPI(networkClient: networkClient)
        let todoAPI = TodoAPI(networkClient: networkClient)
        let calendar = Calendar(networkClient: networkClient)

        // 4단계 — 마지막으로, 직접 의존성만으로 CourseService
        let courseService = CourseService(tutorAPI: tutorAPI, todoAPI: todoAPI, calendar: calendar)
        return courseService
    }
}
```
*(Listing 8.2~8.5를 하나로 합침 — StagingTransport로 시작하는 이유는 그것 자체가 어떤 의존성도 가지지 않기 때문. NetworkTransport는 인터페이스라 초기화 단계에서 건너뛴다는 점에 주의)*

```
1 leaf         StagingTransport()                          ← 의존성 0개
2              NetworkClient(transport:)                    ← transport 필요
3              TutorAPI / TodoAPI / Calendar(networkClient:) ← networkClient 필요
4 root         CourseService(tutorAPI:todoAPI:calendar:)     ← 셋 다 필요, 그래서 마지막
```
*(Figure 8.5~8.8을 압축 — "행"이 곧 "단계"이고, 각 단계는 이전 단계의 산출물만 소비한다)*

> *"코드가 거꾸로 된 그래프를 따른다는 점을 주목하라. 결과적으로 staging 환경에 연결되는 CourseService 인스턴스로 끝나며, 어떤 타입도 자신의 transitive dependency를 인지하지 않는다!"*

> **(p.177) Note** — *"성숙한 앱에서는 로그인 화면, 메인 네비게이션 등 전체 애플리케이션을 설정할 것이다."*

> **오답 지점 — "컴파일러가 순서를 강제한다"는 오해.** CourseService를 마지막에 만드는 이유를 "컴파일러가 클래스 선언 순서대로 초기화를 강제하기 때문"이라고 답해 오답이었다. 실제 이유는 컴파일러 규칙이 아니라 **값의 존재 여부다** — CourseService의 initializer가 `tutorAPI`·`todoAPI`·`calendar`라는 세 값을 요구하는데, 그 값 자체가 3단계 전에는 아직 존재하지 않는다. "먼저 선언됐으니 먼저 만든다"가 아니라 **"참조하려면 그 값이 이미 있어야 하니 나중에 만든다"** — 순서를 만드는 것은 문법이 아니라 데이터 의존 사슬이다.

### § 8.3.2 CourseService 업데이트 — naive 해법을 되돌리기

> *"우리는 이미 CourseService가 NetworkClient를 직접 사용하지 않으므로 NetworkClient 인스턴스를 받아서는 안 된다는 것을 다뤘다. 대신, CourseService는 주입을 통해 — 또는 우리가 부르는 '단지 값을 전달하기' — 자신의 직접적인 의존성을 받아야 한다."*

```swift
final class CourseService {

    // CourseService는 이제 의존성을 받는다.
    // 그것을 인스턴스화하지 않는다.
    private let tutorAPI: TutorAPI
    private let todoAPI: TodoAPI
    private let calendar: Calendar

    // 받은 의존성을 로컬 속성에 할당한다
    init(tutorAPI: TutorAPI, todoAPI: TodoAPI, calendar: Calendar) {
        // 봐라, 더 이상 NetworkClient가 없다!
        self.tutorAPI = tutorAPI
        self.todoAPI = todoAPI
        self.calendar = calendar
    }
}
```
*(Listing 8.6 — CourseService에 직접 의존성만 전달)*

> *"CourseService가 모든 의존성을 받기 때문에, NetworkClient를 인지하지 않는다. ABC problem을 해결했다!"*

**§8.1에서 던진 질문에 이제 답이 나왔다.** "의존성을 설정하는 것이 정말 CourseService의 책임인가?" — 아니다, 그것은 AppSetup의 책임이다. CourseService의 initializer 시그니처가 §8.1의 `init(networkClient:)`에서 §8.3.2의 `init(tutorAPI:todoAPI:calendar:)`로 좁혀진 것이 이 절 전체의 결과물이며, 이 **좁아진 시그니처가 이후 §8.7.3에서 오답의 진짜 판별 기준**이 된다.

---

## § 8.4 테스트 환경 — 같은 로직, transport만 교체

> *"테스팅할 때 대신 MockTransport를 NetworkClient에 전달함으로써 같은 로직을 적용할 수 있다. 테스팅 설정에서, 이전과 비슷하게 환경을 설정한다 — 단지 이제 ProductionTransport 대신 MockTransport를 사용해 네트워크가 mock되는 것만 다르다. 다른 모든 것은 같다."*

```swift
final class TestCourse: XCTestCase {

    func testCourse() {
        let course = makeCourseService()
        // 여기서 course를 테스트할 수 있다.
    }

    // 테스트용 CourseService 인스턴스를 만드는 factory
    func makeCourseService() -> CourseService {
        // 테스팅 동안 어떤 서버에도 연결하지 않는 MockTransport
        let transport = MockTransport()

        // 아래 모든 것은 이전과 정확히 같다.
        let networkClient = NetworkClient(transport: transport)
        let tutorAPI = TutorAPI(networkClient: networkClient)
        let todoAPI = TodoAPI(networkClient: networkClient)
        let calendar = Calendar(networkClient: networkClient)

        return CourseService(tutorAPI: tutorAPI, todoAPI: todoAPI, calendar: calendar)
    }
}
```
*(Listing 8.7 — 테스팅 환경 의존성 설정, §8.3.1과 한 줄만 다르다)*

**저자는 이 코드의 약점을 먼저 인정한다.**

> *"이 접근법은 많은 boilerplate처럼 보일 수 있다. 그래서 사람들이 싱글턴 같은 지름길에 손을 뻗을 수 있다. 그러나 비교적 직관적이고 관리 가능하다는 것을 보장한다. 우리 테스트에서 훨씬 더 많은 실제 코드를 테스트하고 있다는 것은 말할 것도 없다."*

> **(p.178) Note** — *"boilerplate를 다루는 기법이 있다 — System-wide testing 챕터에서 찾을 수 있다."*

**진짜 이점은 boilerplate 감수의 대가로 얻는 일관성이다.**

> *"이 접근법의 이점은 다른 빌드 전체에 컴파일러 플래그를 흩뿌리는 것과 달리, CourseService와 모든 sub-도메인이 환경 전반에서 동일하게 동작한다는 점이다."*

Production setup(Listing 8.5)과 테스트 setup(Listing 8.7)이 **한 줄(transport 종류)만 다르고 나머지는 토씨 하나 같다는 것 자체가 §8.3의 성과 증명**이다. §8.3이 "타입이 transitive dependency를 모르게" 만든 결과, 환경을 바꾸는 데 필요한 변경 폭이 정확히 leaf 한 줄로 줄어들었다.

> *"그러나 우리는 여전히 staging 대신 production 서버에 연결되는 다른 빌드를 지원해야 한다. 이것이 컴파일러 플래그가 필요한 곳이지만, 우리는 그것들을 전략적으로 중앙화할 수 있다."*

---

## § 8.5 애플리케이션 외곽 edge에서 컴파일러 플래그 — §7.4의 약속을 갚는다

> *"이전 챕터에서 컴파일러 플래그를 흩뿌리는 것이 코드베이스에 대해 추론하기 더 어렵게 만든다는 것을 다뤘다. 이를 해결하기 위해 가능하면 그것들을 중앙화하는 것이 더 낫다."*
>
> *"컴파일러 플래그를 위한 사용 가능한 위치는 애플리케이션의 시작 지점이며, 거기서 모든 것을 설정하고 부트스트랩할 수 있다. 이는 동료와 우리 자신이 앱 구성을 찾기 더 쉽게 만든다."*
>
> *"우리는 이미 AppSetup 클래스에 setupCourseService() 메서드를 만들었으며, 거기서 우리 기능을 부트스트랩한다. 지금 우리가 가진 것이 그것뿐이므로, 컴파일러 플래그를 추가하기 위해 이 위치를 사용하자."*

```swift
final class AppSetup {

    static func setupCourseService() -> CourseService {
        // 환경에 따라 staging이나 production에 연결되는 빌드를 만든다.
        #if DEBUG
        let transport = StagingTransport()
        #else
        let transport = ProductionTransport()
        #endif

        // 아래 코드는 transport에 관계없이 정확히 같다.
        let networkClient = NetworkClient(transport: transport)
        let tutorAPI = TutorAPI(networkClient: networkClient)
        let todoAPI = TodoAPI(networkClient: networkClient)
        let calendar = Calendar(networkClient: networkClient)

        return CourseService(tutorAPI: tutorAPI, todoAPI: todoAPI, calendar: calendar)
    }
}
```
*(Listing 8.8 — 애플리케이션의 외곽 edge에서 컴파일러 플래그)*

> *"네트워킹 계층을 제외하고, 우리는 이제 모든 타입이 어떤 환경에서 실행되든 정확히 같게 작동하는 클래스 계층으로 끝났다. 그것은 우리가 다양한 타입에 컴파일러 플래그를 흩뿌리지 않는다는 의미이다. 컴파일러 플래그를 우리 앱의 root에 두는 것은 코드베이스에 대해 추론하기 훨씬 쉽게 만든다."*

> **(p.180) Note — 겸손한 단서** — *"이 방식으로 모든 컴파일러 플래그를 그룹화할 수 없다는 것을 명심하라. 특정 OS 버전이나 플랫폼에 대해서는 다른 위치에 컴파일러 플래그를 두는 것을 피할 수 없다."*

이 단서가 중요한 이유는, §8.5가 "플래그를 전부 없애자"가 아니라 **"흩뿌릴 이유가 없는 플래그만 한 곳에 모으자"** 는 더 좁고 실현 가능한 주장이라는 것을 확인해주기 때문이다. `#if DEBUG`처럼 **환경**을 결정하는 플래그는 root로 모이지만, OS 버전 분기처럼 **그 자리에서만 의미 있는** 플래그는 원래 자리에 남는다.

> *"간단한 setup으로 기능을 위한 DI 계층을 만들었으므로 자신의 등을 두드릴 수 있다. 더 복잡한 setup과 싸우기 전에, 이 솔루션이 ABC problem을 어떻게 피하는지에 대한 secret sauce를 이해하자."*

---

## § 8.6 secret sauce — 왜 이 방법이 작동하는가

**여기까지는 "어떻게"였다. §8.6은 "왜"로 한 발 물러선다.**

> *"ABC problem을 피하는 secret sauce는 타입의 생성자 — 우리의 경우 AppSetup — 가 모든 의존성에 접근할 수 있다는 점이다."*
>
> *"시작 시, 우리 애플리케이션은 아직 계층을 가지지 않는다. AppSetup은 모든 의존성과 그것들의 transitive dependency에 접근할 수 있다. AppSetup은 평평한 의존성 계층으로 작업하고 있었다."*

```
설정 전 — AppSetup은 모두에게 평평하게 접근 가능
   AppSetup ──┬──▶ StagingTransport / ProductionTransport / MockTransport
              ├──▶ NetworkClient
              ├──▶ TutorAPI, TodoAPI, Calendar
              └──▶ CourseService
   (계층이 없다 — 전부 AppSetup의 시야 안에 평평하게 놓여 있다)
```
*(Figure 8.9를 다시 그림)*

> *"의존성을 설정할 때, 우리 계층은 AppSetup이 모든 타입에 도달할 수 있는 이 모습이다. 끝난 후에는 모든 클래스가 자체 의존성을 가지며 계층을 형성한다."*

```
설정 후 — 계층이 형성된다
   AppSetup ──▶ CourseService ──▶ TutorAPI/TodoAPI/Calendar ──▶ NetworkClient ──▶ Transport
   (각 타입은 자신의 직접 의존성만 알고, AppSetup만 전체를 조망했다)
```
*(Figure 8.10을 다시 그림)*

**두 번째 재료가 방금 §8.3.1에서 이미 실행한 것이다.**

> *"이 secret sauce의 두 번째 재료는 모든 타입을 bottom-up으로 연결했다는 점이다. 우리는 가장 낮은 수준의 요소 — 즉 NetworkClient에 전달할 transport — 로 시작했다. 그런 다음 TodoAPI, Calendar, TutorAPI를, 그리고 마지막으로 CourseService를 초기화했다."*

정리하면 secret sauce는 두 재료의 조합이다.

| 재료 | 정체 |
|---|---|
| ① 평평한 접근 | AppSetup은 계층이 형성되기 **이전** 시점에 존재하므로, 모든 타입(가장 깊은 transitive dependency까지)에 동시에 접근할 수 있다 |
| ② bottom-up 순서 | 가장 깊은 것부터 만들어 위로 올라가면, 각 타입은 자신이 실제로 쓰는 **직접 의존성만** 건네받고 그 이상은 필요가 없다 |

**①이 있어야 ②가 가능하고, ②가 있어야 ①의 이점이 실현된다** — 둘은 분리된 두 트릭이 아니라 하나의 메커니즘의 두 얼굴이다.

> **여기가 이 챕터에서 두 번째로 "몰라"가 나온 지점이다.** secret sauce의 두 재료를 묻는 질문에 처음엔 답하지 못했다. 이 절이 다른 절과 다른 점은 **새 기법을 소개하는 게 아니라, 이미 §8.3.1에서 한 일을 한 발 물러나 이름 붙이는 절**이라는 데 있다 — 그래서 "무엇을 새로 배웠나"로 접근하면 오히려 답이 안 나온다. "방금 한 것 중 무엇이 핵심이었나"로 되짚어야 두 재료(평평한 접근 + bottom-up 순서)가 보인다.

### § 8.6.1 ABC 규칙 깨기 — 위반은 없앨 수 없다, 가둘 수 있을 뿐

> *"주목할 만한 한 가지 중요한 점은 의존성을 설정하는 클래스가 ABC 규칙을 깬다는 점이다. 예를 들어, AppSetup은 NetworkClient를 직접 사용하지 않지만, 그것을 만들고 설정한다 — 이는 CourseService가 원래 NetworkClient를 사용해 자신의 속성을 설정한 방식과 비슷하다."*

이 문장이 §8.6 전체의 반전이다. **AppSetup은 §8.1의 CourseService가 저질렀던 것과 똑같은 종류의 위반을 저지르고 있다** — NetworkClient를 직접 쓰지 않으면서 그것을 인지하고 만든다. 다른 점은 딱 하나, **누가 그 위반을 떠안느냐**다.

> *"앱 전체에 걸쳐 다양한 위치가 ABC 규칙을 깰 것이다. 그러나 목표는 이것이 너무 많이 일어나지 않도록 최소화하는 것이다. 의존성 계층을 설정할 특정 위치를 결정하려고 노력하라. 다음 챕터에서 이를 더 탐구할 것이다."*

> **오답 지점 — "컴파일 타임 자동 검증"이라는 상상.** ABC 규칙 위반이 안전한 이유를 "컴파일 타임에 위반 여부가 자동으로 검증되므로 위험이 없다"고 답해 오답이었다. 그런 자동 검증 메커니즘은 이 챕터 어디에도 없다 — Swift 컴파일러는 AppSetup이 NetworkClient를 알든 모르든 신경 쓰지 않는다. 실제 안전장치는 **자동 검증이 아니라 위반의 지리적 집중**이다. ABC 규칙을 어기는 코드가 코드베이스 전체에 흩어져 있다면 위험하지만, **AppSetup 한 파일 — 정확히는 setup 메서드들 — 로 국한**되면 나머지 코드는 전부 규칙을 지킨 채로 남는다. §7.4의 "컴파일러 플래그를 중앙화하자"와 정확히 같은 처방이 여기서 "규칙 위반을 중앙화하자"로 반복되는 것이며, 이것이 §8.6.1이 §7과 이어지는 지점이다.

> *"이 솔루션이 장난감 크기의 예제를 넘어 확장되는지 궁금할 수 있다. 더 복잡한 시나리오를 풀어보자."*

---

## § 8.7 앱 키우기 — 같은 원칙을 더 큰 그래프에

**§8.7~§8.9는 하나의 질문에 답한다: "이 secret sauce가 정말 확장되는가, 아니면 CourseService 하나짜리 장난감에서만 통하는 우연인가?"**

> *"새 Marketplace 기능을 도입할 것이다. 이는 사용자가 튜터의 코스를 찾기 위해 보는 화면이다. marketplace 안에서 사용자는 코스를 검색하고 탐색하고, 가격과 평점을 비교하고, 구독하고, 튜터의 코스를 결제할 수 있다."*
>
> *"이 책에서는 marketplace UI에 집중하지 않을 것이지만, 그 의존성에 집중할 것이다."*

### § 8.7.1 더 많은 클래스로 그래프 확장

> *"이 기능을 지원하기 위해 새 Marketplace 클래스를 도입하자. 이 클래스는 코스를 가져오고 필터링하는 것, 코스 결제 같은 데이터 측면을 처리할 수 있다."*
>
> *"우리 클래스 계층에서 CourseService는 이제 Marketplace가 사용하는 의존성이 되어, Marketplace를 최상위 요소로 만든다."*

**동시에 영속성도 되돌아온다 — 이번엔 다른 위치에.**

> *"예를 더 현실적으로 만들기 위해, 일부 영속성도 다시 추가하자. 이전 챕터에서 CourseService에서 Store를 제거한 것을 기억하는가? 다시 도입하지만, 이번에는 그것을 NetworkClient에 부착해 코스만이 아니라 어떤 네트워크 데이터든 캐시할 수 있게 하자. 이 더 광범위한 캐싱 접근법은 우리 전체 앱을 위한 오프라인 모드까지 가능하게 할 수 있다."*
>
> *"Store 자체는 메모리 저장(테스팅에 유용)과 파일 저장 사이를 교체할 수 있게 해주는 protocol인 StorageType에 의존한다. MemoryStorage와 FileStorage 구현 모두를 제공할 것이다."*

```
              Marketplace                          ← 새 최상위
                   │
             CourseService
       ┌────────────┼────────────┐
   TutorAPI      TodoAPI      Calendar
       └────────────┼────────────┘
                NetworkClient
                     │
                   Store                            ← 새로 부착
                     │
                StorageType (interface)
              ┌──────┴──────┐
       MemoryStorage    FileStorage
```
*(Figure 8.12를 다시 그림 — Store가 CourseService가 아니라 NetworkClient에 붙어, 코스뿐 아니라 어떤 네트워크 데이터든 캐시하게 됨을 보여준다)*

### § 8.7.2 그래프 뒤집기 — 같은 트릭, 더 큰 그래프

> *"이 새 타입을 추가함으로써 ABC problem을 다시 다뤄야 하지만, 이제 더 큰 규모에서이다. ABC problem을 피하기 위해, Marketplace가 TutorAPI나 NetworkClient 같은 자신의 transitive dependency에 대해 알기를 원하지 않는다. CourseService도 다양한 storage 타입에 대해 알기를 원하지 않는다."*
>
> *"이를 설정하기 위해 이전과 같은 방법을 적용할 것이다. 설정 단계 동안, 가장 깊은 요소부터 시작해 모든 타입을 인스턴스화하고 모두 함께 연결할 것이다. 그래프를 다시 거꾸로 뒤집고, 코드에서 사용하는 접근법과 더 가깝게 닮도록 아래로 작업해보자."*

```
   MemoryStorage/FileStorage   StagingTransport/ProductionTransport    ← leaf, 맨 위
              │                              │
            Store                            │
              └──────────────┬───────────────┘
                        NetworkClient
              ┌────────────────┼────────────────┐
          TutorAPI          TodoAPI          Calendar
              └────────────────┼────────────────┘
                        CourseService
                              │
                        Marketplace                                    ← root, 맨 아래
```
*(Figure 8.13을 다시 그림)*

> *"다시 한 번 그래프는 이전과 정확히 같으며, 단지 거꾸로 표현되었을 뿐이다."*

### § 8.7.3 코드에서의 더 큰 ABC problem — 그리고 이 챕터의 진짜 함정

> *"뒤집힌 그래프로, 다시 한 번 코드로 표현할 수 있다. StorageType의 의존성이 이제 맨 위에 있으므로, 그것을 먼저 초기화할 것이다. 먼저, storage 타입을 골라야 한다. Holistic-Driven Design 챕터에서 이미 사용했던 MemoryStorage를 선택하자."*
>
> *"그런 다음 Store, NetworkClient 등을 초기화할 수 있다. 결과적으로 다시 CourseService 인스턴스로 끝나며, 이를 사용해 Marketplace 인스턴스를 초기화하고 반환할 것이다."*

**의존성이 늘어나자 저자는 조립 코드 자체를 조직한다.**

> *"다룰 의존성이 더 많아졌으므로, 전용 메서드를 도입하자. setupCourseService()를 setupMarketplace()로 이름을 바꾼다. NetworkClient 설정이 Store와 함께 더 많은 setup을 요구하므로, 그것을 setupNetworkClient()로 추출할 것이다. 최종 결과는 함께 전체 계층을 만드는 두 setup 메서드이다."*

```swift
final class AppSetup {

    // setupCourseService()는 이제 setupMarketplace()이다
    static func setupMarketplace() -> Marketplace {
        // 새로 추출한 setupNetworkClient를 호출한다
        let networkClient = setupNetworkClient()

        let tutorAPI = TutorAPI(networkClient: networkClient)
        let todoAPI = TodoAPI(networkClient: networkClient)
        let calendar = Calendar(networkClient: networkClient)

        let courseService = CourseService(tutorAPI: tutorAPI, todoAPI: todoAPI, calendar: calendar)

        // NEW: courseService 인스턴스를 전달해 Marketplace 인스턴스를 만들 수 있다.
        let marketplace = Marketplace(courseService: courseService)
        return marketplace
    }

    // NetworkClient 설정을 자체 메서드로 옮겼다
    static func setupNetworkClient() -> NetworkClient {
        // 가장 깊은 요소인 MemoryStorage를 초기화한다.
        let storage = MemoryStorage()
        // 그런 다음 Store를 초기화하고 이전처럼 계속한다.
        let store = Store<Data>(storageType: storage)

        // 이전처럼 transport를 초기화한다
        #if DEBUG
        let transport = StagingTransport()
        #else
        let transport = ProductionTransport()
        #endif

        // NetworkClient는 이제 store에도 의존한다
        let networkClient = NetworkClient(transport: transport, store: store)
        return networkClient
    }
}
```
*(Listing 8.9 — 코드에서의 더 큰 ABC problem. `setupMarketplace()`가 `setupNetworkClient()`를 호출하는 구조에 주의)*

> *"몇 줄의 추가 코드로 전체 계층을 설정할 수 있었으며, 어떤 의존성도 속해서는 안 되는 곳으로 새지 않는다. 다시 한 번 ABC problem을 해결했다!"*
>
> *"인터페이스의 수를 낮게 유지했다는 점에 주목하라. 우리는 NetworkTransport와 StorageType만 가지고 있다. 더 적은 인터페이스는 system-wide testing 챕터에서 다뤘듯이 mock된 코드 대신 우리가 출시하는 코드를 테스트하는 것을 장려한다."*

> **이 챕터에서 유일하게 3연속으로, 서로 다른 각도에서 틀렸던 지점이다.** 질문은 한결같았다 — "왜 `setupNetworkClient()`를 별도 메서드로 뽑았는가?" — 인데 답이 매번 다른 방향으로 빗나갔다.
>
> 1회차: *"NetworkClient가 Marketplace 없이는 생성될 수 없어서 분리했다"* — 정반대다. Listing 8.9를 보면 `setupNetworkClient()`는 Marketplace를 전혀 참조하지 않는다. MemoryStorage → Store → transport → NetworkClient까지, 이 메서드 하나로 완결된다.
> 2회차 (`/lesson` 재교육 직후): *"메서드를 합치면 ABC problem이 재발해 CourseService가 NetworkClient를 알게 된다"* — 여전히 아니다. 메서드를 하나로 합쳐도 `let networkClient = ...` 라는 로컬 변수가 `setupMarketplace()` 안에 그대로 남을 뿐, CourseService의 initializer는 §8.3.2에서 이미 `tutorAPI:todoAPI:calendar:`로 고정됐다. **AppSetup 내부에서 코드를 몇 개 함수로 나누는지는 CourseService의 시그니처와 아무 관계가 없다.**
> 3회차: *"메서드를 합치면 CourseService의 initializer에 NetworkClient 매개변수가 추가된다"* — 가장 근본적인 오해다. **AppSetup *내부의* 메서드 조직**과 **CourseService *자신의* public 계약(initializer 시그니처)** 은 완전히 다른 두 층위인데, 이 둘을 같은 층위로 취급하고 있다. AppSetup이 setup 로직을 한 함수에 몰아 쓰든 두 함수로 나누든, CourseService 코드는 단 한 글자도 바뀌지 않는다.
>
> **진짜 이유는 셋 다와 무관하다.** Listing 8.9 직전 텍스트가 답을 이미 말한다 — *"다룰 의존성이 더 많아졌으므로, 전용 메서드를 도입하자."* StorageType·Store라는 새 타입이 추가되며 `setupMarketplace()` 하나에 몰아 쓰기엔 길어지고 관리하기 번거로워졌다는, **순수하게 가독성·조직 문제**다. 생성 가능 여부도, ABC problem 재발도, initializer 변경도 아니다. 세 번의 오답이 그리는 궤적(생성 순서 → 타입 관계 → public 계약)은 매번 "무언가 구조적인 이유가 있을 것"이라고 더 깊이 파고든 결과였지만, 정답은 그 반대편의 가장 평범한 이유에 있었다. 결국 사용자가 "지엽적"이라 판단해 수동 🟢 처리됐다 — AppSetup 내부 조직과 CourseService 공개 계약의 층위 구분 자체는 안드로이드 실무 관점에서 더 파고들 가치가 낮다고 본 것이다.

**두 개의 대가도 미리 인정한다.**

> *"이 접근법의 한 가지 단점은 모든 것을 미리, 또는 eagerly 인스턴스화한다는 점이다. 이는 보통 문제가 아니지만, 전체 의존성 트리를 미리 만드는 것이 항상 우리가 원하는 것은 아니며, 종종 가능하지도 않다! 앱이 막 시작되었을 때 모든 의존성이 사용 가능하지 않을 수 있기 때문이다. factory를 사용해 이 문제를 해결하기 전에, 먼저 lazy dependency에 대한 우리의 이해를 깊게 하자."*

이 문장이 §8.8~§8.9로 가는 다리다. §8.7까지는 **"모든 것을 미리 만들 수 있다"** 는 전제 위에 서 있었다. 다음 두 절은 그 전제가 깨지는 경우를 다룬다.

---

## § 8.8 의존성이 사용 가능하지 않을 때 — eager의 한계

> *"Marketplace를 그것의 모든 직접 의존성, 그리고 그 의존성의 의존성 — transitive dependency라고도 알려진 — 을 만들어 초기화했다. setupMarketplace()에서 모든 의존성을 만들 수 있었던 이유는 모든 것을 초기화할 수 있기 때문이다 — 모든 (transitive) 의존성이 AppSetup에 사용 가능하다."*
>
> *"그러나 종종 모든 (transitive) 의존성이 사용 가능하지 않아 타입의 인스턴스를 만들 수 없는 경우가 있다."*

### § 8.8.1 결제 플로우 — 런타임에만 결정되는 값

> *"결제 플로우를 만들고 있다고 상상해보자 — 사용자가 marketplace로 이동해 튜터를 선택하고, 튜터의 코스를 결제한다. Marketplace가 이 기능을 지원하기 위해 Payments 인스턴스를 만든다고 가정하자."*
>
> *"Payments가 PaymentProvider 모델을 포함한 자체 의존성과 함께 온다는 점에 주목하라. PaymentProvider는 사용자가 결제하고 싶은 방식을 나타낸다 — 신용카드나 Paypal 같은."*

```
   Marketplace
        │
   CourseService     Payments               ← 새 의존성
                          │
                   PaymentsAPI  PaymentProvider  ← 런타임에만 결정
                          │
                    NetworkClient (깊이 묻힌 transitive dependency)
```
*(Figure 8.14를 압축)*

> **(p.188) Note** — *"이것이 실제 iOS 앱이라면, Apple은 여기서 In-App Purchases를 유일한 결제 제공자로 강제할 것이다. 그러나 그것이 우리 위에 떠 있지 않는 것처럼 계속하자."*

**막히는 지점이 두 겹이다.**

> *"이상적으로 setupMarketplace() 동안 Payments 인스턴스를 만들고, 그것을 사용해 이전처럼 Marketplace 인스턴스를 만들 수 있다. 그러나 Payments가 PaymentsAPI와 PaymentProvider에 의존한다는 점에 주목하라. 그러나 setupMarketplace() 동안에는 사용자가 아직 PaymentProvider를 선택하지 않았으므로, 그것은 사용 가능하지 않다."*
>
> *"PaymentProvider가 있더라도, Marketplace는 여전히 자체 Payments 인스턴스를 만들 수 없다. 이는 Marketplace가 PaymentsAPI에 전달할 — 깊이 중첩된 transitive dependency인 — NetworkClient에 접근할 수 없기 때문이다."*

| 막힌 지점 | 누가 가진 정보인가 |
|---|---|
| PaymentProvider | **사용자**가 화면에서 선택 — AppSetup은 앱 시작 시점에 절대 알 수 없다 |
| NetworkClient | **AppSetup**이 가진 값 — 하지만 Marketplace는 그것을 몰라야 한다(ABC problem) |

> *"우리 의존성이 분리된 것 같다. AppSetup과 Marketplace 모두 Payments에 대해 각각 하나의 의존성만 공급할 수 있다."*

**여기서 빠른 해결책이 제시되고, 곧바로 기각된다.**

> *"빠른 해결책은 NetworkClient를 Marketplace에 주는 것이며, 그러면 그것이 나중에 Payments를 초기화할 수 있다. 이는 우리 문제를 해결하지만, Marketplace가 transitive dependency를 인지하게 되므로 ABC problem을 도입한다."*

§7.5.5에서 봤던 "빠른 해법 두 개를 검토하고 둘 다 기각하는" 패턴이 여기서 세 번째로 나타난다 — **국소적으로는 작동하는 해법이 챕터의 원칙(여기서는 §8.2의 ABC problem 금지)을 정면으로 어긴다.**

### § 8.8.2 Optional 의존성 — 만들 필요가 없을 수도 있다는 문제

> *"사용자가 단일 결제 제공자에 제한된다고 상상해보자. 그것은 Payments가 자체 PaymentProvider를 만들 수 있어 더 이상 전달될 필요가 없다는 의미이다. 그 시나리오에서, AppSetup은 PaymentProvider를 공급할 필요가 없으므로 Marketplace에 줄 Payments 인스턴스를 만들 수 있다."*

그런데도 여전히 문제가 남는다 — 이번엔 **필요성** 자체의 문제다.

> *"그러나 사용자가 코스에 구독하지 않으면 이 플로우를 표시할 필요가 없다. setupMarketplace() 동안 Payments를 미리 만드는 것은 디바이스 자원을 낭비하고 더 느린 앱 시작을 유발한다."*
>
> *"이 의존성을 optional로 간주할 수 있다. 앱의 수명 동안 사용자의 행동에 따라 필요할 수도 있고 그렇지 않을 수도 있다."*
>
> *"이를 해결하기 위해, Marketplace가 필요할 때만 Payments를 만들도록 할 수 있다. 그러나 그러면 Marketplace에 NetworkClient 인스턴스를 전달해야 하며, 이는 ABC problem을 다시 도입한다."*

두 시나리오(런타임 값이 없어서 못 만든다 / 만들 필요가 아직 없다)가 **같은 결론으로 수렴**한다는 것이 이 절의 요점이다.

> *"두 시나리오 모두에서 lazy initialization이 이 문제를 해결한다. 실제로 어떻게 작동하는지 다루자."*

---

## § 8.9 지연(lazy) 의존성 — factory로 늦추기

**§8.9는 §8.8이 남긴 두 문제 중 더 복잡한 쪽(런타임 값이 필요한 경우)을 정면으로 푼다.**

> *"방금 논의한 두 시나리오 중, 더 복잡하므로 첫 번째 시나리오를 처리할 것이다. 결제 플로우가 사용자가 런타임에 선택하는 PaymentProvider를 요구하는 시나리오이다."*
>
> *"일반 의존성과 달리, lazy 의존성은 의존성을 반환하는 함수이다. 따라서 Payments를 전달하는 대신, Payments 인스턴스를 반환하는 함수를 전달할 수 있다."*

**하지만 함수 그 자체는 다루기 번거롭다 — 그래서 타입으로 감싼다.**

> *"그러나 (익명) 함수를 다루는 것은 작업하기 다루기 힘들 수 있다. 코드를 더 이해하기 쉽게 만들기 위해, 이 함수를 PaymentFactory에 넣어 형식화할 수 있다. 이 factory는 Marketplace의 의존성이다 — 그것이 나중 시점에 필요할 때 Payments 인스턴스를 만들 타입이기 때문이다."*

> **여기가 이 챕터에서 세 번째로 "몰라"가 나온 지점이다.** 익명 함수 대신 factory 타입을 도입하는 이유를 묻자 답하지 못했다. 요점은 성능이 아니다 — *함수를 값처럼 들고 다니는 것 자체가 코드를 읽기 어렵게 만드니, 이름 있는 타입(`PaymentsFactory`)으로 형식화해 "이것은 Payments를 나중에 만드는 역할"이라는 의도를 코드에 새겨 넣는다.* §7.3.2가 인터페이스에 대해 했던 말("타입에 이름이 붙으면 역할이 즉시 명확해진다")이 함수 대 타입 사이에서도 그대로 반복된다.

### § 8.9.1 코드에서 factory 표현하기

> *"PaymentsFactory 안을 보면, makePayments(provider:) 메서드가 있는 것을 볼 수 있다. 그것은 PaymentProvider를 받고 Payments 인스턴스를 반환한다. Marketplace가 makePayments(provider:)를 호출하고 PaymentProvider를 전달할 자가 될 것이다. 문제는 Payments 인스턴스를 만드는 것 또한 NetworkClient를 요구한다는 점이다."*

```swift
// 이것은 아직 작동하지 않을 것이다.
final class PaymentsFactory {
    func makePayments(provider: PaymentProvider) -> Payments {
        // 새 Payments 인스턴스를 만들 때, 여전히 NetworkClient가 필요하다
        return Payments(networkClient: ???, provider: provider)
    }
}
```
*(Listing 8.10 — PaymentsFactory 도입, NetworkClient 자리가 비어 있다)*

**빈 자리를 채우는 판단 기준이 이 절의 핵심이다.**

> *"여전히 어디선가 NetworkClient가 필요하다. 그러나 NetworkClient를 makePayments(provider:)의 매개변수로 추가하는 것은 말이 되지 않는다. Marketplace가 그것을 공급할 수 없기 때문이다."*
>
> *"반대로 AppSetup은 PaymentProvider를 공급할 수 없지만, AppSetup의 setupMarketplace() 동안 NetworkClient 인스턴스를 공급할 수 있다."*

```
   AppSetup ──(setup 시점, NetworkClient 공급 가능)──▶ PaymentsFactory.init(networkClient:)
                                                              │
   Marketplace ──(호출 시점, PaymentProvider만 공급 가능)──▶ makePayments(provider:)
```
*(Figure 8.17을 다시 그림 — 두 화살표가 서로 다른 시점·다른 공급자를 나타낸다)*

> *"결과적으로 AppSetup은 생성 시 initializer를 통해 NetworkClient를 factory에 전달할 수 있다."*

```swift
final class PaymentsFactory {
    private let networkClient: NetworkClient

    // 이 factory를 만들 때 NetworkClient 인스턴스를 전달할 수 있다
    init(networkClient: NetworkClient) {
        self.networkClient = networkClient
    }

    func makePayments(provider: PaymentProvider) -> Payments {
        // 미리 로드된 NetworkClient와 새로 전달된 PaymentProvider를 모두 사용한다.
        return Payments(networkClient: networkClient, provider: provider)
    }
}
```
*(Listing 8.11 — PaymentsFactory는 NetworkClient와 미리 로드되어 온다. Marketplace는 PaymentProvider만 공급한다)*

> *"Payments가 factory 자체로부터 NetworkClient를 받는 점에 주목하라. PaymentProvider는 makePayments(provider:)의 매개변수를 통해 받는다. (…) 이 솔루션 덕분에 Marketplace는 NetworkClient에 대해 알 필요가 없으며, 따라서 다시 한 번 ABC problem을 피한다."*

**일반화된 규칙이 여기서 명시된다 — 이 챕터에서 가장 실전적인 한 줄이다.**

> *"이를 자신의 코드에 적용하려면, 의존성을 설정할 때 무엇을 준비할 수 있는지 파악하고, 무엇을 나중에 전달할 수 있는지 파악하라. 할 수 있는 것을 initializer에 추가하려고 노력하라. 나중에 전달할 수 있는 어떤 것이든 factory 메서드의 인자가 된다."*

| 값 | 언제 알려지는가 | 어디에 놓이는가 |
|---|---|---|
| NetworkClient | **setup 시점** (AppSetup이 이미 가지고 있음) | factory의 `init` |
| PaymentProvider | **호출 시점** (사용자가 방금 선택함) | factory 메서드의 인자 |

### § 8.9.2 factory 사용하기 — Marketplace와 AppSetup 양쪽에서

> *"Marketplace 안을 보면, PaymentsFactory를 사용해 Payments 타입을 어떻게 만드는지 볼 수 있다. 초기화 동안, CourseService와 PaymentsFactory를 전달하며, 이는 그것의 직접 의존성이다."*

```swift
final class Marketplace {

    // Marketplace는 자신의 직접 의존성에만 의존한다.
    // NetworkClient 같은 transitive dependency는 없다.
    private let courseService: CourseService
    private let paymentsFactory: PaymentsFactory

    init(courseService: CourseService, paymentsFactory: PaymentsFactory) {
        self.courseService = courseService
        self.paymentsFactory = paymentsFactory
    }

    // Marketplace는 결제 제공자의 default 목록을 제공한다.
    static let paymentProviders: [PaymentProvider] = [
      PaymentProvider.creditCard,
      PaymentProvider.payPal,
      PaymentProvider.inAppPurchases
    ]

    // 사용자가 선택한 selectedProvider를 전달한다.
    func makePayments(selectedProvider: PaymentProvider) -> Payments {
        return paymentsFactory.makePayments(provider: selectedProvider)
    }

    func fetchCourses() async throws -> [Course] {
        try await courseService.fetchCourses(limit: 50, offset: 0)
    }
}
```
*(Listing 8.12 — Marketplace 안에서 factory 사용. 결제 제공자 목록은 하드코딩되어 있고, 사용자가 그중 하나를 고르면 그때 비로소 Payments가 만들어진다)*

**setup 코드도 factory를 반영해 갱신된다.**

```swift
final class AppSetup {

    static func setupMarketplace() -> Marketplace {
        let networkClient = setupNetworkClient()

        let tutorAPI = TutorAPI(networkClient: networkClient)
        let todoAPI = TodoAPI(networkClient: networkClient)
        let calendar = Calendar(networkClient: networkClient)

        let courseService = CourseService(tutorAPI: tutorAPI, todoAPI: todoAPI, calendar: calendar)

        // 여기서 networkClient에 의존하는 payments factory를 도입한다.
        let paymentsFactory = PaymentsFactory(networkClient: networkClient)

        // factory를 marketplace에 전달한다
        let marketplace = Marketplace(courseService: courseService, paymentsFactory: paymentsFactory)
        return marketplace
    }
}
```
*(Listing 8.13 — setupMarketplace()에서 PaymentsFactory 초기화. Payments 자신은 여기서 만들어지지 않는다는 점에 주의 — 오직 factory만 만들어진다)*

> *"기능을 위한 꽤 광범위한 의존성 트리를 설정했다. ABC problem에 대한 우리 솔루션은 이제 lazy initialization 지원, setup 동안의 의존성 인스턴스화, 그리고 더 깊은 수준의 의존성 처리를 포함한다."*

---

## § 8.10 결론 — 장황함이 마법보다 낫다

> *"어떤 멋진 프레임워크에도 손을 뻗지 않고 dependency injection을 해결했다. 마법의 어노테이션도, reflection도, auto-wiring도 없이 — 단지 값을 전달하는 지루하고 예측 가능한 코드일 뿐이다."*

챕터 에피그래프("훌륭한 dependency 설정은 지루하게 느껴진다")가 결론에서 문자 그대로 실현된다.

> *"의존성 계층을 거꾸로 뒤집음으로써 ABC problem을 완전히 피했다. 타입은 자신의 직접 의존성만 알고, 의존성의 의존성은 알지 않는다. 이는 우리 코드를 느슨하게 결합되고 추론하기 쉽게 유지한다."*
>
> *"secret sauce는 놀랍도록 단순하다 — 의존성을 설정하는 자가 필요한 모든 것에 접근한다. AppSetup이 ABC 규칙을 깨므로 다른 어떤 것도 그럴 필요가 없다."*
>
> *"더 큰 의존성 계층과 함께 오는 까다로운 경우도 처리했다. 시작 시 사용 가능하지 않은 의존성, 그리고 절대 필요하지 않을 수 있는 optional 의존성을 모두 factory를 사용해 처리했다."*
>
> *"그 위에 우리는 컴파일러 플래그를 흩뿌리지 않도록 보장했다. 그것들은 한 예측 가능한 위치에 정리되어 있다."*

**그리고 대가를 스스로 인정하며 그것이 왜 가치 있는지를 결론으로 못 박는다.**

> *"물론 이 접근법은 싱글턴을 모든 곳에 흩뿌리는 것보다 더 장황하다. 그러나 앱이 실제로 어떻게 작동하는지 이해하려고 할 때 장황함이 마법보다 낫다."*

§7.5.10이 "값 전달은 보상받는다"고 약속했던 것이 여기서 실제로 지불된 셈이다 — 대가는 코드 줄 수, 보상은 이해 가능성.

**그리고 이 챕터 자신도 한계를 인정하며 다음 챕터로 넘어간다.**

> *"현재 우리는 한 위치에서 단일 기능에 대한 의존성을 처리하고 있다. 그것은 영원히 확장되지 않을 것이다. 다음으로 앱이 더 커지고 모든 것을 미리 준비할 수 없을 때 무슨 일이 일어나는지 다룰 것이다. 다음 챕터에서는 의존성 트리를 확장하고, 모듈을 가로지르는 것까지 더 큰 범위에서 의존성을 다루는 방법을 추론할 것이다."*

## § 8.11 우리가 다룬 것 — 챕터 요약

> - "프레임워크 없이 바닐라 코드를 사용한 dependency injection 기법"
> - "의존성을 평범한 방식으로 전달하는 것이 멋진 솔루션보다 이해하기 더 쉬운 방법"
> - "더 나은 코드 조직을 위해 애플리케이션 진입점에 컴파일러 플래그 통합"
> - "production, staging, 테스팅 환경 사이를 전환하기 위해 NetworkTransport 같은 인터페이스 사용"

> **The ABC Problem**
> - "ABC problem은 타입이 자신의 transitive dependency(의존성의 의존성)를 인지할 때 발생한다"
> - "transitive dependency는 강한 결합을 만들고 코드를 유지보수하기 더 어렵게 한다"
> - "ABC problem을 해결하는 것은 타입이 자신의 직접 의존성만 알게 하는 것을 의미한다"
> - "해결책은 초기화 동안 의존성 계층을 거꾸로 뒤집는 것을 포함한다"
> - "setup 클래스(AppSetup)가 ABC 규칙을 깨므로 다른 어떤 타입도 그럴 필요가 없다"

> **Lazy instantiation and factories**
> - "Factory는 앱 초기화 동안 사용 가능하지 않은 의존성을 처리한다"
> - "의존성이 런타임 값을 요구할 때 factory를 사용하라"
> - "initializer를 통해 사용 가능한 의존성으로 factory를 미리 로드하라"
> - "런타임 전용 의존성을 factory 메서드의 인자로 전달하라"
> - "이는 적절한 dependency injection을 유지하면서 eager 초기화를 피한다"

이 요약을 그대로 축약하면, **결정은 플래그가 하고, 은폐는 인터페이스가 한다** — 는 한 줄로 압축된다. 컴파일러 플래그(`#if DEBUG`)가 AppSetup 안에서 어떤 concrete transport(`ProductionTransport`/`StagingTransport`)를 고를지 **결정**하고, `NetworkTransport` 인터페이스는 그 선택이 NetworkClient 이하의 나머지 코드에 **드러나지 않도록 감추는** 역할이다.

> **오답 지점 — 챕터 요약을 되짚는 마지막 질문.** "컴파일러 플래그와 인터페이스가 각각 무슨 역할을 하는가"를 묻는 요약형 질문에 "인터페이스가 concrete transport를 스스로 판단하고 컴파일러 플래그는 결과를 로깅만 한다"고 답해 오답이었다. 인터페이스에는 판단 능력이 없다 — `NetworkTransport`는 그저 메서드 시그니처의 집합일 뿐, "지금이 DEBUG 빌드인지"를 스스로 알 방법이 없다. **선택은 항상 `#if DEBUG` 분기가 AppSetup 한 곳에서 하고, 인터페이스는 그 선택의 결과(구체 타입)를 아래쪽 코드로부터 감추는 벽 역할만 한다.** §7.3의 NetworkTransport(교체 가능성)와 §8.5의 컴파일러 플래그(선택 지점)가 이 챕터 요약에서 한 문장으로 합쳐지는 자리이며, 둘의 역할을 뒤바꿔 이해하면 이 요약 전체가 흔들린다.

---

## Book 0의 일곱 번째 Mastered 영역, 그리고 다음 챕터

Sane DI(Ch8)는 2026-08-18 하루 일곱 세션(전 개념 22/22 첫 출제 → 🔴 7개 `/lesson` 재교육 → 🔴 7개 drill-weak 전부 해소 → 🟡 22개 전수 재확인(21/22 정답, §8.7.3 신규 🔴) → §8.7.3 재교육 + 2연속 드릴(여전히 미해소) → §8.7.3 3회차 오답(뿌리 원인 확정) → §8.7.3 지엽 판정으로 수동 🟢, 시드 22/22 전부 🟢·🟦 Mastered)를 거쳐 Book 0의 **일곱 번째 🟦 Mastered 영역**에 도달했다 — Briefing→Plan(Ch2)·HDD Plan→Code(Ch3)·HDD Strategic(Ch4)·System-Wide Testing(Ch5)·Cross-Domain Testing(Ch6)·DI Foundations(Ch7)에 이어서다.

**이 챕터가 남긴 학습 프로필은 §7과 다르다.** §7에서는 "절이 무엇을 말하는가"를 묻는 메타 질문에서만 걸렸다. §8에서는 그 패턴에 더해 **하나의 개념(§8.7.3)이 세 번 연속으로, 서로 다른 층위(생성 순서 → 타입 관계 → public 계약)를 오가며 틀렸다.** 이는 앞의 세션들처럼 "질문 각도의 문제"가 아니라, **AppSetup 내부의 사적 조직과 CourseService의 공개 계약을 구분하는 개념 자체가 안드로이드 개발자에게는 자연스럽지 않은 구분선이었다는 신호**로 읽는 편이 정확하다 — 아래 "나누고 싶은 이야기"에서 이 지점을 Hilt 모듈 구조로 다시 짚는다.

다음 챕터(Ch9 — DI Larger Scale)는 §8.10이 예고한 지점에서 시작한다. *"앱이 더 커지고 모든 것을 미리 준비할 수 없을 때"*, 그리고 §7.5.5·§7.5.4가 다뤘던 **모듈 경계를 가로지르는 의존성**을 이 챕터의 도구(뒤집힌 그래프, secret sauce, factory)로 어떻게 확장하는지가 주제다.

---

## 나누고 싶은 이야기

### 1. Android/Hilt로 옮기면 — AppSetup은 이미 익숙한 패턴이다

Ch7과 달리 Ch8은 **거의 전부가 그대로 안드로이드에 옮겨진다.** 이유는 간단하다 — Hilt/Dagger 자체가 정확히 이 챕터가 손으로 하는 일(그래프를 뒤집어 bottom-up으로 조립, 위반을 한 곳에 집중)을 컴파일 타임 코드 생성으로 자동화한 도구이기 때문이다.

- **AppSetup의 `setupCourseService()`/`setupMarketplace()` → Hilt의 `@Module` `@Provides` 함수들.** 정확히 같은 역할이다.

  ```kotlin
  @Module @InstallIn(SingletonComponent::class)
  object CourseModule {
      @Provides
      fun provideNetworkClient(transport: NetworkTransport, store: Store): NetworkClient =
          NetworkClient(transport, store)              // = setupNetworkClient()

      @Provides
      fun provideCourseService(tutorAPI: TutorAPI, todoAPI: TodoAPI, calendar: Calendar): CourseService =
          CourseService(tutorAPI, todoAPI, calendar)    // = setupMarketplace() 안의 CourseService 조립부
  }
  ```

  Hilt가 대신해주는 것은 **§8.3.1의 순서 계산**뿐이다 — "이 `@Provides` 함수를 호출하려면 무엇이 먼저 준비돼야 하는가"를 그래프로 풀어 bottom-up 순서를 스스로 찾아준다. §8.3의 "그래프를 뒤집어 손으로 순서를 정하는" 작업이 Hilt에서는 컴파일 타임 코드 생성으로 자동화된 것뿐, **원칙(직접 의존성만 요구하는 initializer, transitive dependency 은폐)은 완전히 동일하게 적용된다.**

- **§8.2의 ABC problem → Hilt를 쓴다고 사라지지 않는다.** Hilt는 "누가 무엇을 조립하는가"를 자동화할 뿐, "타입이 자기 시그니처에 무엇을 노출하는가"는 여전히 개발자의 설계다.

  ```kotlin
  // ABC problem 그대로 재발 — Hilt가 있어도 CourseService가 NetworkClient를 안다
  class CourseService @Inject constructor(
      private val networkClient: NetworkClient,  // 직접 쓰지도 않는데 시그니처에 등장 (위반)
  ) {
      private val tutorAPI = TutorAPI(networkClient)
      private val todoAPI = TodoAPI(networkClient)
  }

  // §8.3.2가 요구하는 형태 — 직접 의존성만
  class CourseService @Inject constructor(
      private val tutorAPI: TutorAPI,
      private val todoAPI: TodoAPI,
      private val calendar: Calendar,
  )
  ```

  **Hilt는 배선을 대신해줄 뿐, "무엇을 배선해야 하는가"는 여전히 §8.2~§8.3의 규칙을 따라야 한다.** Pillo에서 `MedicationViewModel`이 `DrugBankApi`를 직접 받고 있다면, 그것을 실제로 쓰지도 않으면서 `MedicationRepository`를 통해서만 필요하다면 정확히 이 챕터의 위반이다.

- **§8.6.1의 "위반은 AppSetup 한 곳에 가둔다" → Hilt `@Module`의 존재 이유 그 자체.** Hilt 모듈이 여러 개(`NetworkModule`, `DatabaseModule`, `RepositoryModule`)로 나뉘는 것이 §8.7.3의 `setupMarketplace()`/`setupNetworkClient()` 분리와 **동형이다.** 그리고 §8.7.3의 3연속 오답이 여기서 실질적인 교훈으로 이어진다 — **`@Module`을 여러 개로 쪼개는 기준은 "그래야만 컴파일된다"가 아니라 순수 조직상 편의**라는 것. `NetworkModule`과 `RepositoryModule`을 하나로 합쳐도 앱은 똑같이 컴파일되고 똑같이 동작한다. 나누는 이유는 오직 하나 — 각 모듈 파일이 감당할 수 있는 크기로 유지하기 위해서다. AppSetup 내부의 사적 조직(몇 개 함수·모듈로 나누는가)과 각 타입의 `@Inject constructor` 공개 계약이 서로 다른 층위라는 §8.7.3의 교훈이, Hilt에서는 "모듈 개수"와 "생성자 시그니처"의 관계로 정확히 재현된다.

- **§8.9의 PaymentsFactory → Dagger의 `AssistedInject`와 정확히 같은 패턴.** 이 챕터에서 가장 깔끔하게 옮겨지는 지점이다. "setup 시점에 아는 값은 initializer로, 호출 시점에만 아는 값은 메서드 인자로"라는 §8.9.1의 규칙이 `@AssistedInject`의 존재 이유 그 자체다.

  ```kotlin
  class PaymentsFactory @AssistedInject constructor(
      private val networkClient: NetworkClient,        // setup 시점에 Hilt가 주입 (= initializer)
      @Assisted private val provider: PaymentProvider,  // 호출 시점에 사용자가 공급 (= method arg)
  ) {
      // ...
  }

  @AssistedFactory
  interface PaymentsFactoryFactory {
      fun create(provider: PaymentProvider): PaymentsFactory   // = makePayments(provider:)
  }
  ```

  Pillo라면 `DoseReminderFactory`가 `NotificationManager`(setup 시점에 알려짐)는 `@Inject`로, 사용자가 방금 고른 `reminderTime`(런타임 값)은 `@Assisted`로 나누는 것이 그대로 이 패턴이다. **"할 수 있는 것을 initializer에, 나머지를 method arg에"** 라는 §8.9.1의 한 줄이 사실상 Dagger assisted injection의 설계 원칙 요약이다.

- **§8.4의 테스트 환경 → Hilt의 `@TestInstallIn`.** production 모듈을 테스트 모듈로 통째로 교체하는 Hilt의 표준 패턴이 §8.4가 손으로 한 일(같은 조립 로직, `MockTransport`만 교체)의 자동화 버전이다. `@Provides`가 만드는 그래프 구조는 그대로 두고 leaf 하나만 갈아끼운다는 점에서 개념적으로 동일하다.

### 2. §8.7.3의 반복 오답이 말해주는 것 — Hilt가 감춰버린 층위

이 챕터에서 유일하게 3연속 서로 다른 각도로 틀린 §8.7.3은 우연이 아닐 가능성이 높다. **Hilt/Dagger를 매일 쓰는 안드로이드 개발자에게는 "`@Module`을 몇 개로 나누는가"라는 질문 자체가 평소에 던질 이유가 없는 질문이다** — 어차피 그래프는 어노테이션 프로세서가 조립하므로, 모듈 파일 개수는 순수 취향과 팀 컨벤션의 문제로 느껴지고, "이게 타입 관계에 영향을 주는가?"를 의심할 계기가 생기지 않는다.

반면 이 챕터는 그 조립을 **손으로** 하기 때문에, `setupMarketplace()`와 `setupNetworkClient()`가 같은 파일 안의 두 함수라는 사실이 훨씬 노출되어 있다 — 그리고 바로 그 노출 때문에 "이 둘을 합치면 뭔가 달라지지 않을까"라는 의심이 자연스럽게 생긴다. 세 번의 오답(생성 가능 여부 → ABC problem 재발 → initializer 변경)은 전부 **"AppSetup 내부의 조직이 CourseService의 외부 계약에 영향을 준다"** 는 하나의 착각에서 갈라져 나온 가지였다. Hilt 사용자에게 이 착각이 생기기 쉬운 이유는, 평소 `@Module`을 나누거나 합치는 리팩터링을 컴파일러가 전부 검증해주므로 **"나눠도 합쳐도 아무 일도 안 일어난다"는 사실을 직접 겪어볼 기회가 적기 때문**이다. 이 챕터를 손으로 짚어본 것이 그 기회를 메운 셈이다.

### 3. §7 → §8의 연결 — 미뤄뒀던 것을 전부 회수하는 챕터

§7 정리 문서가 예고했던 수렴이 여기서 실제로 일어난다. §7.4의 "컴파일러 플래그를 중앙화하자"와 §7.5.8의 "싱글턴을 leaf에서 시작해 위로 밀어 올리면 진입점에서만 인스턴스화된다"가 **같은 장소(AppSetup, 진입점)** 로 수렴한다고 예고했는데, §8.5(컴파일러 플래그를 AppSetup에)와 §8.3.1(bottom-up으로 밀어 올려 CourseService를 마지막에)이 정확히 그 두 문장을 코드로 실현한다.

```
§7.4    "플래그를 중앙화하자"        →  §8.5   AppSetup의 #if DEBUG 한 곳
§7.5.5  "기능에서 처음부터 DI 지원"   →  §8.3   AppSetup이 기능 전체를 조립
§7.5.8  "leaf에서 시작해 위로"        →  §8.3.1 transport → ... → CourseService 순서
```

그리고 §8 자신도 §7과 같은 화법으로 절을 닫는다 — **원칙을 다 세운 뒤, 그 원칙이 만드는 대가를 스스로 인정하고 다음 챕터로 미룬다.** §7.5.10이 "값 전달은 보상받는다"며 §8~§9로 증명을 미뤘던 것처럼, §8.10은 "장황함이 마법보다 낫다"며 **모듈 경계를 넘는 확장**을 §9로 미룬다. 이 책이 매 챕터 끝에서 "이 원칙이 다음 규모에서도 버티는가"라는 질문을 스스로 던지고 다음 챕터로 넘기는 태도가, §6→§7→§8을 지나 §8→§9에서도 그대로 이어진다.
