# UI 07·08 이식 가능 기능과 복잡한 UI 정리: 진실 공급원을 UI 밖으로 옮기고, 그렇게 만든 기능으로 복잡한 화면을 짓는다

> Mobile System Design 시리즈 Book 1 (Modular UI & UI Architectures) 의 일곱 번째와 여덟 번째 챕터(Delivering Self-Sufficient Features, Part III: Making Features Portable / Taming Complex UI).
> 핵심 질문: *`CourseView`가 쥐고 있던 로딩 상태와 동기화 판단을 UI 밖으로 옮기면 기능은 무엇을 새로 할 수 있게 되는가? 그리고 그런 기능 여러 개를 한 화면에 모을 때, 복잡성을 부모 뷰 하나에 몰아넣지 않으려면 무엇을 어디로 옮겨야 하는가?*

---

## 도입: 두 챕터는 한 가지 질문의 두 규모다

UI-05·06 정리 문서는 Ch6이 끝난 시점의 `CourseView`를 이렇게 요약했다. `id`만 받으면 어디서든 스스로 로드하고 오류도 스스로 표시하지만 로딩 상태와 `Course`의 현재 값을 **뷰가 소유한다.** Ch7은 이 소유권을 모델 도메인의 새 타입 `CourseSession`으로 옮긴다. Ch8은 그렇게 완성된 자기 충족 기능을 여러 개 한 화면에 모으는 상황을 다룬다.

두 챕터가 던지는 질문은 같다. **"이 복잡성은 어디에 살아야 하는가?"** Ch7은 기능 하나 안에서 이 질문을 던지고 답으로 판단을 UI 도메인에서 모델 도메인으로 옮긴다. Ch8은 화면 하나 안에서 같은 질문을 던지고 답으로 책임을 부모 뷰에서 자식 기능으로 밀어낸다. Ch8의 에피그래프가 두 챕터를 함께 요약한다.

> **(p.153)** ***"복잡성은 어딘가에 살아야 합니다. 현명하게 고르세요."***

| 챕터 | 규모 | 순진한 구조 | 책의 해법 |
|---|---|---|---|
| Ch7 (Part A) | 기능 하나 | `CourseView`가 상태를 보관하고 동기화를 판단한다 | `CourseSession`이 모델 도메인에서 진실 공급원이 되고 뷰는 바인딩만 한다 |
| Ch8 (Part B) | 화면 하나 | 부모 뷰가 모든 자식의 로딩, 오류, 이벤트를 처리한다 | 자식을 자기 충족 기능으로 만들고 부모는 레이아웃만 맡는다 |

Ch7은 새 개념을 많이 도입하지 않는다. 대신 UI-01 정리 문서의 원칙 3("기능을 명령줄 도구라고 상상하라")과 §1.3.3("비즈니스 도메인은 두껍게, UI 도메인은 가볍게")을 `Course` 기능에 실제로 적용한다. Ch8은 코드 리스팅이 `SettingsView`의 속성 목록 두 개뿐인 설계 챕터이며 UI-03 정리 문서의 뷰 분류와 Ch5~7의 자기 충족 기능을 화면 수준으로 확대해 적용한다.

---

## 두 챕터가 내리는 결정 한눈에 보기

| 절 | 결정 | 책이 댄 근거 |
|---|---|---|
| §7.0 | 기능을 **이식 가능**하게 만드는 것을 목표로 삼는다 | UI 마이그레이션, 새 플랫폼, 백그라운드 실행, 단위 테스트가 모두 이식성에 달려 있다 |
| §7.2 | 로직을 `CourseService`에 넣지 않는다 | `CourseService`는 API 호출에 집중해야 하고 여러 코스를 다룬다 |
| §7.2.1 | 단일 `Course`를 맡는 `CourseSession`을 **모델 도메인에** 만들고 ViewModel이라고 부르지 않는다 | UI 프레임워크의 일부가 되면 단위 테스트와 다른 프레임워크에서 쓸 수 없다 |
| §7.2.3 | 변경 감지는 `didSet`으로 한다 | `didSet`은 SwiftUI가 아니라 Swift의 기능이다 |
| §7.3.3 | 디바운싱과 취소를 `CourseSession` 안에 추가한다 | 뷰를 건드리지 않고 동기화 로직을 성숙시킬 수 있다 |
| §8.1 | 스크롤 메뉴를 `CourseView` 밖의 `PaginationView`에 둔다 | 메뉴는 여러 코스를 알아야 하고 `CourseView`는 코스 하나만 안다 |
| §8.2 | 대시보드의 자식을 자기 충족 기능으로 만들고 그 안에서 UI를 한 층 더 분리한다 | 부모에 몰린 복잡성을 여러 층에 나눠 어느 곳도 위험 수준을 넘지 않게 한다 |
| §8.3.3~4 | UI 기반 협력은 부모에, 데이터 동기화는 모델 계층에 둔다 | UI가 없어도 필요한 협력인가로 가른다 |
| §8.4 | 시각적 경계가 없는 리스트도 섹션 타입으로 나누고 행 렌더링은 부모에게 넘긴다 | 리스트가 모든 것을 보여 준다고 해서 리스트 타입이 모든 것을 해야 하는 것은 아니다 |

표의 결정들이 공유하는 태도는 하나다. **"이 로직이 UI 없이도 돌아야 하는가?" 그리고 "이 로직이 없으면 지금 어느 타입이 무거워지는가?"라고 묻고, 답에 따라 로직을 둘 자리를 고른다.**

---

# Part A. Ch7 자기 충족 기능 III: 기능을 이식 가능하게 만든다

## § 7.0 이식 가능 기능이란 무엇인가

> **(p.135)** *"이식 가능 기능(Portable features): UI 패턴, UI 아키텍처, UI 패러다임과 무관하게 플랫폼 전반에서 작동하는 기능."*

책은 Ch6의 선택을 잘못이라고 하지 않는다. *"이 접근은 단일 플랫폼, 패턴, 패러다임이라면 괜찮습니다"*라는 조건을 붙여 두고 그 조건이 깨지는 경우를 네 가지로 나열한다.

1. **UI 마이그레이션.** 팀은 패턴에서 패턴으로, 프레임워크에서 프레임워크로 옮겨 가고 Apple과 Google이 그 이동을 압박하기도 한다. 기능이 특정 UI 구현에 결합되어 있으면 마이그레이션이 기하급수적으로 어려워진다.
2. **새 플랫폼.** Watch 앱이나 VR 앱에 같은 기능을 제공할 수 있다. SwiftUI처럼 멀티플랫폼인 패러다임도 **UI가 없는 타깃**은 포괄하지 못한다.
3. **백그라운드 실행.** 기능 로직이 UI 도메인에 많을수록 백그라운드에서 돌리기 어려워진다.
4. **테스트.** 책이 가장 크게 다루는 이유다.

> **(p.136)** *"더 큰 문제는 기능을 단위 테스트하기가 점점 더 어려워진다는 것입니다. (...) 제가 인터뷰한 개발자 대부분은 UI 테스트를 작성하지 않으며, 수동 확인은 자동화하기 어렵고 수많은 기기에서 재현하기도 어렵습니다."*
>
> ***"그래서 우리는 위험을 감수합니다. 기능이 덜 이식 가능할수록, 테스트되지 않았을 가능성이 더 높습니다."***

이 문장이 Ch7의 논지를 압축한다. 이식성은 "언젠가 플랫폼을 늘릴 때"를 위한 대비처럼 들리지만 책은 그것을 **지금 당장의 테스트 가능성**과 같은 문제로 묶는다. 그래서 플랫폼을 늘릴 계획이 없는 팀에게도 이 챕터가 해당한다.

책은 이식성이 정도의 문제라는 점도 밝힌다. *"UI 기능을 완전히 이식 가능하게 만들 수는 없습니다. UI는 필요하기 때문입니다. 하지만 아주 가까이 갈 수는 있습니다."* 뒤의 §7.2.4에서 `CourseView`가 하는 일이 거의 남지 않지만 0이 되지는 않는다. 남는 일은 상태를 화면에 붙이는 접착(glue)이고 책은 이 접착이 플랫폼마다 다를 수밖에 없다고 본다.

---

## § 7.1 현재 구조와 두 가지 문제

책은 Ch6이 끝난 시점의 흐름을 시퀀스 다이어그램(Figure 7.2)으로 다시 그리고 각 단계를 "UI domain"과 "Model domain"으로 묶는다.

| 단계 | Figure 7.2의 내용 | 도메인 |
|---|---|---|
| 1~2 | `CourseView`가 `CourseService`에 `Course`를 요청하고 받는다 | UI ↔ 모델 |
| 3 | `CourseView`가 자신의 `Course`를 갱신한다 | **UI** |
| 4~5 | `CourseUI`가 화면을 채우고 사용자가 `Course`를 변경한다 | UI |
| 6 | `Course` 바인딩이 `CourseView`의 상태를 갱신한다 | **UI** |
| 7 | `CourseView`가 `update(course:)`를 호출한다 | **UI** → 모델 |

굵게 표시한 3, 6, 7단계가 이 챕터의 표적이다. 세 단계 모두 **판단**을 포함한다. 받은 값을 보관할지, 변경을 받아들일지, 백엔드를 언제 갱신할지를 모두 `CourseView`가 결정하고 `CourseService`는 요청을 실행할 뿐이다.

### § 7.1.1 이식 가능하지 않은 이유: 모델 도메인의 빈자리

> **(p.139)** *"대부분의 동작이 'UI 도메인'에 있으므로, `CourseView`가 이 작업의 **두뇌**라는 것을 알 수 있습니다. (...) 한 가지 중요한 구분은 `CourseView`가 `Course` 데이터의 진실 공급원, 즉 **소유자**라는 점입니다."*
>
> *"아마 무의식적으로, 우리가 이 상황에 이른 이유는 `CourseService`가 `id`만 받으면 어떤 `Course`든 로드할 수 있기 때문입니다. 그러나 모델 도메인의 어떤 것도 **단일** `Course`를 보관하거나 그것에 바인딩하는 방법을 제공하지 않습니다. 그래서 `CourseView`가 단일 `Course`에 바인딩하는 부담을 떠안습니다."*

즉 문제는 `CourseView`가 욕심을 낸 것이 아니라 **모델 도메인에 빈자리가 있었던 것**이다. `CourseService`는 "여러 코스 중 아무거나"를 다루고 "지금 사용자가 보고 있는 이 코스 하나"를 대표하는 타입은 없었다. 뷰가 그 빈자리를 메웠을 뿐이므로 해법은 모델 도메인에 그 자리를 만들어 주는 것이다.

### § 7.1.2 시스템으로 단위 테스트할 수 없는 이유

| 테스트 대상 | 지금 가능한가 | 이유 |
|---|---|---|
| `CourseService.loadCourse(id:)`가 코스를 반환한다 | 가능 | 모델 도메인의 메서드다 |
| 로딩 후 기능이 `loaded` 또는 `failed`에 있다 | **불가능** | 상태 전이를 `CourseView`가 수행한다 |
| `CourseService.update(course:)`가 동작한다 | 가능 | 모델 도메인의 메서드다 |
| 변경이 일어나면 `update(course:)`가 호출된다 | **불가능** | 호출 판단을 `CourseView`가 한다 |

가능한 두 줄은 부품 테스트이고 불가능한 두 줄은 **부품을 엮은 시스템의 테스트**다. `.docs/05-System-Wide-Testing-정리.md`의 논지(mock을 적게 쓰고 큰 표면을 테스트한다)에 비추어 보면, 지금 구조에서는 "큰 표면"의 입구가 뷰 안에 있어서 단위 테스트가 그 입구에 닿지 못한다.

---

## § 7.2 로직을 UI 도메인 밖으로

> **(p.140)** *"처방은 단순합니다. (...) 이 비즈니스 로직을 평이한 말로 하면 이렇습니다. **'기능은 `Course`를 로드하고, 사용자가 `Course`를 갱신하면 백엔드도 갱신되어야 한다.'**"*

이 문장에는 화면, 스피너, 바인딩 같은 단어가 하나도 없다. 그러므로 이 문장을 구현하는 코드도 그런 것에 의존할 필요가 없다는 것이 이 절의 출발점이다.

**왜 `CourseService`에 넣지 않는가.** 책은 *"`CourseService`는 API 호출에 집중해야 하고, 많은 코스를 로드할 수도 있습니다"*라고 답한다. 판단 기준은 **다중성**이다. `CourseService`는 코스 여러 개를 다루는 상태 없는(stateless) 서비스이고 새로 필요한 것은 코스 **하나**의 현재 상태를 쥐는 상태 있는(stateful) 객체다. 둘을 합치면 UI-01 정리 문서 §1.4.2가 경계한 상황, 즉 서비스가 `isLoading`과 `courses`를 들고 그 동기화를 책임지는 상황이 재현된다. 책도 Note에서 *"핵심은 로직을 모델 도메인으로 옮긴다는 점"*이라고 한정한다. 새 타입은 수단이고 목적은 판단의 위치를 옮기는 것이다.

### § 7.2.1 진실 공급원 모델 도입: `CourseSession`

> **(p.140)** *"이 새 타입을 `CourseSession`이라고 부르겠습니다. `CourseSession`은 **단일** `Course`에 대한 로딩, 오류 상태, 변경을 처리합니다. `CourseView`는 더 이상 `Course`를 보관하지 않습니다. `CourseSession`(모델 도메인)을 `CourseUI`(UI 도메인)에 연결, 즉 **바인딩**할 뿐입니다."*

```
Ch6:  CourseView ──▶ CourseUI
          └────────▶ CourseService

Ch7:  CourseView ──▶ CourseUI                          (UI 도메인)
          └────────▶ CourseSession ──▶ CourseService   (모델 도메인)
```

> **(p.141)** *"새 `CourseSession`을 UI 도메인이 아니라 **모델** 도메인에 명시적으로 둔다는 점에 주목하세요. (...) 예를 들어 `CourseSession`을 SwiftUI의 일부로 만든다면, 이 로직을 단위 테스트나 다른 UI 프레임워크에 둘 수 없을 것입니다."*
>
> *"기능을 명령줄 도구라고 상상하면 로직을 어디에 둘지 추론하는 데 도움이 된다고 했던 것을 기억하시나요? 같은 원칙입니다."*

명령줄 도구로 `Course` 기능을 돌린다고 상상하면 "코스 하나를 열고 할 일을 토글하고 서버와 동기화한다"는 흐름은 화면 없이도 성립해야 한다. Ch6 구조에서는 중간 단계(상태 보관, 변경 감지, `update` 호출)가 `CourseView` 안에 있어서 명령줄 도구가 이 기능을 쓸 수 없다. `CourseSession`이 생기면 명령줄 도구는 세션을 만들어 `startLoadCourse()`를 부르고 `state`를 바꾸기만 하면 된다.

> **(p.142) Note** *"이것이 `CourseSession`을 'ViewModel'이라고 부르지 않는 이유이기도 합니다. `CourseSession`은 뷰를 전혀 알지 못하며 뷰가 아닌 코드에서도 사용할 수 있습니다."*

ViewModel이라는 이름은 "특정 뷰를 위해 존재한다"는 뜻을 전달한다. 이름이 관계를 거꾸로 표현하면 나중에 누군가 "ViewModel이니까" 하며 스피너 표시 여부 같은 UI 관심사를 이 타입에 넣기 쉽다. UI-02 정리 문서의 원칙 5("용도가 아니라 본질로 이름 짓기")가 모델 타입에도 적용되는 셈이다. 이 Note는 UI-05·06 정리 문서의 예측도 확인한다. 책의 `CourseSession`은 안드로이드의 `ViewModel`에 대응하지 않으며 그 아래 계층에 대응한다(아래 "안드로이드로 옮기면" 참고).

### § 7.2.2 무게가 오른쪽으로 옮겨진다

Figure 7.4는 같은 흐름을 아홉 단계로 다시 그린다. Figure 7.2에서 판단이 들어 있던 3, 6, 7단계는 Figure 7.4의 4단계(`CourseSession`이 자기 상태를 갱신한다), 8단계(바인딩이 `CourseSession`의 상태 변경을 일으킨다), 9단계(`CourseSession`이 `update`를 호출한다)로 옮겨졌다. 8단계는 UI에서 출발하지만 `CourseView`는 변경을 **전달만** 하고 그 변경으로 무엇을 할지는 `CourseSession`이 정한다. UI 도메인에 남은 단계에는 판단이 없다.

### § 7.2.3 `CourseSession` 구현

> **(p.143)** *"이제는 `CourseSession`의 `didSet` 메서드로 변경을 감지할 수 있습니다. `didSet`은 속성이 갱신되면 어떤 코드를 실행하는 **Swift의** 방식이지, SwiftUI의 방식이 아닙니다. Swift는 SwiftUI보다 더 많은 플랫폼과 타깃에서 실행되므로 이는 좋은 일입니다."*

```swift
@Observable
final class CourseSession {

    // The enum now belongs to CourseSession, it's not UI specific
    enum LoadState {
        case loading
        case loaded(Course)
        case failed(Error)
    }

    private let id: UUID?
    private let courseService: CourseService?

    // CourseSession now has the loading state. Disconnected from UI.
    var state: LoadState = .loading {
        didSet {
            guard case let .loaded(newCourse) = state,
                  case let .loaded(oldCourse) = oldValue,
                  oldCourse.id == newCourse.id else { return }
            Task {
                // Update course if mutated
                try await courseService?.update(course: newCourse)
            }
        }
    }

    init(id: UUID, courseService: CourseService) {
        self.id = id
        self.courseService = courseService
    }

    init(course: Course) {
        self.id = course.id
        self.courseService = nil
        self.state = .loaded(course)
    }

    // CourseSession now handles the loading instead of CourseView
    func startLoadCourse() async {
        guard let courseService, let id else {
            assertionFailure("Expected a CourseService and id instance")
            return
        }
        do {
            state = .loading
            state = .loaded(try await courseService.loadCourse(id: id))
        } catch let error {
            state = .failed(error)
        }
    }
}
```
**Listing 7.1** CourseSession 정의 (p.143~144)

**`LoadState`가 이사했다.** Ch6에서 `CourseView` 안에 있던 enum이 세 케이스를 그대로 유지한 채 `CourseSession`으로 옮겨졌다. Ch6에서 세 상태를 화면 표시가 아니라 **기능의 상태**를 기준으로 골랐기 때문에 위치만 바꿔도 의미가 깨지지 않는다.

**`didSet`의 `guard`가 갱신 조건을 정한다.**

| 전이 | `update` 호출 | 이유 |
|---|---|---|
| `loading` → `loaded` | 하지 않는다 | 서버에서 막 받은 값을 서버에 되돌려 보내지 않는다 |
| `loaded(A)` → `loaded(A')` (같은 `id`) | **한다** | 사용자 변경이다 |
| `loaded(A)` → `loaded(B)` (다른 `id`) | 하지 않는다 | 다른 코스로 바뀐 것이지 변경이 아니다 |
| `loaded` → `loading` 또는 `failed` | 하지 않는다 | `state`가 `loaded`가 아니다 |

Ch6은 이 판단을 바인딩의 `set` 클로저 안에서 했고 Ch7은 같은 판단을 **어떤 경로로 상태가 바뀌든** 적용되는 `didSet`으로 옮겼다. 이제 백그라운드 작업, 푸시 알림 처리, 테스트가 `state`를 바꿔도 같은 규칙이 적용된다.

**이니셜라이저가 두 개다.** `init(course:)`는 서비스 없이 고정된 코스를 보여 주는 용도(프리뷰, 목업)로 읽힌다. `startLoadCourse()`는 Ch6 `CourseView`의 `.task` 본문을 거의 그대로 옮긴 것이며 이제 뷰 없이도 호출될 수 있다.

> **읽을 때 주의할 지점: 같은 값으로 갱신해도 호출된다.** Ch6은 `guard updatedCourse != course`로 같은 값을 걸렀지만 Listing 7.1의 `guard`는 `id`만 비교한다. §7.3.4에서 책이 예로 드는 "같은 상태로 갱신하면 아무것도 하지 않는지 확인하는 테스트"는 Listing 7.1과 7.3의 코드로는 통과하지 못하며 동등성 검사를 추가해야 한다.

### § 7.2.4 `CourseView` 단순화

```swift
struct CourseView: View {

    // This is now part of CourseSession
    // @State private var state: LoadState = .loading

    // We bind to a CourseSession so the UI will respond to changes of its properties
    @Bindable private var courseSession: CourseSession

    init(courseSession: CourseSession) {
        self.courseSession = courseSession
    }

    var body: some View {
        // We inspect CourseSession's state to decide which view to show.
        switch courseSession.state {
        case .failed(let error):
            Text("Error: " + error.localizedDescription)
        case .loaded(let course):
            CourseUI(
                course: Binding(
                    get: { course },
                    // When the user mutates course, forward the changes. CourseSession will keep this in sync with the backend
                    set: { course in courseSession.state = .loaded(course) }
                )
            )
        case .loading:
            VStack {
                Text("Loading")
                ProgressView()
            }.task { await courseSession.startLoadCourse() }
        }
    }
}
```
**Listing 7.2** CourseView가 이제 CourseSession에 의존한다 (p.145)

Ch6과 비교하면 `@State private var state`가 사라졌고, `.task` 안의 `do`/`catch`가 `startLoadCourse()` 호출 한 줄로 줄었으며, 바인딩의 `set` 안에 있던 비교와 `update` 호출이 `courseSession.state = .loaded(course)` 한 줄로 바뀌었다. `CourseUI`는 이번에도 수정하지 않았다.

> **(p.146)** *"`CourseView`는 **더 두꺼운 비즈니스 모델 위의 얇은 UI 베니어(veneer)**가 됩니다."*
>
> *"이 접근의 한 가지 단점은 (...) 간접 계층이 하나 더 생긴다는 것입니다. (...) `CourseView`를 보면 더 이상 부가 가치가 많지 않습니다. 하지만 어느 정도 가치는 있습니다. 코스와 그 로딩 상태를 UI에 접착하는 것입니다."*
>
> **Note** *"약간의 상상력을 더하면 `CourseView`를 수많은 타입을 지원하는 범용 로딩 뷰로 바꾸는 것도 가능할 것입니다."*

`CourseView`에 남은 코드에서 `Course`에 특화된 부분은 `CourseUI`를 호출하는 한 줄뿐이다. 상태에 따라 스피너, 오류 문구, 콘텐츠 중 하나를 고르는 나머지 코드는 어떤 기능이든 같다. Ch5에서 "로딩 컨테이너"로 분리한 역할이 이제 기능과 무관한 부품이 될 만큼 얇아졌다.

> **읽을 때 주의할 지점: 공개 API가 바뀌었다.** Ch6 §6.5.5는 `CourseView(id:courseService:)` 이니셜라이저로 API 안정성을 지켰지만 Listing 7.2는 `init(courseSession:)` 하나만 가진다. 책은 이 변화를 언급하지 않는다. 편의 이니셜라이저를 되살리면 뷰가 세션을 만들고 소유하게 되어 §7.3.1의 이점이 사라진다. 세션을 **누가 만들고 누가 쥐는가**가 이 챕터의 숨은 전제다.

---

## § 7.3 새 구현이 주는 것

> **(p.146)** *"코스 모델 하나를 동기화하려고 `CourseSession`을 도입하는 것이 과해 보일 수 있습니다. 그러나 이 '단순한' 아이디어의 이점을 보여 드리기 위해, `Course` 기능을 다양한 방식으로 **더 많이** 사용하는 법을 살펴봅시다."*

### § 7.3.1 화면에서 기능을 분리한다

> **(p.146~147)** *"사용자는 `Course`를 로드하고 갱신한 뒤 화면을 떠날 수 있고, 백엔드 서버와의 동기화는 백그라운드에서 계속됩니다. (...) 사용자가 같은 `Course`로 `CourseView` 화면을 다시 열 수 있다는 것입니다. 본질적으로 앱이 `CourseSession`을 `CourseView`에 다시 붙이는 것이고, 사용자는 즉시 최신 상태를 봅니다."*
>
> *"더 중요한 것은, 이제 기능이 항상 백그라운드에서 작동한다는 것입니다. 기능은 UI를 **필요로 하지** 않습니다. 이는 우리가 기능을 생각하는 방식에 영향을 줍니다."*

Figure 7.5에서는 `App`이 `CourseSession`과 Notification Center를 초기화하고, 사용자가 화면을 떠난 뒤 `CourseSession`이 로컬 알림을 트리거해 메인 화면에 "Course was updated successfully!" 토스트를 띄운다. 책은 Note에서 여러 코스를 동시에 갱신한다면 최신 갱신만 보여 주거나 묶어서 보여 주는 등 다른 해법이 필요하다고 덧붙인다.

> **읽을 때 주의할 지점: 분리는 소유자가 바깥에 있을 때만 성립한다.** Figure 7.5에서 `CourseSession`을 초기화하는 것은 `App`이다. Listing 7.2의 `CourseView`는 세션을 `@Bindable`로 들고 있을 뿐 소유하지 않으므로(`@State`가 아니다) 세션의 수명은 그것을 만든 쪽이 정한다. 부모 뷰가 그 자리에서 `CourseSession(...)`을 만든다면 부모가 사라질 때 세션도 사라진다. 게다가 §7.3.3의 `deinit { updateTask?.cancel() }` 때문에 디바운스 대기 중이던 마지막 변경이 **취소되어 서버에 가지 않는다.** 이 이점은 세션을 화면보다 오래 사는 무언가(앱, 세션 저장소, 내비게이션 계층)가 들고 있을 때에만 성립하며 책은 그 호출부를 보여 주지 않는다.

### § 7.3.2 다른 플랫폼과 패턴을 지원한다

> **(p.148)** *"예를 들어 SwiftUI의 어떤 단점 때문에 팀이 UIKit으로 돌아간다고 해 봅시다(실제로 일어나는 일입니다). (...) 이제는 그 모든 것을 `CourseSession`에서 공짜로 얻습니다. 공짜로 얻지 못하는 유일한 것은 `CourseSession`을 UIKit에 접착하는 일입니다."*

§7.0의 "완전히 이식 가능하게 만들 수는 없다"는 단서가 여기서 구체화된다. 다시 구현할 것이 접착뿐이고 그 분량이 Listing 7.2 정도라면, 플랫폼을 옮기는 비용은 기능 전체가 아니라 뷰 하나의 `switch` 문이 된다.

### § 7.3.3 동기화 로직을 성숙시킨다

> **(p.148)** *"예를 들어 디바운싱을 지원할 수 있습니다. 사용자가 할 일 항목을 반복해서 탭하면, 서버에 부하를 주지 않도록 네트워크 호출 전에 작은 지연을 둡니다. 그 위에 취소도 지원할 수 있습니다."*

```swift
@Observable
final class CourseSession {
    // ... snip

    // We store a Task
    private var updateTask: Task<Void, Never>?
    // We define a delay
    private let updateDelayInSeconds: UInt64 = 1

    var state: LoadState = .loading {
        didSet {
            // On state changes we cancel the running task (if any)
            updateTask?.cancel()
            guard case let .loaded(newCourse) = state,
                  case let .loaded(oldCourse) = oldValue,
                  oldCourse.id == newCourse.id,
                  let courseService else { return }
            // We store the new task
            updateTask = Task { [courseService] in
                do {
                    // We only make the network call after some inactivity
                    try await Task.sleep(nanoseconds: updateDelayInSeconds * 1000_000_000)
                    guard !Task.isCancelled else { return }
                    try await courseService.update(course: newCourse)
                } catch is CancellationError {
                    return
                } catch {
                    return
                }
            }
        }
    }

    // When removed from memory, we cancel a running task
    deinit {
        updateTask?.cancel()
    }

    // ... snip
}
```
**Listing 7.3** 디바운싱과 취소를 지원하도록 CourseSession 갱신 (p.148~149)

- **디바운스는 "마지막 것만 보낸다"(trailing) 방식이다.** 1초 안에 다시 변경이 오면 대기 중이던 `Task`가 취소되므로 연속 탭은 마지막 변경 하나만 전송된다. `Course` 전체를 보내므로 중간 변경이 유실되지 않는다.
- **취소는 상태가 바뀔 때마다 일어난다.** `cancel()`이 `guard`보다 앞에 있으므로 `loading`이나 `failed`로의 전이도 대기 중인 갱신을 취소한다. 토글 직후 1초 안에 `startLoadCourse()`가 다시 불리면 토글 결과가 전송되지 않고 서버 값으로 덮어써진다. 책은 이것이 의도인지 언급하지 않지만 Listing 7.2는 `loading` 상태에서만 로드하므로 일반적인 화면 흐름에서는 생기지 않는다.
- **`Task`가 `self`를 붙잡지 않는다.** 캡처 목록이 `[courseService]`이므로 세션이 해제될 수 있고 해제되면 `deinit`이 대기 중인 `Task`를 취소한다. §7.3.1의 "소유자가 없으면 마지막 변경이 유실된다"는 이 두 줄의 결과다.
- **오류는 여전히 삼킨다.** 달라진 것은 오류 처리를 추가할 **자리**다. Ch6에서는 뷰의 바인딩 클로저를 고쳐야 했지만 이제는 `didSet` 안에서 끝난다.

| Ch6 §6.6이 열거한 한계 | Ch7 이후의 상태 |
|---|---|
| 디바운싱 없음 | **해결.** 1초 trailing 디바운스 |
| 취소 없음 | **해결.** 새 변경, 상태 전이, 해제 시 취소 |
| 뷰 라이프사이클에 종속 | **조건부 해결.** 세션을 화면 밖의 소유자가 들고 있을 때만 |
| 오류 처리, 재시도 없음 | 해결되지 않음. 해결할 자리만 `CourseSession`으로 옮겨졌다 |

### § 7.3.4 수동 UI 확인 대신 단위 테스트

> **(p.149)** *"비교적 작은 변경 덕분에, 이제 `Course`를 변경하면 백엔드를 동기화하는 호출이 트리거된다는 것을 **단위 테스트**할 수 있습니다. (...) 이제 취소나 디바운싱 로직도 단위 테스트할 수 있습니다. 이 로직이 `CourseView`에 있었다면 거의 불가능했을 것입니다."*

| 테스트 대상 | Ch6 | Ch7 |
|---|---|---|
| 로딩 후 기능이 `loaded` 또는 `failed`에 있다 | 불가능 | 가능. `startLoadCourse()`를 부르고 `state`를 확인한다 |
| 변경이 일어나면 `update(course:)`가 호출된다 | 불가능 | 가능. 가짜 서비스의 호출 기록을 확인한다 |
| 연속 변경은 마지막 것 하나만 보낸다 | 불가능 | 가능 (Listing 7.3) |
| 같은 상태로 갱신하면 아무것도 하지 않는다 | 불가능 | 테스트는 쓸 수 있지만 현재 코드로는 실패한다 |

---

## § 7.4~7.5 결론과 요약

> **(p.150)** *"기능이 UI 없이 대부분의 일을 하도록 보장하는 한, UI 도메인에서는 접착만 신경 쓰면 됩니다. 하지만 코드를 UI에 **어떻게** 접착하는지는 그다지 중요하지 않습니다. 구현마다 다릅니다."*
>
> *"또한 UI에 코드를 더 많이 두는 편이 보통 더 빨리 가동됩니다. 하지만 이 챕터에서 다루었듯이, 비즈니스 로직을 어디에 둘지 의식하는 데는 **수많은** 이점이 있습니다."*
>
> ***"다음에 UI 도메인에서 `loadMyFeature()` 구현을 보게 되면 자세히 들여다보세요. 함수 본문 전체가 모델 도메인으로 옮겨져야 한다는 것을 깨달을 수도 있습니다."***

"UI에 두면 더 빨리 가동된다"는 인정은 Ch6을 변호하는 문장이기도 하다. 책은 가벼운 해법으로 먼저 가동하고 다음 챕터에서 그 한계를 해소하는 순서를 일부러 택했다. `loadMyFeature()` 문장은 UI-01의 "명령줄 도구라면?" 사고 실험을 코드 리뷰에서 바로 쓸 수 있는 **탐지 규칙**으로 바꾼 것이다.

**요약 (p.151)**

- 이식 가능 기능은 특정 UI 패턴, 아키텍처, 패러다임에 (너무 많이) 기대지 않고 타깃 전반에서 작동한다.
- **뷰가 `loadData` 호출을 처리한다면, 데이터와 상태의 진실 공급원이 UI 도메인에 산다는 지표일 수 있다.**
- 로직을 모델 도메인으로 옮기면 모델 도메인이 진실 공급원이 되고, 뷰는 데이터와 상태를 UI에 접착하는 데 집중한다.
- 기능이 UI 없이 작동하면 단위 테스트, 다양한 플랫폼, 다른 UI 패턴으로의 이동, 백그라운드 실행이 쉬워진다.
- 이식 가능 기능은 화면에서 분리되어 사용자가 화면을 떠난 뒤에도 작동할 수 있다. 그러면 화면 밖에서 사용자에게 알리는 방법(알림, 토스트)을 생각해야 한다.
- 뷰에서 로직을 빼내면 취소나 디바운싱 같은 정교한 로직을 더 쉽게 추가할 수 있다.

### Ch6에서 Ch7로: 무엇이 어디로 옮겨졌는가

| 책임 | Ch6 (§6.6) | Ch7 (§7.3.3) |
|---|---|---|
| `LoadState` 정의와 현재 `Course` 보관 | `CourseView`의 `@State` | `CourseSession.state` |
| 로드 시작 시점 결정 | `CourseView` (`.task`) | `CourseView` (`.task`가 `startLoadCourse()` 호출) |
| 로드 실행과 상태 전이 | `CourseView` | `CourseSession` |
| 변경 수용 판단과 백엔드 갱신 | `CourseView`의 바인딩 `set` | `CourseSession`의 `didSet` |
| 디바운싱과 취소 | 없음 | `CourseSession` |
| 상태에 따라 보일 뷰 선택 | `CourseView` | `CourseView` |
| 콘텐츠 렌더링 | `CourseUI` | `CourseUI` (수정 없음) |

로드 시작 시점도 `CourseSession`이 초기화 직후 스스로 처리할 수 있으므로, 원리적으로 `CourseView`에 반드시 남아야 하는 것은 보일 뷰를 고르는 일 하나다. 이것이 책이 말한 "얇은 UI 베니어"의 실제 두께다. 그리고 `CourseView` 위에는 책이 Figure 7.5에서 `App`으로만 표시한 **"세션을 쥐는 바깥 계층"**이 새로 생겼다.

---

# Part B. Ch8 복잡한 UI 길들이기: 복잡성은 없앨 수 없으니 어디에 둘지 고른다

## § 8.0 복잡한 UI란 무엇인가

> **(p.153)** *"이 챕터에서 복잡한 UI라고 말할 때는 **책임이 과중해진 뷰**를 가리킵니다. (...) 복잡한 뷰는 종종 여러 서브뷰를 연결하고 그것들이 모두 작동하도록 보장하는 '거대한 접착제 덩어리(big ball of glue)' 역할을 합니다."*

이 정의에서 복잡성은 **화면의 시각적 복잡성이 아니다.** 화면에 요소가 많은 것이 아니라 뷰 하나가 떠안은 책임이 많은 것이 문제다. 이 구분은 §8.4에서 결정적으로 쓰인다. 시각적으로 가장 단순한 리스트가 책임으로는 가장 복잡할 수 있기 때문이다.

| 사례 | 화면 | 복잡성의 출처 | 순진한 구조 | 부모에게 남는 일 |
|---|---|---|---|---|
| §8.1 | `PaginationView` | 같은 기능 여러 개 | `CourseView` 안에 스크롤 메뉴를 넣거나, 부모가 모든 페이지를 돌본다 | 스크롤, 페이지 추적, 스크롤 메뉴 |
| §8.2 | `DashboardView` | 서로 다른 기능 일곱 개 | 부모 하나가 모든 로딩, 오류, 이벤트를 처리한다 | 그리드 레이아웃 |
| §8.4 | `SettingsView` | 겉보기에는 단순한 리스트 하나 | 부모가 모든 행의 의존성과 로직을 갖는다 | 행을 모아 하나의 리스트로 그리기 |

세 해법의 공통점은 **부모를 레이아웃 전담으로 만들고 나머지 복잡성을 아래로 밀어낸다**는 것이다.

---

## § 8.1 사례 1: 페이지네이션

### § 8.1.1 미루는 것이 개발에 좋은 영향을 줄 수 있다

`CourseView` 디자인의 맨 위에는 처음부터 여러 코스를 오가는 수평 **스크롤 메뉴**가 있었지만, 책은 출시를 앞당기기 위해 이를 미뤘고 "학생은 한 번에 하나만 배울 것"이라고 가정했다. 출시 후 UX 리서치는 많은 사람이 여러 코스를 동시에 수강하고 싶어 한다는 것을 보여 주었다. 책의 표현으로는 *"우리의 가정은 틀렸습니다."*

책은 미루기의 이점을 **속도**와 **설계의 질**로 나누고 후자를 강조한다. 디자인 그림만 보면 스크롤 메뉴는 `CourseView`의 일부처럼 보인다. 처음부터 메뉴를 함께 구현했다면 `CourseView`는 "코스 하나를 보여 주는 기능"이 아니라 "여러 코스를 오가는 화면"이 되었을 것이고, 다른 곳에 임베드하거나 `id` 하나로 딥링크하는 일(Ch6 §6.4)이 훨씬 어려워졌을 것이다. 미루는 동안 `CourseView`가 자기 충족적이 되었으므로 다중 코스는 `CourseView`를 바꾸는 작업이 아니라 그 위에 부모를 하나 추가하는 작업이 되었다.

### § 8.1.2~8.1.3 기술적 설계와 자기 충족 기능의 보상

> **(p.157)** *"스크롤 메뉴를 가지고 여러 `CourseView` 인스턴스를 나란히 놓는 새 부모 `PaginationView`를 도입하겠습니다. 이는 `PaginationView`에 `CourseView` 인스턴스를 최대 세 개까지 놓는다는 뜻입니다. (...) 사용자가 스크롤하는 동안 `PaginationView`가 필요에 따라 이 뷰들을 재사용하거나 초기화할 수 있기 때문입니다."*

```
                PaginationView
              ┌──────────────────┐
              │ Course  Course  Cour│  ← 스크롤 메뉴 (PaginationView 소속)
┌─────────────┼──────────────────┼─────────────┐
│  Previous   │     Current      │    Next     │  ← ScrollView
│ CourseView  │    CourseView    │ CourseView  │
└─────────────┴──────────────────┴─────────────┘
```

스크롤 메뉴가 `CourseView` 밖으로 나온 것이 이 설계의 핵심이다. 메뉴는 "지금 몇 번째 코스인가"를 다루므로 여러 코스를 알아야 하고 `CourseView`는 코스 하나만 안다. 메뉴를 `CourseView` 안에 두면 자기 충족 원칙 3(부모나 조상과 독립적으로 작동한다)이 깨진다.

> **(p.158)** *"각 `CourseView`가 스스로 로드할 수 있습니다. 각 `CourseView`가 자신의 오류를 스스로 처리할 수 있습니다. (...) 이렇게 하면 알림으로 UI를 막지 않습니다. 어느 `CourseView`도 내비게이션을 방해하지 않습니다."*
>
> *"`CourseView`를 자기 충족적으로 만들지 않았다면 (...) 아마도 이 모든 `CourseView` 인스턴스를 **돌보는(babysit)** 접착 코드가 잔뜩 든 맞춤형 `CoursePaginationView`로 끝났을 것입니다."*

세 항목은 자기 충족 4원칙의 원칙 1(자기 로딩), 원칙 2(자체 오류 처리), 원칙 4(내비게이션 비책임)에 하나씩 대응한다. 특히 두 번째 항목은 Ch5에서 "다중 동시 오류"를 다룬 이유를 보여 준다. 세 페이지가 동시에 로드하다 둘이 실패할 때 각자 알림을 띄우면 화면이 막히지만 인라인 오류는 자기 자리에서 실패를 보여 줄 뿐이다.

가상의 이름 `CoursePaginationView`에도 주목할 만하다. 자기 충족적이지 않은 자식을 다루는 부모는 자식의 도메인을 알아야 하므로 이름부터 `Course`에 묶인다. 자기 충족적인 자식을 다루는 부모는 `PaginationView`라는 범용 이름을 가질 수 있다. **자식의 자기 충족성이 부모의 재사용성을 결정한다.**

책은 페이지를 채우는 방법(데이터 배열을 넘기거나 `makeCourseView(course:)` 팩토리를 넘기는 방법)을 플랫폼에 특화된 구현 세부로 열어 두고, 네이티브 페이지네이션 컴포넌트를 써도 된다고 Note로 덧붙인다. 팩토리는 Ch5 §5.5의 "미리 채워진 팩토리"와 같은 생각이다. 내비게이션에 대해서는 *"'Reusing views across flows' 챕터에서 다루었듯이 내비게이션은 문제가 되지 않습니다"*라고만 말한다.

> **읽을 때 주의할 지점**: 'Reusing views across flows'는 이 책의 **다음 챕터(Ch9)**다. 챕터 순서가 집필 중에 바뀐 흔적으로 보인다. 지금은 "내비게이션도 다른 의존성처럼 주입하면 부모가 몰라도 된다"는 결론만 받아 두고 방법은 Ch9에서 확인하면 된다.

---

## § 8.2 사례 2: Bento-Box UI

### 대시보드 디자인

강사가 로그인하면 답하지 않은 채팅, 최근 활동, 다가오는 결제, 분석, 학생 인사이트, 코스 진행 상황 등을 한눈에 보는 개요 화면을 받는다. 책은 이 뷰를 `TutorDashboardView`가 아니라 `DashboardView`라고 부른다. UI-02 정리 문서의 원칙 5(본질로 이름 짓기)를 따른 것이다.

> **(p.160~161)** *"각각의 블록, 즉 기능이 폰에서는 하나의 화면이 될 수 있다고 말할 수 있습니다. (...)* ***이는 기능이 항상 화면은 아니라는 개념을 강화합니다.*** *(...) 이런 종류의 UI를 **Bento-Box UI**라고 부르겠습니다."*

"기능이 항상 화면은 아니다"는 UI-03 정리 문서의 Note(p.58)에서 처음 나온 말이다. 같은 기능 블록이 폰에서는 화면 전체가 되고 태블릿에서는 그리드의 한 칸이 된다. 기능을 "화면"으로 설계했다면 태블릿에서 임베드할 수 없고 "기능 뷰"로 설계했다면 두 곳 모두에 놓을 수 있다. 여기에 모든 기능이 네트워크 데이터를 필요로 하고, 일부는 느리거나 실패하며, 일부는 권한이 필요하다는 조건이 더해진다. 위험은 `DashboardView`가 비동기 호출, 오류 관리, 긴 의존성 목록이 얽힌 그물이 되며 **거대해지는** 것이다.

### § 8.2.1~8.2.2 평평한 계층과 "어두울수록 나쁘다"

순진한 접근은 모든 로직, 로딩, 상태, 오류를 처리하는 거대한 뷰 하나다. 책은 디자인을 등각 투영 3D 그림으로 바꿔 깊이가 두 층(`DashboardView`와 일곱 서브뷰)뿐인 **평평한 계층**임을 보이고, 이 평평함이 복잡해지는 주된 이유라고 말한다. 그리고 책임이 많을수록 어둡게 칠하는 명암 표기를 도입한다. 가장 어두우면 **위험(critical)**, 꽤 어두우면 **경고(warning)**다. 책의 표현으로는 *"어두운 뷰는 나쁩니다."*

> **(p.164)** *"`DashboardView`를 꽤 복잡하게 만듭니다. (...) 정확히는 기능 일곱 개 분량입니다. **그런데 이는 서브뷰가 단순하다면 복잡성이 다른 어딘가로 가야 한다는 뜻입니다.** 여기서는 `DashboardView`로 옮겨졌습니다."*
>
> *"`DashboardView`는 긴 속성 목록과 다양한 상태를 가진, **그물 한가운데의 거미**가 됩니다."*

서브뷰가 "멍청하므로" 버튼 탭 같은 이벤트도 모두 부모로 올라온다. UI-02 정리 문서의 원칙 7("똑똑한 뷰보다 컴포지션을 선호하라")과 이 절은 충돌하지 않는다. 원칙 7의 대상은 뷰 프리미티브이고 이 절의 대상은 기능 뷰다. **기능을 멍청하게 만들면 그 기능의 로직이 사라지는 것이 아니라 부모로 올라간다.**

### § 8.2.3~8.2.4 책임을 아래로 밀어내면 기능이 분리된다

> **(p.164)** *"부모라는 사실이 모든 무거운 일을 해야 한다는 뜻은 아닙니다. 자식들도 자기 뒷정리를 할 수 있다면 좋을 것입니다. (...) `DashboardView`가 무거운 일의 90%를 하는 대신 이제 10%를 하고, 나머지 90%를 각 서브뷰에 나눠 줍니다."*
>
> **(p.165~166)** *"어쩌면 놀랍게도, `DashboardView`가 직속 서브뷰들보다 더 단순합니다. (...) **각 뷰에 자신의 의존성을 주면, 그 뷰가 자기 기능과 관련된 모든 것(로딩, 오류, 사용자 상호작용 등)을 처리합니다.**"*

명암이 뒤집혀 `DashboardView`는 거의 흰색이 되고 서브뷰들은 경고 수준으로 어두워진다. 이 해법에서는 크기와 복잡성이 분리된다. `DashboardView`는 화면에서 가장 큰 뷰이지만 하는 일은 배치뿐이다.

> **(p.166)** *"여러 호출을 한데 묶게 될 가능성이 더 높고, 그러면 각 서브뷰 사이에 **암묵적인 얽힘**이 생깁니다. 한 기능의 데이터 로딩이 실패하면, 모든 것이 서로 연결되어 있으므로 다른 뷰도 제대로 갱신되지 않을 가능성이 높다는 뜻입니다."*

부모가 일곱 API를 모두 호출하면 코드를 쓰는 사람은 자연스럽게 그것들을 한 번에 기다리거나 하나의 로딩 상태, 하나의 오류 블록으로 묶는다. 그 순간 결제 API 하나가 실패해도 대시보드 전체가 오류 화면이 된다. 이 결합은 코드 어디에도 명시되지 않으므로 "암묵적"이다. 기능마다 자신의 로딩을 가지면 이런 묶음이 구조적으로 생기지 않는다.

### § 8.2.5 더 깊은 계층

서브뷰들이 여전히 경고 상태이므로 책은 같은 기법을 한 번 더 적용한다. Ch5에서 `CourseView`에서 `CourseUI`를 추출했듯이 각 기능 뷰에서 단순한 UI를 추출한다. Unanswered Chats는 `ChatsUI`, Payment Overview는 `TransactionsView`, Analytics는 `GraphView`, Course Progress는 `ProgressView`에 의존하는 식이다.

> **(p.168)** *"기능들(가장 어두운 가운데 층)은 로딩과 오류 처리 같은 책임 대부분을 떠맡지만, **오직** 각자의 기능에 대해서만입니다. (...) 가운데 층이 그중 가장 복잡하다는 것을 볼 수 있습니다. 이것은 의도된 것이며, 이제 경고 상태나 위험 상태가 아닙니다."*

| 단계 | `DashboardView` | 기능 (서브뷰) | 단순 UI 뷰 | 깊이 |
|---|---|---|---|---|
| §8.2.2 순진한 구조 | 새까맣다 (**위험**) | 밝다 | 없다 | 2 |
| §8.2.3 책임을 아래로 | 밝다 | 어둡다 (**경고**) | 없다 | 2 |
| §8.2.5 UI를 한 층 더 분리 | 밝다 | 가장 어둡지만 경고는 아니다 | 가장 밝다 | 3 |

전체 어두움의 양, 즉 복잡성의 총량은 세 단계에서 거의 같고 바뀐 것은 **분포**다. 가운데 층이 가장 어두운 것이 "의도된 것"이라는 말은, 목표가 복잡성을 0으로 만드는 것이 아니라 **복잡성이 있어야 할 자리에 두는 것**이라는 뜻이다. 이 3계층은 UI-03 정리 문서의 분류와 대응한다. `DashboardView`는 기능 뷰를 담는 레이아웃 컨테이너이고, 가운데 층은 기능 뷰이며, 맨 앞 층은 뷰 컴포넌트 또는 프리미티브다. Ch5의 `CourseView`/`CourseUI` 분리를 기능 일곱 개에 반복하고 그 위에 레이아웃 층을 얹은 것이다.

---

## § 8.3 접근 평가

> **(p.169)** *"단점은 타입과 추상화 계층을 더 많이 도입했다는 것입니다. 그것이 트레이드오프입니다. 모든 것을 하고 버그가 생기기 쉬운 거대한 **신(god) 타입**보다, **분명한 역할**을 가진 더 작고 중첩된 타입에 대해 추론하는 것이 더 쉽습니다."*
>
> ***"현실은 복잡성이 존재한다는 것입니다. 기능을 일부 없애지 않는 한 그것을 없앨 수는 없습니다. 그래서 질문은 이렇게 됩니다. 복잡성이 어디에 살기를 원하는가?"***

이 문장은 리팩터링의 목표를 다시 정의한다. 리팩터링은 복잡성을 줄이는 작업이 아니라 **재배치**하는 작업이며 재배치의 기준은 "사람이 한 번에 머릿속에 담을 수 있는 크기"다.

### § 8.3.1 버그가 생겼을 때

Activities 기능의 데이터가 제대로 로드되지 않으면, 예전에는 거대한 `DashboardView`를 "마체테를 들고" 헤쳐야 했지만 이제는 `Activities` 클래스로 바로 간다. 그리드가 잘리거나 간격이 맞지 않으면 레이아웃만 맡은 `DashboardView`를 본다. 계층이 생기면 **버그의 종류가 버그의 위치를 알려 준다.**

책은 이 추상화가 "누가 필요할지 모르니 재사용 가능하게 만들자"는 추상화가 아니라 **복잡성을 다루기 위한 것**이라고 덧붙인다. 이 말이 UI-02 정리 문서 §2.5.6("가설적 시나리오를 위해 만들지 마라")과의 경계를 긋는다. 가설적 재사용을 위한 추상화는 **존재하지 않는 미래의 요구**에 대응하고 이 챕터의 추상화는 **지금 존재하는 복잡성**에 대응한다. 판별 기준은 "이 타입이 없으면 지금 어느 클래스가 무거워지는가"를 답할 수 있는지 여부다.

### § 8.3.2 복잡한 화면 문제를 더 단순한 의존성 문제로 바꾼다

> **(p.170)** *"최신 해법에서는 배치할 뷰 목록만 있으면 되고, `DashboardView`는 그것으로 그리드를 만들 만큼 똑똑할 것입니다. 이 뷰들은 아마 (지연해서) 전달되고 필요할 때 만들어질 것입니다. (...) 이는 `DashboardView`가 기능들의 의존성을 알 필요가 없다는 뜻입니다."*
>
> *"이제 이 뷰들 중 어느 것도 그 자체로는 복잡하지 않습니다. **이제 복잡성은 시스템 전체, 그리고 데이터와 의존성을 어떻게 전달하는가에 있습니다.**"*

복잡성은 사라지지 않고 **뷰들 사이의 연결**로 옮겨졌다. 이 연결은 `.docs/08-Sane-DI-정리.md`가 다룬 **ABC problem**과 같은 모양이다. `DashboardView`(A)가 기능(B)을 만들려면 `ChatAPI` 같은 의존성(C)을 알아야 한다면 A는 쓰지도 않는 의존성의 통로가 된다. Sane DI의 해법은 바깥 가장자리(`AppSetup`)에서 B를 C로 미리 채워 두고 A에게는 완성된 B나 B를 만드는 팩토리만 건네는 것이었다. "지연해서 전달된다"는 같은 문서 §8.9의 지연 의존성에 해당한다.

### § 8.3.3~8.3.4 기능끼리 협력해야 할 때

기능을 분리했으므로 기능들이 함께 작동해야 할 때 문제가 생긴다. 책은 두 경우를 구분한다.

> **(p.171)** *"드래그는 UI 기반이므로 이것을 UI 계층(예: `DashboardView`)에 두는 것이 합리적입니다. 그러면 `DashboardView`가 다시 복잡성을 얻게 되는데, 이 기능 뷰들 사이의 드래그를 제공하는 것은 `DashboardView`의 본성의 일부이므로 받아들일 만합니다."*
>
> *"데이터 모델이 갱신되면 여러 기능 뷰도 갱신되어야 한다고 상상해 봅시다. 이것을 `DashboardView`에서 접착할 수도 있지만, 그러면 그 코드가 UI 계층에 있게 되어 기능들의 이식성이 떨어집니다. 대안으로 pub/sub 시스템이나 반응형 시스템 같은 모델을 제공해서, 여러 리스너가 모델 변경을 구독하게 할 수 있습니다."*

| 협력의 종류 | 예 | 둘 곳 | 이유 |
|---|---|---|---|
| UI 기반 상호작용 | 기능 사이의 드래그 앤 드롭, 강조 표시 | UI 계층 (`DashboardView`) | 화면 위치와 제스처에 관한 일이고 공통 부모의 본성에 속한다 |
| 데이터 동기화 | 한 모델의 변경을 여러 기능이 반영 | 모델 계층 (pub/sub, 반응형 스트림) | 모델에 두면 이식성이 유지되고 단위 테스트할 수 있다 |

§8.2.4에서 부정적이던 "우산 뷰"라는 말이 여기서는 긍정적으로 쓰인다. 데이터 로딩은 각 기능의 일이므로 우산 아래에 두면 안 되고 기능 **사이의** 드래그는 어느 한 기능의 일이 아니므로 공통 부모가 맡는다. 판별 기준은 Part A와 같다. "UI가 없어도 이 협력이 필요한가?" 드래그는 UI가 없으면 존재하지 않고 데이터 동기화는 명령줄 도구에서도 필요하다. 책도 이 대목에서 Ch7을 직접 가리킨다.

---

## § 8.4 사례 3: 단순한 UI, 복잡한 기능

### 리스트는 속인다

> **(p.172)** *"리스트는 기본적으로 사용자에게 가장 단순한 표현입니다. 하지만 문제는 기능을 밀어 넣는 순간, 이 리스트 개요가 각각의 '단순한' 리스트 요소에서 온 로직으로 가득 찬다는 것입니다. 한 리스트 요소는 위치 권한을 켜는 토글을 가질 수 있습니다. 그러면 리스트 개요가 위치 설정을 의존성으로 받아야 할 수 있습니다."*

강사용 설정 리스트(Profile, Security, Payments, Privacy, Integrations 등)는 `DashboardView`와 달리 기능마다 상자가 없다. 모든 행이 같은 모양으로 한 리스트에 이어져 있어서 기능의 경계가 화면에 드러나지 않는다.

```swift
struct SettingsView: View {
    // Profile
    let user: User
    let profilePreferences: ProfilePreferences
    // Security
    let securitySettings: SecuritySettings
    // Payments
    let paymentsAPI: PaymentsAPI
    let paymentsSettingsAPI: PaymentsSettingsAPI
    // Privacy
    let privacyAPI: PrivacyAPI
    let secureStore: SecureStore
    // Integrations
    let videoIntegrations: VideoIntegrations
    let integrationsAPI: IntegrationsAPI
    let insightsAPI: InsightsAPI
    // Discussion Board
    let communityAPI: CommunityAPI
    // .. rest omitted
}
```
**Listing 8.1** 다양한 속성을 많이 가진 SettingsView (p.174)

속성 열한 개가 주석으로 여섯 묶음으로 나뉘어 있다. 개발자가 속성을 주석으로 묶었다는 것은 머릿속에 이미 경계가 있다는 뜻이며, 그 경계가 아직 **타입**으로 표현되지 않았을 뿐이다.

### § 8.4.3 첫 번째 해법: 서브 화면으로 자른다

책이 선호하는 해법은 메인 설정 화면을 프로필, 보안, 결제 등 서브 화면으로 나누는 트리 구조다. *"다시 한번 '복잡한 UI' 문제를 의존성 문제로 바꾸는 것입니다."* UI의 계층과 코드의 계층이 일치하므로 가장 단순하고, 책은 거대한 리스트 하나가 검색이 있더라도 강사에게 좋은 UI가 아니라고 덧붙인다.

### § 8.4.4 두 번째 해법: UI를 기능에서 분리한다

> **(p.176)** *"CEO가 거대한 리스트 하나가 '어쨌든' 더 낫다고 고집해서, 지금은 그것을 구현하고 있습니다. (...)* ***하지만 리스트가 모든 것을 할 수 있다고 해서, 리스트 뷰 타입도 모든 것을 해야 한다고 속지 마세요.***"*
>
> *"여기서 요령은 가능한 곳에 **여전히** 계층을 도입하되, (...) `SecuritySection`, `PaymentsSection`, `PrivacySection`, `IntegrationSection` 같은 섹션을 추출하는 것입니다. (...) 각 섹션을 자기만의 자기 충족 기능, 즉 기능 뷰로 볼 수 있습니다. **하지만 UI를 스스로 렌더링하는 대신, 자신의 UI(행)를 부모에게 넘겨 렌더링하게 합니다.**"*

사용자에게 보이는 구조(리스트 하나)와 코드의 구조(타입 하나)는 같을 필요가 없다. Figure 8.10은 부모 리스트의 빈 슬롯을 각 섹션 타입이 내놓는 행(`SecuritySection`의 2단계 인증 토글, `PaymentsSection`의 지급 설정 등)이 채우는 모습을 그린다.

이 구조는 Ch5의 `CourseView`/`CourseUI` 분리와 닮았지만 **방향이 반대**다.

| 구조 | 부모 | 자식 | UI를 그리는 쪽 | 데이터가 흐르는 방향 |
|---|---|---|---|---|
| Ch5 / §8.2.5 | 기능 (`CourseView`, Unanswered Chats) | UI (`CourseUI`, `ChatsUI`) | 자식 | 아래로 (기능 → UI) |
| §8.4.4 | 리스트 (`SettingsView`) | 섹션 (`SecuritySection` 등) | 부모 | 위로 (섹션이 행을 내놓는다) |

두 구조가 공유하는 것은 **로직과 렌더링을 서로 다른 타입에 둔다**는 원칙이고, 어느 쪽이 부모가 되는지는 디자인이 정한다. 디자인이 기능마다 상자를 주면 기능이 부모가 되고 하나의 균일한 리스트를 요구하면 렌더러가 부모가 된다.

### § 8.4.5~8.4.6 더 적은 의존성, 더 잘게 나누기

```swift
struct SettingsView: View {
    let profileSection: ProfileSection
    let securitySection: SecuritySection
    let paymentSection: PaymentsSection
    let privacySection: PrivacySection
    let integrationSection: IntegrationSection
    let communitySection: CommunitySection
    // .. rest omitted
}
```
**Listing 8.2** SettingsView가 이제 복잡성을 하위 타입에 넘긴다 (p.178)

> **(p.178)** *"메인 `SettingsView`가 그 이상의 세부를 알 필요가 없기 때문입니다. `PaymentsAPI`나 `PrivacyAPI` 같은 **의존성의 의존성**을 알 필요가 없습니다."*

속성은 열한 개에서 여섯 개로 줄었지만 더 중요한 것은 **종류**의 변화다. Listing 8.1의 속성은 API, 저장소, 설정 객체처럼 서로 다른 계층의 것들이 섞여 있었고 Listing 8.2의 속성은 모두 "행을 내놓는 섹션"이다. `SettingsView`가 아는 개념이 하나로 줄었다.

섹션도 너무 복잡해지면 행 하나를 자기 타입으로 추출할 수 있다. 책은 *"실제로는 이것이 필요한 경우가 드물지만"*이라는 단서를 단다. 행 단위 타입은 행이 실제로 복잡해졌을 때 도입하는 것이지 대칭을 위해 모든 행에 미리 만드는 것이 아니며, 이는 §8.3.1의 원칙을 다시 확인한다.

---

## § 8.5~8.6 결론과 요약

책은 세 가지 교훈을 정리한다.

1. **기능과 화면은 같지 않다.** *"같을 **수는** 있지만 항상 그렇지는 않습니다."* 기능 뷰는 전체 화면일 수도, 임베드될 수도 있다.
2. **임베드하려면 자기 충족적이어야 한다.** *"기능 뷰가 자기 충족적일수록 배치하거나, 임베드하거나, 더 큰 무언가의 일부로 만들기가 더 쉽습니다."*
3. **계층이 복잡성을 다룰 수 있게 한다.** 복잡성은 기능을 삭제하지 않는 한 사라지지 않지만, 타입이 늘어나는 비용을 치르면 다룰 만해진다. UI가 허락하지 않을 때도 섹션에 계층을 도입하고 각 섹션이 부모에게 행을 건네게 할 수 있다.

**요약 (p.180)**

- 결정적이지 않은 요구사항을 미루면 더 많고 더 나은 정보로 해법을 설계할 수 있다.
- 기능이 항상 전체 화면은 아니다. 이 생각을 떼어 내면 기능을 임베드할 준비가 되고 더 복잡한 화면이 가능해진다.
- 복잡한 디자인에서 부모를 가장 똑똑하게 만들면 서브뷰가 **너무** 단순해지고, 부모가 유지보수하기 어려워지며, 서브뷰 사이에 암묵적 결합이 생긴다.
- 대신 복잡성을 서브뷰나 하위 기능으로 **밀어내라.** 부모는 레이아웃에 집중하고, 하위 기능이 여전히 복잡하면 더 작은 타입으로 나눈다.
- 큰 리스트는 섹션을 자기 타입으로 추출해 행 채우기와 데이터 로딩을 맡기고, 렌더링은 부모에 넘겨 균일한 전체를 만든다. 대안으로 큰 리스트를 작은 리스트들로 자를 수 있다.

### 세 사례를 한 표로: 복잡성은 어디로 옮겨졌는가

| | §8.1 페이지네이션 | §8.2 대시보드 | §8.4 설정 리스트 |
|---|---|---|---|
| 자식 | 같은 기능의 인스턴스 (`CourseView` ×3) | 서로 다른 기능 일곱 개 | 섹션 여섯 개 |
| 시각적 경계 | 있다 (페이지) | 있다 (그리드 칸) | **없다** (한 리스트) |
| 자식이 하는 일 | 로드, 오류, 렌더링 전부 | 로드와 오류. 렌더링은 손자(UI 뷰)에게 | 로드, 상태, 행 생성. 렌더링은 **부모**에게 |
| 부모에게 남는 일 | 스크롤, 페이지 추적, 스크롤 메뉴 | 그리드 레이아웃, 기능 사이 드래그 | 행을 모아 리스트로 그리기 |
| 부모가 받는 의존성 | `CourseView` 팩토리 또는 데이터 배열 | 뷰 목록 또는 팩토리 | 섹션 여섯 개 |

세 사례에서 부모가 받는 의존성은 모두 "완성된 자식 또는 자식을 만드는 방법"이다. 부모는 자식이 무엇에 의존하는지 모른다. 이것이 §8.3.2의 "의존성 문제로 바꿨다"는 말의 구체적인 결과다.

---

# 안드로이드로 옮기면

Ch7의 문제는 UI-05·06 정리 문서에서 옮긴 안드로이드 코드에 이미 들어 있다. 그 `CourseViewModel`은 `uiState`를 직접 보관하고 `toggleTodo`로 변경했는데, `ViewModel`은 UI 계층에 속하므로 그 안에 진실 공급원을 두면 Ch6의 `CourseView`와 같은 한계를 갖는다. Ch7을 옮기는 작업은 진실 공급원을 `ViewModel` **아래로** 내리는 작업이다. Ch8은 코드가 거의 없는 설계 챕터이고 Compose가 이미 `HorizontalPager`, 지연 그리드, `LazyColumn`을 제공하므로 부모를 직접 만들 일은 적다. 대신 **`ViewModel`의 범위와 내비게이션 인자가 "기능 = 화면"을 전제로 설계되어 있다**는 함정이 있다.

## 책의 요소와 안드로이드의 대응

| 책 (Swift) | 안드로이드 (Kotlin) | 비고 |
|---|---|---|
| 모델 도메인 | `android.*`에 의존하지 않는 순수 Kotlin 모듈 | |
| `CourseSession` | 평범한 Kotlin 클래스 (`ViewModel`이 **아니다**) | 책의 "ViewModel이라고 부르지 않는다"를 따른다 |
| `@Observable` + `didSet` | `StateFlow` + 변경 입구 메서드 `update(course)` | |
| `Task` + `Task.sleep`, `deinit` | `Job` + `delay`, 소유자가 부르는 `close()` | Kotlin에는 소멸자가 없다 |
| Figure 7.5의 `App` | Hilt `@Singleton` 세션 저장소 + 앱 수준 `CoroutineScope` | 세션을 화면보다 오래 살게 하는 소유자 |
| `CourseView` (`@Bindable`) | 얇은 `ViewModel` + `CourseRoute` | `ViewModel`은 세션을 찾아 노출만 한다 |
| `PaginationView`, 최대 세 개 | `HorizontalPager` + `beyondViewportPageCount = 1` | 스크롤 메뉴(`ScrollableTabRow`)는 페이저 쪽에 둔다 |
| `DashboardView` (그리드) | `LazyVerticalGrid` 또는 `LazyVerticalStaggeredGrid` | |
| 뷰 목록 또는 팩토리 | Hilt 멀티바인딩으로 모은 타일 목록 | |
| pub/sub 모델 | 저장소가 노출하는 `Flow` | 모델 계층에 둔다 |
| 섹션이 행을 부모에게 넘긴다 | `LazyListScope` 확장 함수 + 섹션별 상태 소유자 | |

## CourseSession (Ch7)

```kotlin
// :feature:course:domain. 순수 Kotlin 모듈이며 android.*에 의존하지 않는다.
class CourseSession(
    private val id: String,
    private val courseService: CourseService,
    private val scope: CoroutineScope,              // 화면보다 오래 사는 스코프를 바깥에서 받는다
    private val updateDelay: Duration = 1.seconds,  // 책의 updateDelayInSeconds. 테스트에서 바꿀 수 있다
) {
    sealed interface LoadState {
        data object Loading : LoadState
        data class Loaded(val course: Course) : LoadState
        data class Failed(val error: Throwable) : LoadState
    }

    private val _state = MutableStateFlow<LoadState>(LoadState.Loading)
    val state: StateFlow<LoadState> = _state.asStateFlow()

    private var updateJob: Job? = null

    suspend fun startLoadCourse() {
        _state.value = LoadState.Loading
        _state.value = try {
            LoadState.Loaded(courseService.loadCourse(id))
        } catch (e: CancellationException) {
            throw e
        } catch (e: Exception) {
            LoadState.Failed(e)
        }
    }

    // 책의 didSet 자리. 변경이 들어오는 입구를 메서드 하나로 드러낸다.
    fun update(course: Course) {
        val current = (_state.value as? LoadState.Loaded)?.course ?: return
        if (current.id != course.id || current == course) return  // 책의 guard + "같은 상태면 아무것도 하지 않는다"
        _state.value = LoadState.Loaded(course)

        updateJob?.cancel()                                         // 책의 updateTask?.cancel()
        updateJob = scope.launch {
            delay(updateDelay)
            try {
                courseService.update(course)
            } catch (e: CancellationException) {
                throw e
            } catch (e: Exception) {
                // 책과 같이 아직은 삼킨다. 재시도와 실패 알림을 추가할 자리다.
            }
        }
    }

    // 책의 deinit. 소유자가 세션을 버릴 때 부른다.
    fun close() {
        updateJob?.cancel()
    }
}
```

책의 코드와 다른 점이 세 가지 있으며 모두 의도한 차이다. 첫째, `state`를 읽기 전용으로 노출하고 변경을 `update(course)` 하나로만 받으므로 `didSet`이 주던 "어떤 경로로 바뀌든 같은 규칙"을 다른 방식으로 얻는다. 둘째, `update` 안에서만 취소하므로 재로드가 대기 중인 갱신을 지우지 않는다. 셋째, 같은 값이면 아무것도 하지 않는다.

## 세션의 소유자와 얇은 ViewModel (Ch7 + Ch8)

§7.3.1의 이점은 세션을 화면 밖에서 소유할 때만 성립하므로, 소유자를 `@Singleton` 저장소로 만든다.

```kotlin
// 책의 Figure 7.5에서 App이 CourseSession을 초기화하는 자리
@Singleton
class CourseSessions @Inject constructor(
    private val courseService: CourseService,
    @ApplicationScope private val appScope: CoroutineScope,  // 프로젝트가 정의한 앱 수준 스코프라고 가정한다
) {
    private val sessions = mutableMapOf<String, CourseSession>()

    fun get(id: String): CourseSession =
        sessions.getOrPut(id) { CourseSession(id, courseService, appScope) }
}
```

`ViewModel`은 세션을 찾아 노출할 뿐 상태를 보관하지 않는다. 여기서 Ch8의 페이지네이션이 함정을 드러낸다. `ViewModel`이 `SavedStateHandle`에서 `id`를 읽으면 `CourseRoute`가 **내비게이션 목적지**라고 가정한 셈이다. 이 `CourseRoute`를 페이저 안에 세 개 놓으면 두 가지가 깨진다. `hiltViewModel()`은 키가 없으면 가장 가까운 `ViewModelStoreOwner`에 클래스별로 하나만 만들므로 세 페이지가 같은 인스턴스를 받는다. 그리고 `SavedStateHandle`에는 페이지마다 다른 `id`가 없다. 그래서 처음부터 기능의 입력을 **매개변수로** 받는다.

```kotlin
// 기능의 입력은 내비게이션 인자가 아니라 id 매개변수다.
@HiltViewModel(assistedFactory = CourseViewModel.Factory::class)
class CourseViewModel @AssistedInject constructor(
    @Assisted id: String,
    sessions: CourseSessions,
) : ViewModel() {

    private val session = sessions.get(id)
    val state: StateFlow<CourseSession.LoadState> = session.state  // 복사하지 않고 그대로 넘긴다

    fun load() {
        viewModelScope.launch { session.startLoadCourse() }
    }

    fun toggleTodo(item: TodoItem) {
        val current = (state.value as? CourseSession.LoadState.Loaded)?.course ?: return
        session.update(current.withToggled(item))  // 가정한 함수. 불변 모델이므로 copy로 새 Course를 만든다
    }

    @AssistedFactory
    interface Factory {
        fun create(id: String): CourseViewModel
    }
}

// 책의 CourseView. 상태를 골라 렌더러를 호출하는 일만 한다.
@Composable
fun CourseRoute(
    id: String,
    viewModel: CourseViewModel = hiltViewModel<CourseViewModel, CourseViewModel.Factory>(
        key = id,                                        // 키가 없으면 모든 페이지가 같은 인스턴스를 받는다
        creationCallback = { factory -> factory.create(id) },
    ),
) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    when (val s = state) {
        is CourseSession.LoadState.Failed -> InlineError(s.error.message.orEmpty())
        is CourseSession.LoadState.Loaded -> CourseScreen(course = s.course, onToggleTodo = viewModel::toggleTodo)
        CourseSession.LoadState.Loading -> {
            CircularProgressIndicator()
            LaunchedEffect(Unit) { viewModel.load() }  // 책의 .task { await courseSession.startLoadCourse() }
        }
    }
}

// 책의 PaginationView. CourseRoute가 무엇에 의존하는지 모른다.
@Composable
fun CoursePager(courseIds: List<String>) {
    val pagerState = rememberPagerState { courseIds.size }
    Column {
        CourseTabs(courseIds, pagerState)                // 책의 스크롤 메뉴. CourseRoute 밖에 둔다
        HorizontalPager(
            state = pagerState,
            key = { page -> courseIds[page] },
            beyondViewportPageCount = 1,                 // 이전, 현재, 다음 세 페이지만 구성한다
        ) { page ->
            CourseRoute(id = courseIds[page])
        }
    }
}

// 단독 목적지로 쓸 때는 인자를 꺼내 같은 CourseRoute에 넘긴다
fun NavGraphBuilder.courseDestination() {
    composable("course/{id}") { entry ->
        CourseRoute(id = checkNotNull(entry.arguments?.getString("id")))
    }
}
```

이 구조에서 확인되는 점은 세 가지다.

- **`ViewModel`이 진실 공급원이 아니다.** 사용자가 화면을 떠나 `ViewModel`이 정리되어도 세션과 `appScope`의 `updateJob`은 살아 있으므로 디바운스 중이던 변경이 서버까지 간다. 같은 코스로 돌아오면 세션이 이미 `Loaded`이므로 로드 없이 바로 그린다. 책의 §7.3.1이 묘사한 "다시 붙이기"가 그대로 일어난다.
- **`ViewModel`의 존재 이유가 바뀐다.** 이 `ViewModel`에는 비즈니스 로직이 없고 세션을 찾아 컴포저블에 연결하는 **라이프사이클 어댑터**만 남는다. UI-03 정리 문서의 "안드로이드의 `ViewModel`은 관심사 분리보다 라이프사이클 대응이 주된 존재 이유"라는 관찰이 여기서 드러난다.
- **같은 기능 뷰가 페이지, 단독 목적지, 딥링크 도착지로 모두 쓰인다.** `CoursePager`는 `CourseSessions`나 `CourseService`를 모르며 책의 "`PaginationView`는 `CourseView`를 돌보지 않는다"를 만족한다.

## 디바운스를 단위 테스트하기 (Ch7)

`CourseSession`이 순수 Kotlin 클래스이므로 `kotlinx-coroutines-test`의 가상 시간으로 1초 디바운스를 실제로 기다리지 않고 검증한다.

```kotlin
class CourseSessionTest {

    private val course = Course(id = "c1", title = "Italian")  // 필드 구성은 가정한 것이다

    @Test
    fun `연속된 변경은 마지막 것 하나만 1초 뒤에 백엔드로 간다`() = runTest {
        val service = FakeCourseService(course)
        val session = CourseSession(course.id, service, scope = backgroundScope)
        session.startLoadCourse()

        session.update(course.copy(title = "A"))
        advanceTimeBy(500.milliseconds)
        session.update(course.copy(title = "B"))  // A의 대기를 취소한다
        advanceTimeBy(999.milliseconds)
        assertTrue(service.updated.isEmpty())     // B 이후 아직 1초가 지나지 않았다

        advanceTimeBy(2.milliseconds)
        assertEquals(listOf("B"), service.updated.map { it.title })
    }
}

private class FakeCourseService(private val course: Course) : CourseService {
    val updated = mutableListOf<Course>()
    override suspend fun loadCourse(id: String): Course = course
    override suspend fun update(course: Course) { updated += course }
}
```

`CourseSession`이 스코프를 생성자로 받으므로 `runTest`의 `backgroundScope`(같은 가상 시계를 쓰고 테스트가 끝나면 취소된다)를 그대로 넘길 수 있다. 에뮬레이터도, UI 테스트도 필요 없다. 같은 방식으로 "같은 상태면 아무것도 하지 않는다"와 로드 실패도 검증한다.

## 대시보드: 타일 목록을 받는 레이아웃 (Ch8)

```kotlin
// :feature:dashboard. 타일이 무엇인지, 무엇에 의존하는지 모른다.
interface DashboardTile {
    val order: Int
    val span: Int
    @Composable fun Content(modifier: Modifier)
}

@Composable
fun DashboardScreen(tiles: List<DashboardTile>) {
    LazyVerticalGrid(columns = GridCells.Fixed(3)) {
        items(tiles, span = { GridItemSpan(it.span) }) { tile -> tile.Content(Modifier.fillMaxWidth()) }
    }
}

// :feature:chats. 가운데 층(기능). 로딩과 오류는 여기서 끝난다.
class UnansweredChatsTile @Inject constructor() : DashboardTile {
    override val order = 0
    override val span = 1

    @Composable
    override fun Content(modifier: Modifier) {
        val viewModel: UnansweredChatsViewModel = hiltViewModel()
        val state by viewModel.state.collectAsStateWithLifecycle()
        when (val s = state) {
            ChatsState.Loading -> TileLoading(modifier)
            is ChatsState.Failed -> TileError(onRetry = viewModel::load, modifier = modifier)  // 다른 타일에 영향을 주지 않는다
            is ChatsState.Loaded -> ChatsUI(s.chats, onOpenChat = viewModel::openChat, modifier = modifier)
        }
    }
}

@Module
@InstallIn(SingletonComponent::class)
interface ChatsTileModule {
    @Binds @IntoSet
    fun bindTile(tile: UnansweredChatsTile): DashboardTile
}
```

`DashboardScreen`이 맨 뒤 층, `UnansweredChatsTile`이 가운데 층, `ChatsUI`가 맨 앞 층으로 Figure 8.7의 세 층에 하나씩 대응한다. 각 기능 모듈이 타일을 `@IntoSet`으로 기여하면 `:feature:dashboard`는 `:feature:chats`에 의존하지 않고, 앱 모듈이 그래프를 조립한다. Sane DI의 "바깥 가장자리에서 조립한다"가 Hilt 멀티바인딩으로 실현된다. 멀티바인딩의 `Set`은 순서를 보장하지 않으므로 `order` 같은 명시적 정렬 기준이 필요하다. 타일마다 `ViewModel` **클래스**가 다르므로 키 없이 `hiltViewModel()`을 불러도 서로 다른 인스턴스를 받고, 한 타일의 실패는 그 타일의 상태에만 머문다.

## 설정 리스트: 섹션이 행을 내놓는다 (Ch8)

§8.4.4의 "섹션이 자신의 행을 부모에게 넘긴다"는 `LazyListScope` 확장 함수로 거의 문자 그대로 옮겨진다.

```kotlin
// :feature:settings:security. 섹션이 자기 행을 부모의 리스트에 내놓는다.
fun LazyListScope.securitySection(
    state: SecurityState,
    onToggleTwoFactor: (Boolean) -> Unit,
    onResetPassword: () -> Unit,
) {
    item(key = "security-header") { SectionHeader("Security") }
    item(key = "security-2fa") { SwitchRow("Enable 2factor Authentication", state.twoFactorEnabled, onToggleTwoFactor) }
    item(key = "security-reset") { ClickableRow("Reset Password", onResetPassword) }
}

// 책의 SettingsView. 행을 모아 하나의 리스트로 그리는 일만 한다.
@Composable
fun SettingsRoute(
    security: SecuritySectionViewModel = hiltViewModel(),
    payments: PaymentsSectionViewModel = hiltViewModel(),
    /* profile, privacy, integrations, community 생략 */
) {
    val securityState by security.state.collectAsStateWithLifecycle()
    val paymentsState by payments.state.collectAsStateWithLifecycle()

    LazyColumn {
        securitySection(securityState, security::toggleTwoFactor, security::resetPassword)
        paymentsSection(paymentsState, onOpenPayout = payments::openPayout)
    }
}
```

`SettingsRoute`가 받는 것은 섹션 `ViewModel` 여섯 개로 Listing 8.2와 같은 수이며, `PaymentsAPI` 같은 의존성은 각 섹션 `ViewModel`의 생성자에 들어가므로 `SettingsRoute`는 모른다. `SectionHeader`, `SwitchRow` 같은 공용 행 컴포저블과 하나의 `LazyColumn`이 함께 "균일한 모습의 응집된 전체"를 만든다. `LazyColumn`의 `key`는 리스트 전체에서 유일해야 하고 여러 모듈에 흩어진 섹션의 키 충돌은 컴파일러가 잡지 못하므로, 섹션 이름을 접두사로 붙이는 규칙을 두는 편이 안전하다.

## 다만 조금 다른 것

**`StateFlow`는 같은 값을 다시 내보내지 않는다.** 그래서 `current == course` 검사가 없어도 화면은 다시 그려지지 않는다. 하지만 네트워크 호출은 여전히 일어나므로 이 검사는 화면이 아니라 **서버 호출을 막기 위해** 필요하다.

**책이 말한 "백그라운드"는 프로세스 안의 백그라운드다.** `appScope`의 `updateJob`은 시스템이 앱 프로세스를 종료하면 함께 사라진다. 프로세스 종료 이후까지 보장해야 한다면 변경을 로컬 저장소에 먼저 기록하고 WorkManager로 전송한다. 이때도 "동기화는 모델 도메인이 책임진다"는 원칙은 그대로이며 달라지는 것은 수단뿐이다.

**알림은 기능이 직접 띄우지 않는 편이 낫다.** `CourseSession`이 `NotificationCompat`를 직접 부르면 순수 Kotlin 모듈이 아니게 된다. 세션은 "동기화가 끝났다"는 이벤트(`SharedFlow`)만 내보내고 알림이나 스낵바로 바꾸는 일은 앱 모듈의 관찰자가 맡는다. 책의 그림에서도 `CourseSession`과 Notification Center는 별개의 상자다.

**"기능 = 화면" 가정은 안드로이드 도구에 더 깊이 박혀 있다.** SwiftUI의 `@State`와 `@Observable`은 뷰 인스턴스에 붙으므로 같은 뷰를 세 번 놓으면 상태도 세 벌이 된다. 안드로이드의 `ViewModel`은 내비게이션 항목 단위로 범위가 정해지고 `SavedStateHandle`도 내비게이션 인자를 담는다. 그래서 자기 충족 원칙 3을 안드로이드에서 지키려면 부모가 건네는 최소한의 입력(`id`)이 **어디서 오는가**까지 열어 두어야 한다.

---

## 앞 챕터와의 관계, 그리고 Ch9로

**Ch5~8을 합쳐 보면.** `CourseView`의 입력과 진실 공급원은 이렇게 바뀌었고, Ch8은 그 작업이 무엇을 위한 것이었는지 보여 준다.

| 시점 | 바깥에서 쓰는 `CourseView`의 입력 | 진실 공급원 | 그 시점의 성질 |
|---|---|---|---|
| Ch4 | `@Binding var course: Course` | 부모 | 렌더링만 한다 |
| Ch6 | `id`, `courseService` | `CourseView`의 `@State` | 스스로 로드하고 변경을 순진하게 동기화한다 |
| Ch7 §7.2.4 | `courseSession` | **`CourseSession`** (모델 도메인) | 바인딩만 한다. 기능은 UI 없이 돈다 |
| Ch7 §7.3.3 | `courseSession` | `CourseSession` | 디바운스와 취소가 붙었지만 `CourseView`는 바뀌지 않았다 |

| 자기 충족 기능의 성질 | 만든 곳 | Ch8에서 쓰인 곳 |
|---|---|---|
| 스스로 로드한다 | Ch6 §6.3 | §8.1.3 페이지마다, §8.2.3 타일마다 독립 로드 |
| 오류를 인라인으로 스스로 표시한다 | Ch5 §5.1.2~5.1.3, Ch6 §6.3 | §8.1.3 알림으로 UI를 막지 않음, §8.2.4 한 기능의 실패가 번지지 않음 |
| 로딩과 렌더링이 분리되어 있다 | Ch5 §5.4 (`CourseView`/`CourseUI`) | §8.2.5 3계층, §8.4.4 섹션과 리스트의 분리 |
| 진실 공급원이 모델 도메인에 있다 | Ch7 §7.2 | §8.3.4 기능 사이 데이터 동기화를 모델 계층에 둔다 |

**UI-01·02·03과의 관계.** Ch7은 UI-01의 원칙 3과 §1.3.3을 이미 UI 쪽으로 기울어진 기능에 사후적으로 적용했다. Ch8의 3계층은 UI-03의 뷰 분류를 화면 하나에 쌓은 것이고, 첫 번째 교훈은 UI-03의 Note("기능 뷰가 항상 화면 전체를 쓰는 것도 아니다")를 결론으로 승격한 것이다. UI-02의 원칙 7(똑똑한 뷰보다 컴포지션)은 §8.2.2와 함께 읽어야 정확해진다.

**Book 0와의 관계.**
- §7.1.2와 §7.3.4는 `.docs/05-System-Wide-Testing-정리.md`의 "시스템 전반 테스트"를 기능 하나의 수준에서 실현한다.
- `CourseSession`을 누가 만드는가, 그리고 §8.3.2와 §8.4.5의 "의존성의 의존성을 몰라도 된다"는 `.docs/08-Sane-DI-정리.md`의 주제(바깥 가장자리 조립, ABC problem, §8.9 지연 의존성)다.
- §8.1.1의 "미루면 더 나은 정보로 설계한다"는 `.docs/03-HDD-Plan-To-Code-정리.md`와 `.docs/04-HDD-Strategic-정리.md`의 가설 주도 개발과 같은 태도다. UX 리서치가 "학생은 한 코스만 수강할 것"이라는 가설을 반증했고, 미뤄 둔 덕분에 그 반증을 반영해 설계할 수 있었다.

**Ch9로 넘어가며.** 책은 Ch8에서 내비게이션을 두 번 미뤘다. §8.1.3에서 "`CourseView`에 내비게이션 의존성을 미리 채워 두면 `PaginationView`는 몰라도 된다"고만 말했고 결론에서 "다음에는 내비게이션을 살펴보겠다"고 예고했다. 이제 기능은 페이지 하나, 그리드의 칸 하나, 리스트의 섹션 하나가 될 수 있다. 남는 질문은 다음과 같다.

1. 같은 `CourseView`가 페이저 안에 있을 때와 단독 화면일 때, 그 안의 버튼이 일으키는 내비게이션은 어떻게 달라져야 하는가.
2. 그 차이를 `CourseView`가 모른 채로 두려면 내비게이션을 어떤 형태의 의존성으로 건네야 하는가.
3. Ch4 이후 콘솔 스텁으로 남아 있던 내비게이션 액션(원칙 4)은 그 구조에서 어떤 모습으로 채워지는가.
4. Ch7이 숨은 전제로 남긴 "세션을 쥐는 바깥 계층"은 내비게이션 구조의 어디에 자리 잡는가.

Ch7과 Ch8은 **기능은 UI 없이 돌고, 화면이 아니며, 자기 충족적이라면 어떤 부모에게든 놓일 수 있다**는 것을 확정했다. 다음 챕터는 **놓인 자리에 따라 기능이 어디로 이동해야 하는가**를 다룬다.
