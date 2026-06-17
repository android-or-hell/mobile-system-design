# Scale 07·08 정리 — 언제·무엇·어떻게 모듈화하고, 그 모듈을 어떻게 다듬는가

> Mobile System Design 시리즈 Book 3 (Scale) 의 여덟 번째·아홉 번째 챕터.
> 핵심 질문 — *"토큰으로 UI 를 확장했으니, 이제 코드베이스 나머지를 모듈러 아키텍처로 어떻게 확장하는가 — 언제 모듈을 시작하고, 무엇을 먼저 추출하며, 추출한 모듈을 어떻게 성숙시키는가?"*

> *"모듈로 영역을 import 하기 전에 도메인이 본성을 드러내게 하라(Let the domain reveal its nature before you import its territory into a module)."* — Ch.7 에피그래프
> *"모든 모듈을 오픈소스 프로젝트처럼 다루라(Treat every module like an open-source project)."* — Ch.8 에피그래프

---

## Scale-07 · Modularization: When, What, How (전략·타이밍 챕터)

> Ch.5~6 이 *UI 확장*(토큰·디자인 시스템)을 닫았다면, Ch.7 은 *코드베이스 구조* 로 시선을 옮긴다. 이 챕터의 가장 큰 주장은 역설적이다 — *모듈을 만들지 않은 것이 의식적 결정이었다*. 모놀리스로 시작해 통증이 임계점에 이르렀을 때, *기초부터(bottom-up)* 추출하라는 것.

Ch.7 이 다루는 것:

- 모듈을 너무 일찍 시작하는 것이 앱을 해칠 수 있는 이유
- 팀과 앱이 자라며 *진짜 모듈러 필요* 가 어떻게 만들어지는가
- 흔한 함정을 피하는 *실용적 모듈 추출 프로세스*
- 새 기능에 모듈을 만드는 것이 *항상 통하지 않는 이유*
- *기초 모듈을 먼저* 정의하면 다른 모듈 추출이 쉬워지는 이유

---

### § 7.1~7.2 단순하게 시작 — 동작하는 모놀리스

현재 앱은 *단일 타깃·단일 빌드* 다. course 로직, 결제, 인증, UI 라이브러리가 전부 한 곳에 산다. *모듈을 만들지 않은 것은 실수가 아니라 의도* 였다 — 속도를 유지하기 위해서.

> **Note** — 여기서 "모듈" 은 일단 *별도로 빌드 가능한 타깃* 이라 가정한다. 모듈의 다양한 형태는 §7.5 에서 다룬다.

#### 모놀리스가 의미 있는 이유

- *앱을 머리에 담을 수 있다* — 인지 부하가 낮다. 기능 업데이트도 쉽다(어디서든 발견 가능, 어디서든 호출 가능).
- 모듈·패키지·package manager·모듈별 access level 유지보수가 불필요하다. public/internal 결정도 없이 *그냥 코드와 작업* 한다.
- "모든 새 기능은 자기 모듈을 가져야 한다" 같은 격언이 있지만 — *이 스케일에서 셋업이 얼마나 쉬운지 과소평가하지 마라*. 단순함은 결함이 아니라 기능이다.

#### § 7.2.1 도메인 발견의 이점

새 앱을 만들 때는 *도메인이 무엇인지 아직 발견 중* 이다. 초기 PRD 가 명확한 경계를 제안해도 *현실은 그 선을 흐린다*.

> Course 앱을 시작하며 Course / Payments / Authorization 도메인을 정의했지만, 구현하며 *실제 관계가 출현* 한다. 코스 진척이 결제 알림을 트리거하는가? 사용자 선호가 코스 추천과 결제 플로 양쪽에 공유되는가? — 도메인이 어떻게 상호작용해야 하는지뿐 아니라, *초기 도메인 개념 자체가 옳았는지* 를 학습하는 중이다.

#### § 7.2.2 조숙한 아키텍처 결정 회피

> *모듈은 문제 공간을 완전히 이해하기 전에 "무엇이 함께 속하는가" 를 결정하라고 강요한다.*

User 모듈로 시작했다면 모든 새 발견이 아키텍처 결정을 요구한다(코스 진척은 User 모듈? Course 모듈? 앱 자체?). 반면 *모놀리스에서 추상화를 잘못 잡으면 "클래스만 옮기면" 된다* — 모듈 분할·이동·import 수정이라는 재설계가 아니라.

> 저자의 비유: *Bert 와 Ernie 도 장난감 정리로 다툰다*. "소방차는 빨간 장난감 박스에" vs "분명 자동차 박스에". 두 머펫도 합의 못 하는데, 의견 강한 개발자들이 모듈 경계를 처음부터 결정한다고 상상해보라.

→ *유연하게 시작하되 명확한 내부 경계를 두고*, 빌드와 사용을 통해 실제로 출현하는 경계를 나중에 형식화한다.

---

### § 7.3 반론 — 모듈로 시작이 의미 있을 때

모놀리스 우선은 보편 법칙이 아니다. 처음부터 모듈이 합리적인 세 시나리오:

| 시나리오 | 설명 |
|---|---|
| **도메인 경계가 잘 이해됨** | 도메인에 확신이 있고 좋은 아키텍처 결정 경험이 충분하면 첫날부터 가능. 예: 핵심이 *비디오 처리* 라면 첫날 `VideoProcessor` 모듈. 기존 앱의 "App 2.0" 은 경계가 *전투 검증* 됨. e-commerce 를 여러 번 빌드한 팀은 User/Product/Cart/Payment 가 *장기 안정 경계* 임을 안다. |
| **팀 확장이 예측 가능** | 5 → 15명으로 6개월 안에 자란다고 *확신* 하면, 모놀리스의 임시 단순함은 *상속할 조정 문제의 가치가 없다*. 시간대·조직 경계로 분산된 팀에서 모듈 경계가 *자연스러운 조정 지점*. |
| **첫날부터 도메인 재사용 필요** | *다중 앱/타깃이 이미 필요* 하면 모듈은 즉시 필요. 예: Tutor 앱과 Student 앱을 동시 출시 → *공유 Payments 모듈*. 핵심은 *미래 예측이 아니라 비즈니스 요구가 이미 존재* 하는 것. |

> 그러나 우리 코스 앱은 이 어느 것도 해당하지 않는다. 단일 앱에서 튜터·학생이 같은 인터페이스로 상호작용하므로, 지금 공유 모듈은 *현재 문제 없는 복잡성* 이다.

---

### § 7.4 팀 확장의 통증 — 모놀리스가 균열하는 지점

> 통신 복잡성은 팀 크기와 함께 *지수적* 으로 증가한다. 3명 → 3채널, 4명 → 6채널, **8명 → 28채널**(Figure 7.2).

#### § 7.4.1 의도치 않은 결합 (가장 큰 원인)

무해한 편의(`CourseView` 에 `Video` 모델 import)가 *유기적으로* 자란다.

```
CourseView → VideoData (thumbnail)
Video → CourseProgress (완료 추적)
VideoAnalytics → Course
Course → VideoState (재생 위치)
VideoPlayer → UserCoursePreferences (화질 설정)
```

곧 Course 와 Video 가 너무 얽혀 하나를 바꾸지 않고 다른 하나를 변경할 수 없다.

> *진짜 문제는 Course 가 Video 에 의존하는 것이 아니라 — 의존성이 양방향이고 통제되지 않는다는 것.* 잘 설계된 시스템에서는 Course 가 Video 에 의존하되 Video 는 코스를 몰라야 한다. *모듈러 셋업에서는 이 장벽이 깨기 어렵다* — 의존성이 기본적으로 단방향이고, 역방향은 순환 의존성을 일으킨다.

#### § 7.4.2~7.4.4 나머지 통증

- **빈번한 머지 충돌** — 비디오 팀과 코스 팀이 같은 사용자 모델·네트워킹 레이어를 건드려 충돌. 공유 모듈로 변경이 *명시적·조정* 되면, 대형 PR 에 묻히는 대신 *별도·계획된·가시적* 변경이 된다.
- **기어가는 빌드 타임** — 큰 프로젝트에서 30~40분 빌드가 드물지 않다. 작은 변경이 광범위한 재빌드를 트리거(30초 → 10분 커피 브레이크). *자기 모듈에서만 일하면 빌드가 초·분 단위* 로 줄고 테스트에도 큰 이점.
- **우발적 지식 사일로** — 사일로 자체는 나쁜 게 아니다(한 팀이 Networking·Security 소유 = *의도된* 사일로). 문제는 모놀리스의 사일로가 *소유권이 아니라 복잡성* 주위에 형성된다는 것 — *전문성이 아니라 두려움* 에 기반한다. "무서운 네트워킹 코드를 이해하는 유일한 사람" 이 생긴다.

---

### § 7.5 모듈로 전환 — "모듈" 이란 무엇인가

> *모듈* = 독립적으로 개발·테스트·유지보수 가능한 별도 코드 조각. 더 단순하게는 *"파일과 다른 폴더가 들어있는 폴더"*. standalone 이지만 결국 앱(또는 다른 모듈)에 import 된다.

용어는 다양하지만 본질은 같다:

| 부르는 이름 | 무엇이냐 |
|---|---|
| *라이브러리* | 순수 코드 — 정적/동적 링크 |
| *프레임워크* | 리소스 포함(localization, 이미지 자산) |
| *프레임워크* | 작업 방식을 *지시* 할 만큼 많은 일을 함 (Jetpack Compose, SwiftUI) |
| *패키지 / 의존성 / third-party* | package manager 로 원격에서 가져옴 |

단순함을 위해 이 모두를 *모듈* 이라 부른다.

#### § 7.5.2 분배의 차이

- **로컬 모듈** — 프로젝트 리포 안. 메인 앱과 함께 개발·배포. *단일 리포의 단순함* + 모듈러 이점.
- **원격 모듈** — package manager/artifact 리포로 분배. *독립 릴리스 주기*, semantic versioning, *다중 프로젝트 공유*. 그러나 버전·호환성·릴리스 조정의 복잡성.

> *대부분 팀은 로컬 모듈로 시작* 하고, 다중 앱 공유나 팀 독립성이 단순함보다 가치 있을 때만 원격으로 마이그레이션한다. (이 챕터는 *로컬 모듈* 에 집중.)

---

### § 7.6 순진한 추출 프로세스 — feature-module 부터의 함정

가장 흔한 직관 — "기존 기능을 모듈로 옮기거나, 새 기능을 모듈로 만들자" — 은 *둘 다 함정* 이다.

#### § 7.6.1~7.6.2 순환 의존성과 인터페이스 우회

Course/Payments 도메인은 둘 다 어느 도메인에도 속하지 않은 *떠도는 공유 타입*(`User`, `NetworkClient`)에 의존한다. 여기서 새 `Social` 모듈(`MediaShare`)을 도입하면:

- 앱이 Social 모듈에 의존(정상).
- 그러나 Social 모듈이 *앱* 의 User/NetworkClient 에 의존 → **순환 의존성**(Figure 7.4).

```
:app ──▶ :social ──▶ :app   ⇒ 순환! 빌드 불가
```

*유혹적 우회* — Social 모듈 안에 `UserInterface`/`NetworkClientInterface` 를 정의하고, 앱이 concrete 타입을 inject(Figure 7.5). 컴파일 화살표가 `:app → :social` 단방향이 되어 순환은 깨진다. **단, 핵심 오해 주의** — 이때 `:social` 은 인터페이스만 담는 모듈이 아니다. 기능 구현 전부(MediaShare + 공유 UI·로직, 코드의 ~95%)가 그대로 들어 있고, *역방향 import 지점만* 인터페이스(소켓)로 대체된다.

#### § 7.6.3 인터페이스의 문제 — 확장하지 않는 해법

이 작은 예시조차 인터페이스 2개를 요구한다. 큰 스케일에서는:

```
10개 모듈 × {User, NetworkClient}  →  인터페이스 20개 (전부 미묘하게 다른 변형)
NetworkClient 에 메서드 1개 추가  →  10곳 수정
```

> *우리는 확장하려고 모듈을 쓰는데, 이 솔루션 자체가 확장하지 않는다.*

기존 기능(Course)을 추출해도 마찬가지로 *Course 모듈이 다시 앱 안의 타입에 의존* 해 순환이 생긴다(Figure 7.6). → *제대로* 다시 시도하자.

---

### § 7.7 실용적 추출 프로세스 — Foundational 모듈 먼저 (핵심)

> *더 나은 접근은 bottom-up* — 저수준 공유 타입을 먼저 추출해 feature-module 이 의존할 수 있는 *기초 모듈* 에 배치한다.

1. **`Fundamentals` 모듈 도입** — User, NetworkClient 같은 느슨한 공유 타입을 여기로. 이 모듈은 *의존성이 없는 "leaf" 모듈*(플랫폼 제공 Foundation 에는 의존 가능, 우리 코드에는 의존 안 함)(Figure 7.7).
2. **새 feature-module 도입** — Social 모듈이 Fundamentals 에 *직접 의존* 가능. 새 타입이나 인터페이스 없이 추가(Figure 7.8).
3. **기존 feature 추출** — Course, Payments 도 Fundamentals 에 의존하면 *그냥 작동*(Figure 7.9~7.10).

> *깔끔한 아키텍처 — 새 타입이나 interface module 을 도입할 필요가 없다!* 도메인이 2개든 20개든 같은 프로세스다.

> **핵심 교훈** — *기초 모듈을 정의·추출하는 데 먼저 투자하라. 그 후 위에 feature-module 을 얹어라.* §7.6 의 깨지기 쉬운 인터페이스 경계가 통째로 사라진다.

---

### § 7.8~7.9 결론과 정리 — 의도된 여정

> 모놀리스로 시작한 것은 *의도* 였다 — 모멘텀을 유지하고 경계를 "딱 맞게" 잡는 걱정을 회피하기 위해. 앱과 팀이 자라며 인지 복잡성이 임계점에 이르면(앱이 더는 "머리에 담기지 않음"), 그때 *단방향 아키텍처* 로 분할한다 — "clean architecture" 를 위해서가 아니라 *실제 문제에 대응* 하기 위해.

**Ch.7 한눈에:**

| 주제 | 요점 |
|---|---|
| 왜 처음부터 모듈이 아니었나 | 도메인 발견 중엔 모놀리스가 빠르다. 조숙한 모듈화는 *나중에 후회할 경계* 에 묶이고, 빌드 오버헤드·마찰을 더한다. |
| 모듈로 시작이 맞을 때 | 도메인 경계가 명확 / 팀 확장이 확실 / 첫날부터 다중 앱 재사용. |
| 모놀리스의 한계 | 머지 충돌·우연 결합·느린 빌드. 누구나 내부 코드 접근. 사일로가 *복잡성* 주위에 형성. |
| 순진한 추출의 역효과 | feature-module 부터 = 순환 의존성 → 확장 안 되는 인터페이스 우회. |
| 실용적 추출 | *기초(Foundational) 모듈을 먼저* 추출 → 순환 없이, 기능 재작성 없이, 인터페이스 없이. |

---

## Scale-08 · Module Design (추출 후 정제 챕터)

> Ch.7 이 *모듈을 어떻게 만드는가* 였다면, Ch.8 은 *만든 모듈을 어떻게 다듬는가* 다. 대부분 팀은 "파일 옮기고 import 고치고 끝" 이라 부르지만, 거기서 멈추면 valuable 한 것을 놓친다. 관통하는 정신 모델은 — *모든 모듈을 오픈소스 프로젝트처럼 다루라*.
>
> 이 챕터는 *feature 모듈*(사용자가 직접 상호작용, 뷰 + 비즈니스 로직)에 집중한다. Course 모듈이 전형적 예. 유틸리티·Foundational 모듈은 Ch.9 에서.

---

### § 8.1 Access Control 의 한계 — 모듈이 강제하는 경계

추출 직후 컴파일러 에러를 따라 *블라인드로 public 을 붙이는* 유혹이 있다. 그러나 이 반응적 접근은 빠르게 컴파일되는 대신 *부풀린 public 인터페이스* 를 남긴다.

#### § 8.1.1~8.1.2 모듈의 두 가지 슈퍼파워

모놀리스에서 `internal`(비-private) 코드는 *같은 타깃의 모든 코드* 가 접근 가능했다. 그래서 Settings 팀의 Steve 가 Course 모듈의 `TutorView` 를 몰래 쓸 수 있었다(Figure 8.2).

> **Note** — Swift 는 `internal` 이 default, Kotlin 등 일부 언어는 `public` 이 default.

모듈은 두 슈퍼파워를 준다:

1. *`internal` 의 의미 변화* — "앱 안 누구나" → **"이 모듈 안 코드만"**.
2. *`public` access level 획득* — 외부에 보일 것을 *선택*.

```swift
public struct CourseView: View { ... }   // 노출
struct TutorView: View { ... }            // 모듈 안에 숨김 (internal)
```

이제 Steve 가 `TutorView` 를 쓰면 *컴파일 에러*. 그가 public 으로 바꾸려면 *명시적 PR* 이 필요하고, 모듈 소유자가 거부할 수 있다. shortcut 이 *명시적* 이 된다.

#### § 8.1.3 API 는 약속이다

> *모듈의 public API 는 미래 호환성 약속이다.* 모든 public 타입·함수·프로퍼티는 앞으로 지원하겠다는 약속이다. *작고 집중된 채로 유지하라.*

- public 타입 100개 → 모든 변경이 메인 앱을 깬다.
- `paymentFlow.start()` 한 줄 → *3-스크린에서 300-스크린* 플로로 바꿔도 앱에 영향 없음.

> *지금 디자인하지 않으면 영원히 안 한다.* 다시 기능 빌드 모드로 돌아가기 *전* 에 모멘텀을 활용하라.

---

### § 8.2 Public API 줄이기 — 레이어드 API 설계

추출 후 *소비자가 진짜 필요로 하는 것* 부터 시작한다(우리가 빌드한 것이 아니라).

| Course 모듈 — 노출할 것 | 숨길 것 |
|---|---|
| 데이터 fetch `courseAPI.load(id:)` | 서브뷰 `TutorView` |
| 진척 추적 `course.todos`, `updateProgress(...)` | 내부 캐시 `CourseCacheManager` |
| 모델 `CourseProgress`, `CourseStatus` | DB 스키마 `CourseEntity`, 네트워크 빌더 `CourseAPIRequest`, 뷰 상태 헬퍼 `CourseLoadingState` |

> 모듈이 막 추출된 *지금이 크고 광범위한 breaking 변경에 적기* 다.

#### § 8.2.1 레이어드 API 설계 — 데이터 기반

`VideoPlayer` 를 그대로 노출하면 cacheManager·decoder·bufferManager·request 등 모든 내부 복잡성을 소비자가 조립해야 한다. 대신 *사람들이 실제로 어떻게 쓰는지* 분석:

- **89%** — 비디오 품질·autoplay 만 커스터마이즈
- **8%** — 나쁜 네트워크용 timeout
- **3%** — 특정 콘텐츠용 specialized codec

→ 사용 분포가 그대로 *레이어드 API* 가 된다:

```swift
// Layer 1 (89%) — 기본 한 줄
VideoPlayer().play(videoId: "abc123")

// Layer 2 (8%) — 설정 struct
let config = VideoConfig(quality: .hd, autoplay: true, networkTimeout: 30, videoId: "abc")
VideoPlayer(config: config)

// Layer 3 (3%) — 고급 체이닝
VideoPlayer().with(decoder: .custom(...)).with(caching: .disabled)
```

> *섬세한 균형 — 유연성과 작은 API.* 모듈이 internal 타입으로 모든 초기화 부담을 짊어지고, 대부분의 소비자는 한 줄만 쓴다.

---

### § 8.3 모듈 성숙 — 샘플 앱과 비관습 테스트

반드시 즉시 할 필요는 없지만, 시간에 걸쳐 모듈을 개선할 *전투 검증된* 방법들.

#### § 8.3.1~8.3.2 샘플 앱

모듈과 같은 공간에 위치하는 *standalone 샘플 앱* — 빠르게 점프하고픈 *시나리오 버튼 목록* 이다. 각 시나리오가 다른 변형(happy path, problem path, 모호한 edge case)을 보여준다.

> Course 모듈 샘플 앱 예: 새 코스(진행률 0%), 거의 완료된 코스(완료 전환 트리거), tutor 공지 상태, 스케줄 충돌·오프라인 새로고침 실패 같은 에러 경로, "튜터가 코스를 제거했는데 학생이 브라우징 중" 같은 edge case 까지 즉시 점프.

이점:

- *모듈 소비자처럼 사고하게 강제* — public API 만으로 기능을 셋업하게 되어, 쓰기 어려운 메서드·복잡한 셋업·누락된 편의 함수를 빠르게 발견.
- 숨은 부수 이점 — 신규 팀원 *온보딩*, 몇 달 후 다른 프로젝트 통합, *모듈이 진정 standalone 인지 검증*.

#### § 8.3.3 UI 테스트 분리 — 책임의 경계

샘플 앱이 있으면 UI 테스트는 작은 점프다. 핵심 원칙은 *질문을 둘로 쪼개는 것*:

| 레벨 | 묻는 질문 | 어디서 |
|---|---|---|
| **App-level** | 버튼 탭 시 기능이 *시작·완료* 되는가? (이음새) | 메인 앱 — 1~2개만, 얕게 |
| **Module-level** | 모든 개별 화면·전환·상호작용이 제대로 작동하는가? (내부) | 샘플 앱 위 — 수십 개, 깊게 |

거대 UI 테스트는 사실 두 질문(연결 + 내부 동작)을 한 번에 물어, 셋업이 비대해지고(로그인→등록→강의 19개 완료) 무관한 변경에도 전부 돌아 느리고 flaky 하다. 분리하면 샘플 앱 시나리오 버튼이 상태를 즉시 만들어줘 셋업이 사라지고, 실패 시 범인이 명확해진다.

#### § 8.3.4 Public-only API 테스트 (비관습)

보통 `@testable import Course` 로 internal 까지 접근하지만, *`@testable` 을 생략하고 앱처럼 `import Course`* 하면 *public API 만* 테스트하게 된다.

```swift
// @testable import Course   ← 생략
import Course

class CoursePublicAPITests {
    func testCourseEnrollment() {
        let course = CourseAPI.loadCourse(id: "123")
        XCTAssertTrue(course.isEnrolled)
    }
}
```

> *실제 사용을 모방* 한다. 내부와 public 테스트가 흐려지지 않고, *public API 가 깨지는 변경의 신뢰할 수 있는 indicator* 가 된다. regular 단위 테스트의 *대체가 아니라 추가*.
>
> **Note** — Android 는 internal 코드가 테스트에서 기본 접근 가능하므로, public-only 테스트는 *별도 테스트 모듈* 로 물리 분리해야 한다(아래 매핑 참조).

---

### § 8.4 오픈소스 유지보수자처럼 사고 — 에러가 가이드가 되도록

> *강력한 멘탈 모델*: 모듈을 GitHub 에 공개 발행하고 *완전한 낯선 사람* 이 통합한다고 가정하라. 그러면 *부족 지식(tribal knowledge)*, Slack 대화, "Sarah 에게 물어봐" 에 의존할 수 없다.

숙련된 유지보수자가 절대 빠뜨리지 않는 것: *사전 컨텍스트를 가정하지 않는 문서*, *실제로 컴파일·실행되는 예시*, *합리적 default*, *문제 해결을 돕는 에러 메시지*, *직관적 public API*.

#### § 8.4.1 구현자를 안내하는 에러 메시지

```swift
// 가이드 없는 에러 — 개발자가 코드를 파거나 Slack에서 물어야
try await courseAPI.enroll(...)
// → "Enrollment failed: constraint violation"
//   (만석? prerequisite 부족? 스케줄 충돌? 알 수 없음)

// 가이드 있는 에러 — 에러가 곧 문서
catch CourseError.capacityReached(let waitlistInfo) { showWaitlistOption(waitlistInfo) }
catch CourseError.prerequisitesMissing(let required)  { showPrerequisites(required) }
catch CourseError.scheduleConflict(let conflicting)   { showConflictResolution(conflicting) }
```

> 차이가 극적이다. 두 번째 버전은 *에러를 문서의 일부* 로 만든다 — 정확히 무엇이 잘못됐고 무엇을 할 수 있는지. *같은 기준을 내부 모듈에도 적용* 하라. 미래의 자신과 팀원이 그 세심함에서 이익을 본다.

---

### § 8.5~8.6 결론과 정리 — 유일한 명료성의 순간

> 모듈을 추출하면 *유일한 명료성의 순간* 이 온다 — 컴파일러가 그 코드가 앱 나머지에 *연결되는 모든 곳* 을 알려준다. **빌드 에러 목록 = 공짜로 받은 의존성 전수 감사 보고서** 다(컴파일러는 거짓말을 못 하므로 누락 없음). 그리고 *초록불이 되는 순간 영영 소멸* 한다.

이 순간을 어떻게 쓰느냐가 갈림길이다:

- **반응 A (함정)** — 에러마다 기계적으로 public → "파일 위치만 바뀐 모놀리스". Steve 의 몰래 의존도 public 으로 공식 승인됨.
- **반응 B (이 챕터)** — 고치기 전에 에러를 한 줄씩 심문. 정당한 것만 public, 나머지는 연결 삭제·정식 API 로 교체 → 진짜 경계가 있는 모듈.

> *이 챕터의 아이디어들은 nice-to-have 가 아니다.* 작은 public 인터페이스(요구사항 변경 시 수정 시간 절감), public-only 테스트(다른 팀에 영향 주기 *전* 에 문제 포착), 샘플 앱(사용성 이슈 발견) — *중간~거대 앱* 에서 가장 잘 작동한다. *모티베이션이 있고 무엇이 변경돼야 하는지 명확한 시야가 있는 지금* 작업하라.

**Ch.8 한눈에:**

| 주제 | 요점 |
|---|---|
| 경계 바로잡기 | 컴파일러 만족시키려 전부 public 금지. 추출 직후 *모든 의존성이 보일 때* 활용. `internal` = "이 모듈만". 컴파일러가 경계 위반을 잡게. Public API 는 약속. |
| public API 줄이기 | *소비자가 필요로 하는 것* 에서 시작. API 가 지저분해진 신호 탐지. 복잡성을 단순 인터페이스 뒤에 숨김. *레이어드 API*. |
| 모듈 성숙화 | *샘플 앱* 으로 진짜 어려운 점 발견. *public-only 테스트* 로 breaking change 포착. 앱 테스트는 기능 시작·완료, 화면 테스트는 모듈에 위임. *낯선 사람이 쓴다고 가정* 하고 문서화. |

---

## Android / Jetpack Compose 매핑

> Ch.7~8 은 대부분 플랫폼 독립적인 아키텍처 원칙이다. 다만 Gradle 멀티모듈 / Kotlin 가시성 / Compose 데모 패턴에서 iOS 예제와 *구체가 갈리는* 지점이 있어 옮긴다.

#### 모듈 = Gradle 모듈, "Foundational 먼저" = `:core` 우선 (§7.7)

iOS 의 별도 빌드 타깃은 Android 에선 Gradle 모듈(`:feature:course`, `:core:network`)이다. §7.7 의 bottom-up 추출은 Android 모듈화 가이드의 *권장 구조와 정확히 일치* 한다 — `:core:*`(network, model, common) 같은 leaf 모듈을 먼저 만들고 그 위에 `:feature:*` 를 얹는다. 순환 의존성은 Gradle 이 *"Circular dependency between :app and :social"* 빌드 에러로 즉시 거부하므로, §7.6 의 함정이 컴파일 단계에서 강제로 드러난다.

#### Access control 의 차이 — Kotlin `internal` 은 *모듈 스코프* (§8.1)

가장 큰 플랫폼 차이. Swift 는 `internal` 이 *타깃* 스코프지만, **Kotlin 의 `internal` 은 Gradle *모듈* 스코프** 다. 즉 Android 에서는 모듈로 분리하는 행위 자체가 §8.1 의 "슈퍼파워" 를 자동으로 부여한다 — `:feature:course` 의 `internal class TutorScreen` 은 `:feature:settings` 에서 *애초에 보이지 않는다*. 단 Kotlin 은 `public` 이 default 이므로(Swift 와 반대), 노출하고 싶지 않은 것에 *명시적으로 `internal` 을 붙이는* 규율이 필요하다. `api` vs `implementation` Gradle 의존성 선언이 §8.1.3 "API 는 약속" 의 빌드 그래프 버전이다 — `implementation` 으로 선언한 의존성은 소비자에게 전이되지 않아 public 표면을 좁힌다.

#### 샘플 앱 = `:feature:course:demo` 데모 모듈 (§8.3.1)

iOS 의 standalone 샘플 앱은 Android 에선 `com.android.application` 인 데모 모듈이다. 코드가 사실상 *시나리오 버튼 목록*:

```kotlin
// :feature:course:demo
@Composable
fun DemoMenu() {
    LazyColumn {
        item { DemoButton("새 코스 (진행률 0%)")        { nav.go { CourseScreen("demo-new", {}) } } }
        item { DemoButton("완료 직전 코스 (진행률 95%)") { nav.go { CourseScreen("demo-almost-done", {}) } } }
        item { DemoButton("오프라인 — 새로고침 실패")    { /* fake 오프라인 주입 */ } }
        item { DemoButton("학생 보는 중 튜터가 코스 삭제") { /* fake 삭제 이벤트 주입 */ } }
    }
}
```

메인 앱이라면 "진행률 95%" 재현에 빌드→로그인→등록→강의 19개 완료가 필요하지만, 데모에선 탭 한 번. 데모가 `:app` 없이 빌드되어야 하므로 *standalone 여부가 컴파일 단계에서 검증* 된다. Compose 의 `@Preview` 가 *컴포넌트 레벨* 에서 하는 일을 *플로 레벨* 로 확장한 것 — Now in Android 의 카탈로그/데모 모듈 패턴과 같은 발상.

#### Public-only 테스트는 *별도 모듈* 로 (§8.3.4)

Swift 는 `@testable` 생략만으로 public 면을 강제하지만, Kotlin 은 *같은 모듈의 테스트가 internal 에 기본 접근* 가능하다. 따라서 public-only 테스트는 물리적으로 분리해야 한다:

```kotlin
// (A) 일반 단위 테스트 — :feature:course/test/  (internal 접근 O)
class CourseCacheManagerTest { @Test fun `캐시 만료 시 재요청`() { ... } }

// (B) public-only — 별도 모듈 :feature:course:public-api-test
//     dependencies { implementation(project(":feature:course")) }   ← 일반 의존성
class CoursePublicApiTest {
    @Test fun `코스 로드 후 등록 여부`() {
        val course = CourseApi().loadCourse(id = "123")   // public 만 보임
        assertTrue(course.isEnrolled)
    }
}
```

6개월 뒤 `loadCourse(id: String)` → `loadCourse(request: CourseRequest)` 로 바뀌면, (A)는 통과해도 (B)는 *컴파일부터 실패* → "모듈 밖 모든 소비자를 깨는 변경" 경보가 PR 단계에 울린다. (A)는 "동작이 맞나", (B)는 "약속이 유지되나" — 목적이 달라 대체가 아닌 추가.

#### 빌드 타임·테스트 분리의 Android 체감 (§7.4.3, §8.3.3)

Gradle 의 *configuration cache + 모듈 단위 incremental 빌드* 가 §7.4.3 "자기 모듈만 빌드" 의 실체다. §8.3.3 의 테스트 분리는 Android 에서 특히 절실한데 — 결제 팀의 한 줄 변경이 `:app`/androidTest 전체(CI 40분)를 돌리던 것을, App-level Espresso 테스트 1~2개(이음새) + 데모 모듈 위 화면 테스트 수십 개(내부)로 쪼개면 코스 내부 수정 시 *demo 테스트만 몇 분* 으로 줄어든다.

---

## 정리 — *Ch.7 은 타이밍과 방향, Ch.8 은 정제*

- **Ch.7** 의 가장 큰 통찰은 *모듈화는 타이밍의 문제* 라는 것이다. 모놀리스로 시작한 것은 게으름이 아니라 *도메인이 본성을 드러낼 때까지 기다리는 의식적 선택* 이며, 통증이 임계점에 이르렀을 때 *기초부터 bottom-up* 으로 추출하면 순환 의존성·인터페이스 지옥을 통째로 피한다. "feature 부터 모듈로" 라는 자연스러운 직관이 왜 역효과인지, 왜 `Fundamentals` 모듈이 먼저인지가 핵심이다.
- **Ch.8** 은 *추출이 끝이 아니라 시작* 임을 말한다. 추출 직후는 컴파일러가 모든 의존성을 펼쳐 보이는 *유일한 명료성의 순간* 이며, 이때 public API 를 작게(레이어드 API), 샘플 앱으로 사용성 검증, public-only 테스트로 약속 보호, 공감 어린 에러로 문서화하면 모듈이 진짜로 성숙한다. 관통하는 한 문장 — *모든 모듈을 오픈소스 프로젝트처럼 다루라*.

> 두 챕터를 합치면 — *모듈은 clean architecture 를 위해서가 아니라 실제 통증에 대응해 만들고, 만든 직후의 명료성을 활용해 다듬는다.* 이제 모듈의 종류(feature / foundational / support)를 정식 카테고리로 나누는 Ch.9 로 이어진다.
