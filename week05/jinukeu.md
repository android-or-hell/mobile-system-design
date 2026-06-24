# Scale 09 정리 — 모듈의 세 카테고리: Feature · Support · Foundational, 그리고 의존성 계층의 패턴

> Mobile System Design 시리즈 Book 3 (Scale) 의 열 번째 챕터.
> 핵심 질문 — *"모놀리스에서 모듈을 추출했더니 전부 똑같은 '폴더'처럼 보인다 — 이것들을 역할별로 어떻게 분류하고, 각 역할에 맞는 *다른* API·테스트·자원 전략을 어떻게 적용하는가?"*

> *"카테고리가 없으면 모듈은 그저 화려한 폴더에 불과하다(Without categories, modules are just fancy folders)."* — Ch.9 에피그래프

> **이 챕터의 성격** — Ch.9 는 *개념적* 이다. "이 세 가지 수상작 아키텍처를 그대로 써라" 같은 레시피북이 아니라, 모듈을 바라보는 *공유 멘탈 모델* 을 준다. 정답 아키텍처는 없고, 오직 *어디에 노력과 자원을 둘지* 를 판단하는 틀이 있을 뿐이다.

---

## Scale-09 · Module Categories (분류 챕터)

> Ch.7 이 *모듈을 언제·무엇·어떻게 만드는가*, Ch.8 이 *만든 모듈을 어떻게 다듬는가* 였다면, Ch.9 는 한 발 물러서서 묻는다 — *추출된 모듈들은 사실 다 같지 않다*. 어떤 것은 사용자를 향하고, 어떤 것은 다른 모듈을 떠받치며, 어떤 것은 모두의 바닥에 깔린다. 이 역할 차이를 *세 카테고리* 로 정식화하면, 의존성 그래프에서의 위치만으로 API 크기·테스트 깊이·인력 배치가 *자동으로 따라 나온다*.

Ch.9 가 다루는 것:

- *명확한 구조 없이* 모듈이 자랄 때 떠오르는 문제
- *다른 모듈 타입은 다른 디자인 접근* 을 요구한다는 사실
- *카테고리화* 가 아키텍처 부패와 팀 갈등을 어떻게 방지하는가
- 세 모듈 타입(Feature / Support / Foundational)과 그 *고유 책임*
- 장기 코드베이스 건강을 지원하는 *지속 가능 패턴*

---

### § 9.1 비구조 모듈의 문제 — 카테고리 없으면 그냥 폴더다

처음엔 모듈화가 *진보처럼 느껴진다*. 폴더가 정돈되고, 빌드 타깃이 작아지고, 모던해 보인다. 그러나 *코드를 폴더로 옮기고 "모듈" 이라 부른다고 강한 아키텍처가 자동으로 보장되지는 않는다*. 모든 모듈이 똑같아 보이면, 무엇이 안전한 변경이고 무엇이 앱의 절반을 깨뜨릴 변경인지 구분할 수 없다.

| 무구조 모듈의 4가지 통증 | 증상 |
|---|---|
| **§ 9.1.1 흐려진 우선순위** | 단순한 `StringUtils` 를 고치려는데 *다른 3개 모듈이 import* 하고 있음을 발견. 어떤 게 안전·위험한지 모른다. *소유권도 흐려져* `SharedModels` 가 모든 팀의 *덤핑 그라운드* 로 자란다. |
| **§ 9.1.2 침식되는 아키텍처** | 데드라인이 닥치면 누군가 Marketplace 에 user 데이터가 필요하다며 *적절한 채널 없이 `UserProfile` 을 직접 import*. 6개월 후엔 Shopping/Commerce/Payments 중 어디에 코드를 둘지 아무도 모른다 — *모두 상호의존*. |
| **§ 9.1.3 "어느 박스에" 토론** | 코드 리뷰가 기능 검토가 아니라 *배치 토론* 이 된다 — "이거 `Utils` 야 `Helpers` 야?" 시간이 토론으로 샌다. |
| **§ 9.1.4 불균형 테스트** | 버튼 컴포넌트는 빈틈없이 테스트하면서, *모든 게 의존하는* 인증 모듈은 "복잡하니까" 최소 커버리지. **단순한 부분은 over-engineering, 결정적 부분은 under-engineering**. |

#### § 9.1.5 구조를 더한다는 것

> 이 문제들은 *모듈화 자체에 대한 반대가 아니다*. *구조 없이* 모듈화하고, 개발자의 *shortcut* 으로부터 모듈러 아키텍처를 보호하지 않은 것에 대한 반대다.

> **핵심** — *완벽한 단일 아키텍처는 없다*. 대신 *멘탈 모델을 공유* 한다. 어떤 모듈은 더 안정적이어야 하고, 어떤 것은 더 일반적이어야 하며, 어떤 것은 더 단순해야 한다 — 카테고리는 *어디에 노력과 자원을 집중할지* 를 알려주는 지도다.

---

### § 9.2 세 카테고리 한눈에 — Feature · Support · Foundational

앱(App)은 계층 *맨 위* 에서 모든 것을 "볼 수 있고", 보통 feature 들을 *접착(glue)* 하는 역할을 한다(Figure 9.1). 그 아래로 세 카테고리가 *의존성 그래프의 서로 다른 높이* 에 자리잡는다.

| 카테고리 | 한 줄 정의 | 그래프 위치 | 예시 | 비즈니스 로직 |
|---|---|---|---|---|
| **Feature** | standalone 가치를 전달하는 *완전한 사용자 여정* | 앱 *바로 아래* (맨 위) | Course, Marketplace, Onboarding | 많음 (도메인 ⭐⭐⭐) |
| **Support** | 다중 feature 가 필요로 하지만 *그 자체로는 완전한 경험이 아닌* 능력 | *중간* (샌드위치) | Authentication, Social, FeatureFlags, A/B Testing | 있음 (도메인 ⭐⭐) |
| **Foundational** | 모든 것이 그 위에 빌드되는 *안정적 기술 인프라* | *맨 아래* (leaf) | Networking, UI Library, Analytics, Utils | 거의 없음 (도메인 ≈ 0) |

#### § 9.2.1 Feature module

> *Feature module 은 standalone 가치를 전달하는 완전한 사용자 여정이다.* End-to-end 경험을 소유한다. 고객은 "코스를 둘러보고 싶다", "진척을 확인하고 싶다" 를 생각하지, *결제·인증은 그 부산물* 일 뿐이다(Figure 9.2).

- *모듈러 아키텍처를 채택하는 주된 이유* — 빌드하는 경험 주위의 **고립(isolation)**.
- *좋은 scoping 테스트*: **제거하면 완전한 사용자 플로가 사라지지만 앱은 여전히 작동** 하면, 적절히 scoped 된 feature 다.

#### § 9.2.2 Foundational module

> *Foundational module 은 모든 것이 그 위에 빌드되는 안정적 기반* 이다. 의존성 그래프 *맨 아래*. 네트워킹 스택, UI 컴포넌트 라이브러리, analytics 프레임워크, 유틸 모음 등(Figure 9.3).

- 비즈니스 로직이 거의 없어 *feature 가 아니다*. **사용자가 신경 쓰지 않으면 feature module 이 아니다** — 강력한 indicator.
- *의존자(dependent)는 많지만 의존성(dependency)은 적은 leaf 모듈*. 때로 다른 foundational 에 의존한다(예: UI Library → Fundamentals).
- > **Note** — *Core / Platform / Base module* 이라고도 부른다. 이름은 본인 선택.

#### § 9.2.3 Support module

> *Support module 은 다중 feature 가 필요로 하는 문제를 해결하지만, 그 자체가 완전한 사용자 경험은 아니다.* feature 와 foundational 의 *중간 지대*(Figure 9.4).

- 예: *Authentication*(생체/PIN 인증), *Social*(SNS 공유). 고객은 "오늘 인증해야지, 신난다!" 라고 생각하지 않는다 — 코스에 접근하고 싶을 뿐. *인증은 부산물*.
- UI 가 없어도 된다 — *FeatureFlags / A/B Testing* 도 support 다(다른 feature 가 다른 화면을 보이게 돕는다).
- > **Note** — *capability module* 이라고도 부른다.

#### § 9.2.4 왜 이 분류가 중요한가

> Social 을 feature 라 부르든 support 라 부르든 *왜 중요한가?* — **앱이 진지해지고 확장되면 결정적으로 중요해진다.** 각 카테고리는 *근본적으로 다른 디자인 요구* 를 갖는다. 분류는 팀이 *시간을 어디에 투자할지* 를 결정한다 — *API 디자인, 테스트 전략, 가장 강한 개발자를 어디에 둘지* 까지 전부.

---

### § 9.3 Feature 모듈 — 사용자를 향한 자율성(User-Facing Autonomy)

> *Rule of thumb* — 사용자가 *무언가를 달성하려고* 앱을 열 때, 그것이 feature 다. Feature module 은 *다듬어진 UX, 포괄적 에러 핸들링, 사용자 중심 테스트* 에 투자할 곳이다.

#### § 9.3.1 디자인 원칙 — 의견을 강하게(opinionated)

> *Marketplace 모듈은 17개의 다른 컨텍스트에 임베드될 걱정을 할 필요가 없다.* 둘러보기·구매 경험에만 집중한다. Course 가 두 위치에 임베드되더라도 *"generic 하고 reusable" 할 필요가 없다*. — **"이 모듈을 쓰는 방법은 하나" 라고 자신 있게 말할 수 있다.**

#### § 9.3.2 자율성과 확장(Autonomy and Scaling)

> *작은 API 로 — 다른 누구에게도 영향을 주지 않고 모듈 내부를 완전히 재설계할 수 있다.* Course 의 데이터 레이어를 재작성? 가능. 다른 UI 프레임워크를 실험? 앱은 신경 쓰지 않는다 — *public 인터페이스가 안정한 한*.

- *Feature 는 가장 격리된 타입* 이라, **feature 팀이 서로의 작업을 거의 방해하지 않으며** 효과적으로 확장한다.
- 대조 — foundational 의 `AnalyticsEvent` struct 하나를 바꾸면 *수십 개 feature 의존성을 검증* 해야 한다. **Feature module 은 이 조정 오버헤드를 완전히 피한다.**

#### § 9.3.3 Feature 는 변화를 끌어안는다(embrace change)

> *Feature module 은 앱 아키텍처 변경에 가장 취약하다.* CEO 가 마음을 또 바꿀 때, 디자인이 업데이트될 때 — *feature 팀이 적응* 한다. SwiftUI/Jetpack Compose 가 다음 새것으로 교체되면 *feature 팀이 전체 플로를 재구축* 한다(반면 네트워킹 스택은 그대로 작동). 사용자가 VR 지원을 요구하면 feature 가 구현을 떠안고, 제품 팀이 다른 onboarding 을 A/B 테스트하면 Onboarding 모듈이 재작성된다.

> **그러므로 feature module 은 *적극적 캡슐화* 에서 이익을 본다.** 다음 메이저 플랫폼 업데이트가 도착해도 — *독립적으로, 빠르게, 앱 나머지를 깨지 않고* 진화할 수 있다.

---

### § 9.4 Support 모듈 — Feature 뒤의 enabler

> *Support module 은 기능을 가능하게 하는 신뢰할 수 있는 일꾼이다.* 다중 feature 가 필요한 문제를 풀면서 *비즈니스 로직과 도메인 지식을 담는다*. 앱이 자라며 더 자주 등장한다.

#### § 9.4.1 Support vs Feature — 가장 흔한 혼동

> *Support module 은 feature module 로 가장(假裝)하기 쉽다.* 차이를 아는 것이 중요하다 — *고유한 디자인 도전* 을 갖기 때문.

- Social 모듈은 UI 가 있고 사용자가 실제로 SNS 에 공유한다. 하지만 사용자의 생각은 "sharing feature 를 쓰고 싶다" 가 아니라 *"완료한 코스를 자랑하고 싶다"* — **공유는 목표 달성의 수단**. Marketplace 와 Course 가 *둘 다 Social 을 직접 의존* 한다(Figure 9.5).
- > **간단한 테스트** — *비즈니스 로직이 무거운 모듈을 다중 feature 가 쓴다면, support module 일 가능성이 높다.*

#### § 9.4.2 통합의 도전(Integration challenges)

> *Support module 은 소비하는 feature 들에 의해 여러 방향으로 끌린다.* Course 팀은 한 방식을, Marketplace 팀은 다른 방식을, Subscriptions 는 또 다른 요구를 들고 온다.

- *scope 확장의 유혹* — "Payments 가 loyalty point 도 다룰 수 있나?", "VideoPlayer 가 이미지 슬라이드쇼도?" 모든 요청을 수용하면 *부풀어 뭐 하나도 잘 못하는 모듈* 이 된다.
- > **Support module 소유자에게는 *백본(backbone)* 이 필요하다 — 많은 요청에 "No" 라고 말할 수 있는.** 누군가는 *어떤 사용 사례를 지원하고 어떤 것을 거부할지* 어려운 결정을 내려야 한다.
- 예: Payments 팀은 "무엇이든 처리하는 generic transaction processor" 대신, *깊은 결제 이해* 로 *진짜 필요한 두 결제 타입 + 미래 확장 옵션* 만 짠다.
- > *기예(craft)는 균형이다 — 너무 경직되지도, 너무 generic 하지도 않은 모듈.* 추상화의 균형은 소프트웨어 엔지니어링의 craft 다.
- *현실 신호* — **Support 팀은 feature 팀보다 다른 팀과 훨씬 더 많이 대화한다.** 미팅을 사랑한다면 여기가 갈 곳.

---

### § 9.5 Foundational 모듈 — 인프라 레이어(The Infrastructure Layer)

> *Foundational module 은 앱의 빌딩 블록이다.* 비즈니스 로직이 거의 없고 주로 *기술 인프라*. feature/support 보다 *더 많은 책임* 을 진다 — 다중 팀이 의존하므로 *실패가 연쇄(cascade)* 되고 잘못된 결정이 모두에게 영향을 준다.

#### § 9.5.1 Foundational vs Support — 경계는 "비즈니스 관여"

> support 와 foundational 의 선은 흐릴 수 있다. 차이는 *비즈니스 관여(business involvement)* 다. **Foundational 은 비즈니스 입력이 거의 또는 전혀 없다.**

- Networking 모듈에서 — generic HTTP 요청·응답 파싱·에러 핸들링은 *foundational*. 그러나 `/api/courses` 와 `/api/payments` 엔드포인트를 알거나 도메인별 보안 규칙을 구현하면 *더 이상 generic 인프라가 아니다 — support territory*.
- *"Networking 에 코스 API 파싱을 추가해도 되나?"* 에 대한 두 답:
  - **support 로 보면**: "좋아!" 하지만 코스 파싱 로직·API 변경·Course 팀과의 조정을 떠안는다.
  - **foundational 로 보면**: "아니, 요청·응답 셋업은 *너의* 책임. 안 그러면 networking 이 모든 feature 와 강하게 결합한다."

#### § 9.5.2 변경이 더 계획적이다(Change is more planned)

> *Foundational 은 다른 세계 — 더 깊고 안정적인 레이어* 에서 작동한다. Apple/Google 이 새 제스처 패턴을 도입해도 analytics 프레임워크는 업데이트가 *불필요* 하고, 디자인 시스템이 flat → neumorphic 으로 바뀌어도 *네트워킹 레이어는 그대로*.

- *변경은 더 예측 가능하고 덜 빈번* 하다 — 새 플랫폼 지원, 언어 업데이트, 보안 패치, 성능 최적화. **플랫폼 릴리스 주기에서 일어나지, 사용자 피드백 주기에서가 아니다.**
- *안정성은 책임을 동반한다* — 변경이 드물어 *변경 하나의 stake 가 더 크다*. 따라서 *더 신중한 API 디자인, 더 철저한 테스트, 더 보수적인 진화* 가 필요하다.

#### § 9.5.3 Foundational 에 대한 경고 — junk drawer

> *Foundational 은 어디서나 import 되므로, vetted 되지 않은 코드의 dumping ground 가 되기 쉽다.* "Fundamentals 가 이미 import 돼 있으니 `UserSettings` 도 foundational 이지" 하며 테스트나 신중한 API 디자인 없이 쑤셔 넣는다.

> **함정 회피** — 모든 추가는 *feature 코드보다 더 신중히 리뷰* 하고, *전담 owner 를 지정* 하라.

#### § 9.5.4 알 수 없는 소비자를 위한 빌드(Building for unknown consumers)

> *Foundational 은 generic 코드 작성을 강제한다.* feature 에서는 정확히 무엇을 빌드하는지 알기에 가정을 hard-code 할 수 있지만, *foundational 은 모두에게 봉사한다 — 오늘과 내일 모두*.

- `TextField` 는 단순 입력만 가정할 수 없다 — 특정 제스처나 커스텀 search bar 안에 놓일 수도 있다. `Money` 타입은 USD/EUR 로 시작해도 *새 통화, 다른 소수점, RTL 형식* 을 우아하게 다뤄야 한다.
- > *Feature 는 specific 하고 의견 강하게.* *Foundational 은 알 수 없는 미래 요구를 사고하면서도 깨끗하고 사용 가능한 인터페이스를.* — generic·reusable 코드에 대한 지식이 필수다.

---

### § 9.6 API 표면 디자인 — 카테고리별 크기

> *API 표면 크기가 카테고리 차이를 가장 명확히 보여준다.* API 표면 = 모듈이 외부에 노출하는 모든 것. *작은 표면은 변경을 쉽게, 큰 표면은 안정성을 비용으로 치르고 유연성을* 준다.

| 카테고리 | API 표면 | 소비자 수 | 변경 위험 | 디자인 태도 |
|---|---|---|---|---|
| **Feature** | *가장 작음* (Figure 9.6) | 보통 하나 (앱) | 낮음 → 적극적 리팩터·실험 가능 | 의견 강하게, 전부 internal |
| **Support** | *중간* (Figure 9.7) | 다중 feature | 큼 → 한 팀의 버그픽스가 다른 팀에 문제 도입 | specificity ↔ reusability 균형 |
| **Foundational** | *가장 큼* (Figure 9.8) | 위 모두 | 가장 큼 → 변경·버그가 *모두에게 공짜로* 전파 | 광범위한 적용성 우선 |

- **§ 9.6.1 Feature** — UI 레이아웃·내비게이션·에러 핸들링·비즈니스 규칙이 전부 internal. *트레이드오프는 재사용성* — Course 는 교육 콘텐츠에 최적화돼 제품 카탈로그나 레시피 컬렉션에 쉽게 적응하지 않는다. *그러나 모든 게 generic 할 필요는 없다.*
- **§ 9.6.2 Support** — *다중 feature 에 봉사할 만큼 generic + 도메인 복잡성을 다룰 만큼 specific*. Social 은 `shareToSocial()` 한 줄로 끝나지 않는다 — Marketplace 는 product preview, Course 는 커스텀 인증서 이미지, Chat 은 avatar + 사진. *이미지 cropping, 콘텐츠 필터, 다양한 형식, preview 생성* 을 지원해야 하므로 *더 큰 API*. *더 많은 통신과 테스트* 가 필요하다.
- **§ 9.6.3 Foundational** — *가장 큰 public API*. UI Library 는 거의 모든 뷰·컴포넌트가 사용 가능하고, Utils 는 통화 변환기·날짜 포맷터·문자열 확장을 노출. *이점*: 변경 시 모두가 *공짜로 업데이트* 를 받는다. *대가*: 버그를 도입해도 *모두가 공짜로* 받는다 → **가장 철저한 테스트와 가장 신중한 API 디자인.**
  - > **Note** — 예외: Analytics 모듈은 foundational 이면서도 `trackEvent()` / `trackPage()` 만 노출하고 추적 로직은 숨긴다. *큰 표면이 foundational 의 법칙은 아니다* — "어떻게 쓰이는가" 가 표면을 결정한다.

---

### § 9.7 완전한 그림 — 의존성 계층이 모든 것을 설명한다

**빠른 카테고리화 가이드:**

- *사용자가 앱을 열어 이걸 한다* → **Feature module**
- *다중 feature 가 이 능력을 필요로 한다* → **Support module**
- *비즈니스 로직이 거의 없는 기술 인프라* → **Foundational module**

세 카테고리를 한 장의 계층으로 그리면(Figure 9.9), *위치만으로* 거의 모든 속성이 따라 나온다.

```
            ┌─────────────────────────────────────┐
            │                App                  │  모든 것을 보고 feature 들을 접착
            └──┬─────────────┬──────────────┬─────┘
       ┌───────▼──┐   ┌──────▼──────┐   ┌───▼────────┐
       │  Course  │   │ Marketplace │   │ Onboarding │   ◀ Feature (맨 위)
       └──┬────┬──┘   └──┬──────┬───┘   └─────┬──────┘     의견 강하게 · 작은 API
          │    └────┬────┘      │             │
       ┌──▼─────────▼──┐   ┌────▼─────┐        │
       │     Social    │   │   Auth   │◀───────┘            ◀ Support (중간/샌드위치)
       └──┬────────────┘   └────┬─────┘                       균형 · 중간 API
          │                     │
       ┌──▼─────────────────────▼──────────────────────┐
       │   Network  ·  UI Library  ·  Analytics  · Utils │   ◀ Foundational (맨 아래 leaf)
       └────────────────────────────────────────────────┘     generic · 큰 API

   변경 ▲ 위로만 흐른다   │   의존자 수 ▼ 아래로 증가   │   팀 격리 ▼ 아래로 감소
```

> 이 계층이 드러내는 *중요한 패턴* 들 —

| 패턴 | Feature (위) | Support (중간) | Foundational (아래) |
|---|---|---|---|
| **변경의 방향** | 변경해도 시스템 나머지 영향 없음 | 변경이 소비 feature 들에 파급 | 변경 시 *파급이 앱 모든 부분* 에 — *결코 아래로 흐르지 않음* |
| **의존성 수** | 보통 앱만 봉사 | 다중 feature | 위 전부 봉사 → *breaking change 피하기 가장 어려움* |
| **팀 격리** | *독립적* 으로 일 가능 | 다중 feature 팀과 *조정* | *시스템 전반 영향* 고려 |
| **유연성 패턴** | 의견 강하게 | specificity ↔ reusability 균형 | 광범위한 적용성 우선 |

> **계층은 더 나은 아키텍처 결정을 돕는다.** 누군가 foundational 에 비즈니스 로직을 추가하자고 제안하면 — *의존성 수를 가리키며* 조정 오버헤드를 설명할 수 있다.

> **자원 배분의 나침반** — *가장 강한 개발자와 가장 엄격한 프로세스* 는 *의존자가 가장 많은 모듈* 에 둔다: **foundational > support > feature**. (테스트 깊이·리뷰 엄격도도 같은 순서.)

> **Note** — 앱이 더 복잡해지면 *조정 자체* 를 위한 특수 솔루션이 필요해진다. *어떤 모듈은 다른 모듈 간 상호작용을 조율하기 위해 출현* 한다 — **orchestration module**. Ch.10 에서 다룬다.

---

### § 9.8 Ch.9 한눈에

| 주제 | 요점 |
|---|---|
| 무구조 모듈의 문제 | 흐려진 우선순위(안전/위험 변경 구분 불가) · 데드라인 shortcut 에 의한 침식 · "어느 박스에" 토론 · 불균형 테스트(단순 over, 결정적 under). *카테고리 없으면 화려한 폴더.* |
| 세 카테고리 | **Feature**(완전한 사용자 여정, 맨 위) · **Support**(다중 feature 의 비즈니스 능력, 중간) · **Foundational**(안정적 기술 인프라, 맨 아래). 각각 *다른 안정성·API·유지보수 전략* 요구. |
| Feature = 자율성 | 작은 API(재사용 불필요) · 잦은 변경 흡수 · 최소 cross-dependency 로 병렬 개발 · 다듬어진 UX·고객 중심 테스트. |
| Support = 균형 + 백본 | feature 보다 큰 API · 다중 컨텍스트 통합을 우아하게 · *강한 ownership 으로 scope creep 저항* · 변경이 다중 팀에 영향 → 폭넓은 조정. |
| Foundational = 인프라 | 가장 큰 API(재사용 빌딩 블록) · breaking change 가 앱 전체에 연쇄 → 신중한 유지보수 · *알 수 없는 미래 소비자* 를 위한 generic 디자인 · 가장 강한 개발자·가장 엄격한 프로세스. |

---

## Android / Jetpack Compose 매핑

> Ch.9 의 카테고리 모델은 *플랫폼 독립적인 사고 틀* 이지만, Android 에서는 이것이 **Now in Android(NiA) 권장 구조와 거의 1:1 로 맞아떨어진다** — `:feature:*` / `:core:*` 모듈 명명이 곧 이 카테고리다. 아래는 Pillo(복약 앱: DrugBank·FDA·Pillo 다중 API 통합) 도메인으로 옮긴 매핑이다.

#### 카테고리 ↔ Gradle 모듈 명명 (§ 9.2, § 9.7)

| MSD 카테고리 | Android(NiA) 관례 | Pillo 예시 |
|---|---|---|
| Feature | `:feature:*` (`com.android.library` 또는 dynamic-feature) | `:feature:medication`, `:feature:interactions`, `:feature:reminder` |
| Support | `:*`(도메인 능력 모듈, 별도 네임스페이스) | `:auth`, `:social-share`, `:analytics-domain`, `:drugbank`, `:fda`, `:pillo-api` |
| Foundational | `:core:*` (leaf) | `:core:network`, `:core:ui`(디자인시스템), `:core:analytics`, `:core:common`, `:core:model` |

의존 방향은 항상 **아래로만** — `feature → support → foundational`. Gradle 이 역방향(순환)을 *"Circular dependency between ..."* 빌드 에러로 즉시 거부하므로, § 9.7 의 "변경은 위로만 흐른다" 가 *컴파일 단계에서 강제* 된다.

#### Feature 끼리는 옆으로(↔) 의존하지 않는다 — 네비게이션·공유의 처리 (§ 9.3)

"이동한다 ≠ 상대를 안다." `:feature:medication` 이 `:feature:interactions` 로 *가고 싶을 뿐* 그 화면 코드를 알 필요는 없다. 사이에 추상화를 끼운다.

- **패턴 A (route 계약)** — 각 feature 가 진입점을 route 로만 노출하고 `:app` 이 `NavGraph` 를 조립. 공유되는 건 화면 코드가 아니라 *route 문자열/타입* 뿐이며, 그 정의만 `:core:navigation`(Foundational)에 둔다.
- **패턴 B (Navigator 인터페이스 + DI / 의존성 역전)** — `interface InteractionsNavigator` 를 공유 모듈에 두고, 구현은 `:feature:interactions` 의 `internal class`, 호출은 `:feature:medication` 에서. Hilt 가 런타임 주입 → *컴파일 타임 화살표 `medication → interactions` 가 생기지 않는다*.

> **공유 규칙** — *여러 feature 가 공유하는 순간 그건 더 이상 feature 가 아니다 → 아래로 승격시킨다.* 범용 컴포넌트(버튼/카드/테마)는 Foundational(`:core:ui`), 로직 있는 능력(인증/공유 시트)은 Support 로. 단 *"비슷해 보인다 ≠ 공유해야 한다"* — 섣부른 공유는 § 9.1 의 덤핑 그라운드를 재발시키므로, 진짜 공통 추상이 확인될 때까진 *복제가 더 싸다*. feature 끼리 옆으로 연결하고 싶어지는 순간이 곧 *"공유 대상을 아래로 내려라"* 는 신호다.

#### Support vs Foundational 을 가르는 두 질문 (§ 9.5.1)

UI 유무는 분류선이 *아니다* (디자인시스템도 인증도 UI 가 있다). 진짜 분류선은 두 질문:

1. **도메인/비즈니스 로직이 있는가?** — 있으면 Support, 없으면(순수 기술 인프라) Foundational.
2. **사용자가 그 존재를 (부산물로라도) 인지하는가?** — 인지하면 Support 쪽, 존재조차 모르면 Foundational.

> **객관적 증거 = import 방향.** 아래로 의존이 거의 없으면 Foundational(바닥 leaf), *위에서도 의존되고 아래로도 의존하면* Support(중간 샌드위치). `:core:ui`(디자인시스템)는 도메인 0 이라 Foundational, `:auth`(토큰·세션 정책을 앎)는 Support.

```
사용자 인식  목적 ───────── 수단(부산물) ───────── 존재조차 모름
           Feature        Support              Foundational
          도메인 ⭐⭐⭐      도메인 ⭐⭐             도메인 ≈ 0
그래프:  :feature ──→ :auth(Support) ──→ :core:network(Foundational)
                          (중간 샌드위치)        (바닥 leaf)
```

#### Kotlin `internal` = 카테고리 경계의 무료 집행관 (§ 9.6.1)

§ 9.6 의 "feature 는 전부 internal, foundational 은 대부분 public" 이 Android 에선 *공짜* 다. **Kotlin 의 `internal` 은 Gradle 모듈 스코프** 이므로, `:feature:medication` 의 `internal class MedicationDetailScreen` 은 다른 모듈에서 *애초에 보이지 않는다*. 다만 Kotlin 은 `public` 이 default(Swift 와 반대)이니, *노출하고 싶지 않은 것에 명시적으로 `internal` 을 붙이는 규율* 이 필요하다. Gradle 의 `api` vs `implementation` 선언이 § 9.6 표면 크기의 빌드그래프 버전 — foundational 의 전이 의존은 `api`, feature 내부 의존은 `implementation` 으로 표면을 좁힌다.

#### Analytics — how(전송) 와 what(텍소노미) 를 쪼갠다 (§ 9.6.3 예외)

"Analytics = Foundational(도메인 0)" 인데 이벤트 이름·유저 속성은 도메인 지식이라 모순처럼 보인다. 해법은 *한 모듈로 보지 않는 것*:

- **전송 인프라(how) = Foundational** — `:core:analytics` 는 `fun track(name, params)` / `setUserProperty(key, value)` *메커니즘* 만. Firebase/Amplitude 추상화·배칭. `"medication_logged"` 같은 이름은 *하나도 모른다* → 도메인 0 유지(§ 9.6.3 의 `trackEvent()`-만-노출 예외와 정확히 일치).
- **텍소노미·공유 유저 속성(what) = 도메인** — Foundational 에 넣으면 § 9.1 덤핑 그라운드가 된다. 기본값은 *분산*: 각 feature 가 자기 이벤트를 `internal object MedicationEvents` 로 소유하고 범용 client 를 호출. 여러 feature 가 공유하는 속성(`adherence_streak` 등)이나 명명 규칙 강제가 필요해지면 `:analytics-domain`(Support)로 승격.
- *신호* — 새 이벤트 추가하려고 `:core:analytics` 를 매번 수정 중이라면, 텍소노미가 잘못된 층에 있다는 강한 증거.

#### Foundational 의 base URL — 받기는 OK, 박기는 냄새 (§ 9.5.3 / § 9.5.4)

엔드포인트 경로(`/api/drugs`)는 도메인이라 Support 로 가야 하지만, base URL 은 비즈니스 규칙이 아니라 *"어느 서버를 가리키나"* 라는 **환경 설정값**이라 판정이 다르다.

```kotlin
// :core:network (Foundational) — 값을 "받는" 메커니즘만. 어떤 host 도 모른다.
interface NetworkConfig { val baseUrl: String }
class ApiClient(config: NetworkConfig) { /* config.baseUrl 사용 */ }

// :app — 실제 값 소유 (flavor 가 dev/staging/prod 결정)
@Provides fun config() = object : NetworkConfig { override val baseUrl = BuildConfig.BASE_URL }
```

`:core:network` 안에 `"https://api.x.com"` 하드코딩은 ⚠️ — dev/staging/prod 전환 불가, MockWebServer 테스트 불가(§ 9.5.2 "철저한 테스트" 자해), § 9.5.4 unknown-consumer 위반.

#### 다중 외부 서비스(DrugBank·FDA·Pillo) = base URL 이 아니라 *별개 Support 모듈* (§ 9.5.1)

"base URL 이 여러 개" 라도 **환경이 여럿**(dev/staging/prod = 설정값)과 **서비스가 여럿**(DrugBank/FDA/Pillo = 별개 통합)은 완전히 다른 축이다. 후자는 config 로 풀지 않고 *각 통합을 별도 Support 모듈로 쪼갠다* — host·엔드포인트·인증·응답 스키마가 한 묶음의 *정체성* 이기 때문(인증마저 다르다: DrugBank/FDA 는 API key, Pillo 는 Bearer).

```
:feature:medication   :feature:interactions
        │                    │
        ├─→ :pillo-api ─┐
        ├─→ :drugbank  ─┤  (각 Support: 자기 URL·인증·응답 모델·에러 매핑 소유)
        └─→ :fda       ─┤
                        ▼
            :core:network (Foundational: 공유 OkHttp·인터셉터·JSON, host 모름)
```

`:core:network` 는 `RetrofitFactory.create(baseUrl, extra: Interceptor?)` 만 제공. `:drugbank` 는 `create("https://api.drugbank.com/v1/", apiKeyInterceptor)`, `:fda` 는 `create("https://api.fda.gov/drug/")` 로 각자 인스턴스를 만든다 — 공유 커넥션 풀은 단일 OkHttpClient 재사용, 서비스별 차이는 각 Support 에 캡슐화 → *DrugBank 변경이 FDA 를 안 건드린다*.

#### 카테고리 vs 레이어 — 그리고 과모듈화 회피 (§ 9.5 보충)

- **두 축은 직교한다.** *카테고리*(Feature/Support/Foundational = 그래프상 역할)와 *레이어*(UI/domain/data = 내부 코드 종류)는 다른 축. 한 모듈 = 카테고리 1개 × 레이어 1개 이상. "Support 가 feature 를 갖나?" 는 축이 섞인 질문 — feature 는 *옆 카테고리* 이지 Support 내부 레이어가 아니다.
- Support 는 *필요한 레이어만* 갖는다: 데이터 통합형 `:drugbank` 는 data(주력)+domain(약간), **UI 없음(headless)**; 상호작용형 `:auth` 는 data+domain+**UI 있음**(로그인 화면).
- **과모듈화 회피** — `:drugbank` 를 처음부터 `:data:drugbank` + `:domain:drugbank` 로 쪼개는 건 대부분 과하다(통합 N개 × 레이어 M개 = N×M 폭발). 기본은 *단일 모듈 + 내부 레이어 패키지*. `internal` 로 `data` 패키지를 잠그면 모듈 분할 없이 *레이어 캡슐화의 90%* 를 얻는다. 분할은 측정 가능한 트리거(모델을 여러 모듈이 공유 + 무거운 deps 회피 / 실측 빌드 통증 / 팀 경계)가 올 때만. *비대칭* — 나중에 떼는 건 쉽지만 잘못 합친 걸 다시 쪼개는 건 번거로우니, 의심스러우면 *합쳐 두되 인터페이스+`internal` 경계는 미리 세워* 둔다.

---

## 정리 — *Ch.9 는 모듈에 이름을 주는 챕터다*

- Ch.9 의 가장 큰 통찰은 **"모듈은 다 같지 않다"** 는 것이다. 모놀리스에서 막 추출한 모듈들은 폴더처럼 균질해 보이지만, 의존성 그래프에서의 *높이* 가 각자의 운명을 결정한다 — 위(Feature)일수록 *의견 강하고·작고·자주 변하고·격리* 되며, 아래(Foundational)일수록 *generic 하고·크고·드물게 변하고·모두를 떠받친다*. Support 는 그 사이에서 *균형과 백본* 으로 산다.
- 이 분류가 *행정적 라벨링이 아닌* 이유는, 위치가 곧 **투자 전략** 을 지시하기 때문이다 — API 표면 크기, 테스트 깊이, 변경 조정 비용, 그리고 *가장 강한 개발자를 어디에 둘지* 까지. 자원은 언제나 *의존자가 가장 많은 곳* (foundational > support > feature)으로 흐른다.
- 세 가지 빠른 판별 질문으로 압축된다 — *사용자가 앱을 열어 이걸 하나?* → Feature. *다중 feature 가 이 능력을 필요로 하나?* → Support. *비즈니스 로직 거의 없는 기술 인프라인가?* → Foundational.

> Ch.7 이 *타이밍과 방향*, Ch.8 이 *정제*, Ch.9 가 *분류* 였다면 — 이제 이 깔끔한 삼분법이 *현실에서 깨지는 지점* 으로 간다. 한 모듈이 Feature 이면서 동시에 Support 인 **dual-role 모듈**, 그리고 모듈 간 상호작용을 조율하는 **orchestration 모듈** — Ch.10 "Modular Architecture in Practice" 가 그 실세계 도전을 다룬다.
