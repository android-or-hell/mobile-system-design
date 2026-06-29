# Scale 10·11 정리 — 모듈러 아키텍처의 실전, 그리고 시리즈를 닫는 철학

> Mobile System Design 시리즈 Book 3 (Scale) 의 열한 번째 챕터(실전)와 마지막 결론(Conclusion).
> 핵심 질문 — *"카테고리로 모듈에 이름을 줬다. 그런데 현실의 모듈은 깔끔한 삼분법에 안 들어맞는다 — Feature이면서 Support인 모듈, 옆으로 통신해야 하는 모듈, '분리하면 좋다'는 직감으로 끼워 넣은 interface 모듈. 이 회색 지대를 어떻게 다루는가?"*

> *"경계가 명확하고 목적이 공유될 때, 시스템은 견디고 앱은 번성한다(Where boundaries are clear, and purpose is shared, the system endures, and the app thrives)."* — Ch.10 에피그래프

> **이 챕터의 성격** — Ch.10 은 *실전 트러블슈팅* 챕터다. Ch.9 가 준 깔끔한 Feature/Support/Foundational 삼분법이 현실에서 *어디서 깨지는지*, 그리고 그 깨짐을 *카테고리를 버리지 않고* 어떻게 봉합하는지를 다룬다. 일관된 처방은 하나다 — **복잡성은 사라지지 않는다, 다만 어디로 옮기느냐가 선택일 뿐이다.** Ch.11 (Conclusion)은 시리즈 3권 전체를 *하나의 철학* 으로 압축한다 — *간결하게 시작, 그 후 합성으로 개선*.

---

## Scale-10 · Modular Architecture in Practice (실전 챕터)

> Ch.7 이 *언제·무엇·어떻게* 모듈화할지, Ch.8 이 *모듈을 어떻게 성숙*시킬지, Ch.9 가 *모듈을 어떻게 분류*할지였다면 — Ch.10 은 분류가 현실의 모서리에 부딪히는 지점을 본다. *한 모듈이 두 카테고리를 겸할 때*, *모듈이 옆으로 의존하고 싶을 때*, *추상화로 분리하고 싶을 때* 무엇을 하는가.

Ch.10 이 다루는 것:

- feature와 support 둘 다인 *dual-role 모듈* 다루기
- 여러 아키텍처 레이어에 걸친 *복잡한 feature* 의 배치
- 조정 로직을 위로 또는 아래로 — *lateral dependency* 해결과 *orchestration 모듈*
- *interface 모듈* 이 가치를 더할 때 vs 불필요한 오버헤드만 더할 때
- interface 모듈이 복잡성을 *제거가 아니라 이동* 시킨다는 이해
- 별도 interface 모듈 없이 테스트 유연성을 주는 *하이브리드 접근*
- 기초 아키텍처 *swap* 의 트레이드오프

---

### § 10.1 Dual-role 모듈 — Feature이자 Support인 경우

> *어떤 모듈은 두 역할을 동시에 가진다.* `Payments` 는 사용자가 "결제를 끝내야 한다" 며 앱을 열 수 있으니 *feature* 다. 그러나 "코스 구독" 플로 안의 한 *단계* 이기도 하니 다른 feature 가 빌려 쓰는 *support* 다(SDK 인프라). Feature인가 Support인가? — **둘 다.**

#### § 10.1.1 도메인 분할 — 한 모듈, 두 도메인

> *모듈에 단일 도메인을 강요하는 법은 없다.* Payments 모듈을 *두 도메인으로 분할* 한다 — feature 도메인 `Payments` 와 support 도메인 `PaymentsSDK`(Figure 10.1).

- **PaymentsSDK = 결제의 brain.** Payments = 그 위에 얹은 *기본(opinionated) 구현*.
- *best of both worlds* — 모듈은 *하나로 유지* 하되 내부는 multi-domain. **Payments feature 는 PaymentsSDK 와 얽히지 않고, PaymentsSDK 는 feature 의 존재조차 모른다**(단방향 의존).

#### § 10.1.2 모듈 분할 — 더 강한 분리가 필요할 때

> *기본 도메인 분할로 부족할 때만* Payments / PaymentsSDK 를 별도 모듈로 쪼갠다 — Payments 가 PaymentsSDK 내부에 접근 못 하게 *강제* 해야 하거나, 둘 다 매우 커지거나, *두 팀이 각자 도메인을 맡을 때*(Figure 10.2).

- > **Note** — *대부분의 사용 사례에서 기본 도메인 분할이 90% 의 문제를 해결한다.* 모듈 분할은 *두 모듈이 의미 있는 기능을 담을 때만* 추천 — 몇 클래스를 위한 모듈은 "순수주의자"의 비실용적 선택이다.

---

### § 10.2 복잡한 Feature — 함께 둘까, 분배할까

> *복잡한 feature 를 빌드할 때의 근본 결정* — 한 모듈에 둘까, 자연스럽게 속하는 다른 모듈들에 분배할까? 예: 인증은 login UI · biometric · token 관리 · auth 엔드포인트 네트워크 요청 · keychain 저장으로 *여러 레이어에 걸친다*(Figure 10.3).

#### § 10.2.1 모멘텀 유지 — 함께 시작, 나중에 donate

> *HDD 챕터의 원칙을 적용* — 빌드 중에는 *함께 유지* 하되 명확한 도메인 경계와 API 분리를 둔다. 시간이 지나며 어떤 도메인은 자기 모듈로, 또는 다른 모듈의 일부로 간다. **"what if" 시나리오가 아니라 실제 결정에 기반해 추출.**

- *왜 함께 시작하나* — 새 `LoginView` 를 다른 팀 소유의 *UI Library 모듈* 에 처음부터 두려 하면, 승인 전에 *새 요구사항 목록* 을 받고 *모든 변경이 그들의 승인을 필요* 로 한다. 모멘텀이 죽는다.
- *권장 흐름* — 컴포넌트를 *자기 모듈에 두고 feature 를 전달*, 잠시 *영글게 두며* 버그를 제거한 뒤 *donate*. 강한 API 디자인·도메인 모델링으로 컴포넌트를 *독립적으로 셋업* 해 넘기면 *부서 전체 속도가 오른다*.
- > **all-or-nothing 이 아니다** — 90% 는 자기 모듈에, 그러나 Network 모듈에 auth 요청 지원을 확장(자기 모듈에 두는 게 더 번거로우므로)하는 식으로 *일부는 처음부터, 나머지는 나중에 donate*. 단 — *donate 단계는 잊히기 쉽다.*

---

### § 10.3 Lateral Dependencies — 위로 옮기거나 아래로 추출

> *같은 레벨의 두 모듈이 통신해야 하는 도전.* **Feature 모듈끼리 직접 의존하지 말 것.** 예 — Profile 이 user 의 posts 를 보여주고, Posts 가 author 의 profile 을 보여주면 *Profile → Posts*, *Posts → Profile* 의 **순환 의존성**(Figure 10.4).

#### § 10.3.1 모듈에서 lateral 의존은 더 치명적이다

> *단일 앱 안에서는 순환 의존이 "컴파일은 된다".* 하지만 *좋은 아이디어는 아니다* — 나중에 모듈 추출이 훨씬 어려워진다. **둘 다 모듈이 되면 순환 의존성으로 빌드가 실패** 한다(Figure 10.5) — 즉 모듈 경계가 lateral 의존을 *물리적으로 거의 불가능* 하게 만든다.

#### § 10.3.2 조정 로직은 위로, 의존성은 아래로

> *모듈이 sideways 통신이 필요하면 — 위로.* 조정 로직을 *두 모듈이 모두 보이는 더 높은 레벨* 로 올린다 — 보통 *앱*(Figure 10.6). **의존성은 아래로 단방향, peer 사이로 sideways 는 안 된다.**

#### § 10.3.3~10.3.5 Orchestration 모듈 — 조정 로직을 앱에서 빼내기

> *조정 로직이 상당해지면* 앱 대신 *전용 모듈* 로 뺀다(Figure 10.7~10.8). 앱이 glue 코드로 비대해져 feature 테스트가 느려지거나, *멀티 white-label 앱 / headless client* 같은 다중 타깃을 봉사할 때.

| orchestration 모듈의 성격 | 내용 |
|---|---|
| **정의** | *다른 모듈 간 상호작용 조정만 전담* 하는 모듈. 앱 레이어엔 너무 무겁지만 개별 feature/support 엔 속하지 않는 *복잡한 조정* 을 담는 *지휘자*. |
| **위치** | 보통 *feature 모듈 바로 위*. 조정에 집중, 도메인 로직은 적게. |
| **명명** | 까다롭다 — `ProfilePosts` 같은 기계적 이름보다 *더 fitting 한* `UserHub`. |
| **도입 시점** | *꼭 필요할 때만* — 4번 메서드 호출을 위해 모듈 전체를 도입하지 말 것. |

#### § 10.3.6 아래로 추출 — 공유 로직은 support 로

> *때로는 아래로.* Posts 가 *콘텐츠 moderation* 로직을 많이 갖고 있고 Profile 팀도 같은 로직이 필요하면 — Posts → Profile 의 feature-to-feature 의존(순환은 아니지만 *lateral*)이 생긴다(Figure 10.9). 이때 moderation 을 *support 모듈로 아래로 추출* 하면 lateral 의존이 사라진다(Figure 10.10).

> **두 방향의 처방** — *조정 로직은 위로(앱/orchestration), 공유 로직은 아래로(support)*. 이 둘로 계층을 더 정확하게 유지한다.

---

### § 10.4 Interface 모듈 — 추상화의 숨은 비용

> *다른 모듈의 의존성·디테일을 숨기려는 모듈 = Interface module.* 모바일 코드베이스에서 인기지만 — *그 복잡성에 값할 가치가 있는가?* 직감("decoupling = good")으로 도입하지만, **복잡성을 제거가 아니라 위로 이동** 시킨다.

#### § 10.4.1 왜 팀은 interface 모듈에 손을 뻗나

> 엔지니어들은 "관심사 분리!" 를 외치며 *"interface module 이 better"* 라 중얼거린다. Network/Persistence 가 깨지면 *모든 feature 팀이 지연* 된다는 두려움이 동기. Course/Marketplace 가 *interface 에 의존* 하고 실제 Network/Persistence 가 *구현을 제공* 한다(Figure 10.12). *decoupling 은 충분하다 — 그러나 진짜 더 나은가?*

#### § 10.4.2 복잡성의 이주(migration)

> *기만적으로 좋아 보이는 이유 — 중요한 차원을 빼먹기 때문이다 — 바로 복잡성.* (Figure 10.13)

- *복잡성이 위로 이동* — Course 가 더 이상 Network 를 직접 쓰지 않고, Marketplace 도 Persistence 를 직접 쓰지 않는다. *모두 interface 에 의존* → *wiring 이 다른 곳(앱)으로* 간다.
- *앱이 모든 것을 본다 → 앱이 orchestrator* 가 된다. 어떤 구현을 쓸지, 어떤 순서로, 다른 버전을 어느 feature 에 줄지를 앱이 결정한다.
- > **모순** — 목표는 *더 많은 decoupling 과 자율성* 이었다. 그러나 코드가 위로 — *공유되고 빌드가 느린 앱* 으로 — 이동했다. **정반대 효과.** Marketplace 개발자가 *Course 의 glue 코드를 우연히 바꿀* 수도 있다.
- > **Note** — Persistence 처럼 *leaf 모듈* 은 자기 안에 복잡성을 머물게 할 수 있다(아래로 더 의존할 곳이 없으므로).

#### § 10.4.3 public API 가 비대해진다

> *시간이 지나면 NetworkInterface 에 unique storage key 를 넘겨야 할 수 있다* — `cacheKey: ???`(Figure 10.14). 이 키는 *Persistence 개념* 인데, interface 끼리는 서로를 몰라야 하므로 **세 가지 나쁜 옵션** 만 남는다:

| 옵션 | 내용 | 대가 |
|---|---|---|
| **Option 1: decoupling 깨기** | `import PersistenceInterface` | *모듈 의존성이 다시 생긴다* — 분리한 의미 소멸 |
| **Option 2: 중복 추상화** | `NetworkCacheKey` 새 타입을 NetworkInterface 안에 | *PersistenceKey 개념 중복* + *번역 코드* 필요(불일치 버그 온상) |
| **Option 3: god-module** | 모든 interface 를 *한 모듈* 에 | 작은 팀(2~5명)엔 OK, *큰 팀엔 거대 병목·fragile* |

#### § 10.4.4 그 밖의 실용 문제

| 문제 | 증상 |
|---|---|
| **동기화 오버헤드** | interface 와 implementation 을 늘 맞춰야 함 |
| **navigation 마찰** | "Go to Definition" 이 *빈 protocol* 로 감 — 실제 코드가 아님 |
| **의존성 곱셈** | 앱과 sample 앱이 *모듈 쌍에 각각 의존* |
| **변경 증폭** | 파라미터 하나 추가에 interface · 모든 구현 · 앱 glue 코드까지 ripple |
| **가파른 학습 곡선** | 신규 개발자가 추상 인터페이스 + 구체 구현 + DI 시스템을 *모두* 이해해야 |

> **진짜 문제** — *interface 모듈은 가설적 미래 문제를 풀면서 구체적 현재 비용을 부과* 한다.

---

### § 10.5 기초 코드 Swap — 패러다임 문제

> *시나리오*: NoSQL document DB 를 쓰는데 "언젠가 교체할 수 있으니 인터페이스 뒤에 숨기자". 합리적으로 들리지만 — *실전에서는 인터페이스를 결국 그 database 개념에 맞춰 디자인* 하게 된다(flexible 스키마 · document 쿼리 가정).

- 나중에 relational DB 로 전환 시 *근본적으로 호환 안 되는 패러다임* 과 충돌 — nested JSON → foreign key flatten 테이블, flexible 스키마 → rigid column, document 쿼리 → multi-table JOIN, *트랜잭션·일관성 모델 완전히 다름*.
- *복잡한 어댑터로 SQL 을 document 모양 인터페이스에 욱여넣게* 되어 — **두 접근의 이점을 다 잃는다**(relational 을 document 처럼 위장).
- > **솔직한 질문** — *이런 근본 컴포넌트를 실제로 얼마나 자주 swap 하나?* 극히 드물다. 그리고 그런 변경이 필요할 땐 *구현 swap 이 아니라 — 전체 데이터 모델·쿼리 패턴·핵심 비즈니스 로직의 재설계* 다. **어차피 대부분 재작성.** → 미래의 swap 을 위해 오늘 인터페이스 비용을 내는 건 헛돈이다.

---

### § 10.6 오버헤드 없는 테스트 — 같은 모듈 안의 seam

> *"테스트와 swap 유연성을 위해 interface 모듈이 필요하지 않나?"* — interface 모듈이 *진짜 테스트 이점*(무거운 의존성 없는 빠른 단위 테스트, 쉬운 mocking, 깨끗한 격리)을 주는 건 *합법적 우려* 다. **그러나 같은 이점을 더 단순한 접근으로 얻을 수 있다.**

- *하이브리드 접근* — concrete `NetworkClient` 를 Network 모듈에 두고 `NetworkTransport` 라는 *seam(절취선)* 에 의존하게 한다. 앱이 *Production / Staging / Mock Transport* 를 주입(Figure 10.15).
- **별도 interface 모듈 없이, 앱 레벨로 glue 코드를 밀어 올리지 않고** swap·test 가 된다. *Sample 앱도 같은 MockTransport 를 재사용*. → *모든 networking 이 한 곳에, 한 팀 소유* 로 남는다.
- > **간결성 우선 최적화 → 필요할 때만 복잡성 추가.**

#### § 10.6.1 문제를 더 일찍 잡는다

> *분리는 진짜 깨짐을 막아주지 않는다.* mock 으로 격리해서 잘 돌아도, *실제 앱에서 진짜 Persistence 를 쓰면 버그가 드러난다.* 차이는 *언제 발견하느냐* 다 — **interface 모듈은 통합 문제를 앱에서 나중에 발견하게 만들고, 하이브리드 seam 은 feature 개발 중 일찍 잡게 한다.** 더 일찍 잡고 싶지 않은가?

---

### § 10.7 Interface 모듈이 의미 있을 때 — "문제적" 의존성

> *interface 모듈이 항상 나쁜 건 아니다 — 단지 default 로 시작하지 말 것.* **훌륭한 이유는 모듈이 "문제적(problematic)" 일 때다.**

- *예* — 다른 회사가 만든 `ThirdPartyAnalytics` SDK: 안 안정적이고, *singleton public API 라 테스트 어렵고*, *원격 의존성 20개* 를 끌고 온다. 이를 `AnalyticsInterface` 뒤에 두면 Course/Marketplace 가 *그 20개 의존성을 안 가져온다*(Figure 10.16). 앱이 "이 짜증 나는 모듈은 내가 처리할게, 너흰 인터페이스에 집중해" 라고 말하는 셈.
- *다른 사용 사례* — 빌드가 매우 느린 모듈, 의존 팀을 심하게 늦추는 모듈, *순환 의존성*.
- > **결론** — *모듈이 "문제적" 일 때, 또는 테스트 이점이 조정 비용보다 명백히 클 때* 만 interface 뒤에 숨겨라. (안정적 인터페이스 + 단일 팀이 조정 오버헤드 소유 + 빠른 단위 테스트 사이클에 의존하는 워크플로.)
- > **Note** — *문제적 모듈이 거대한 public API* 면 *거대한 인터페이스 미러* 도 유지해야 함을 유의.
- > **먼저 물어라** — interface 에 손 뻗기 전에 *"무엇이 이 의존성을 문제적으로 만드나, 그것을 대신 고칠 수 있나?"* **인터페이스 뒤에 숨기기보다 모듈 자체를 더 좋게 만들 수 있는 경우가 많다.** *Interface 모듈은 종종 code smell* — 필요할 수도 있지만 *더 나은 해법이 있는 근본 문제* 를 가리킨다.

---

### § 10.8 Ch.10 한눈에

| 주제 | 요점 |
|---|---|
| Dual-role 모듈 | 한 모듈이 *다중 카테고리*(feature+support)에 적합 가능. *모듈 안에서 도메인 분할* 이 기본 처방 → *둘 다 의미 있는 기능을 담을 때만* 별도 모듈로. |
| 복잡한 feature | *처음엔 함께* (명확한 경계+API 분리) → *투기적 재사용이 아닌 실제 사용 패턴* 으로 추출 → 안정될 때까지 ownership 유지하다 *공유 모듈에 donate*. |
| Lateral 의존 | sideways 의존 금지. 조정 로직은 *위로*(두 모듈이 보이는 레이어 → 앱 → 커지면 orchestration 모듈), 공유 로직은 *아래로*(support). |
| Interface 모듈 — 쓸 때 | *불안정 third-party / 느린 빌드 / 순환 의존* 을 wrap 하거나 *테스트 이점 > 조정 비용* 일 때만. 안 그러면 *code smell*. |
| Interface 모듈 — 비용 | 복잡성을 *제거가 아니라 위로 이동*(앱이 orchestrator·비대) · 동기화 오버헤드 · public API 비대(타입 중복 or decoupling 깨짐) · navigation 마찰 · 변경 증폭. |
| 하이브리드 | *단일 모듈 안에 인터페이스+concrete 구현* (Transport seam) → ceremony 없이 유연성 + *통합 문제 조기 발견*. |
| 추상화 가치 평가 | *다른 패러다임이 아니라 — 비슷한 개념의 다른 구현* 에만 추상화. 기초를 *실전에서 얼마나 자주 swap* 하나 물어라. *간결성 우선, 구체적 필요가 정당화할 때만 복잡성 추가.* |

---

## Scale-11 · Conclusion — 오래 가는 시스템을 빌드하기

> *모바일 시스템 디자인 여정의 끝.* feature debrief 부터 회사·커리어를 위한 *오래 가고 확장 가능한 아키텍처* 까지 — 많은 영역을 다뤘지만, 더 중요한 건 *체계적으로 모바일 개발을 사고하는 법* 을 배운 것이다.

### § 결론.1 우리가 걸어온 길

> 3권의 여정을 관통하는 *한 주제 — 간결하게 시작, 그 후 개선*. 모든 고급 패턴이 *직접적 구현으로 시작* 되어 *실제 문제를 해결하며* 진화했다.

| 권(Book) | 다룬 것 |
|---|---|
| **Fundamentals** | 홀리스틱 접근 — UI 로 바로 뛰어들기 전 *요구사항 발굴* 우선, 단단한 기초. *API 디자인, 의존성, singleton, UI 무관 아키텍처*, UI 무관 부분의 빠른 테스트. |
| **UI** | UI 아키텍처 심화 — 자연스럽게 합성되는 *재사용 컴포넌트*, 다른 컨텍스트에 적응하는 *self-sufficient 뷰*, 복잡성에 따라 확장되는 *내비게이션 플로*. |
| **Scale** | 확장 도전 — 단순 UI 라이브러리 → *토큰 기반 디자인 시스템*(다중 플랫폼 동기화), *모놀리스 모듈화*, 팀 협업을 위한 코드 조직, 독립 feature 개발 지원 아키텍처. |

### § 결론.2 멘탈 모델 뒤의 철학 — 합성(composition)

> *책의 모든 코드 예시와 아키텍처 결정 뒤의 더 깊은 철학*: **독립적으로 잘 작동하는 시스템을 빌드한 뒤, 복잡성과 유연성을 위해 합성한다.** *강한 최소 API 의 도메인* 을 디자인함으로써, 함께 더 큰 아키텍처가 되는 *서브시스템* 을 빌드.

- *전통적 모바일 개발은 끊임없는 전투처럼 느껴진다* — tight coupling, 일관성 없는 UI, 같은 feature 의 다중 개발자 머지 충돌에 맞선. *이 책의 멘탈 모델은 그런 문제가 애초에 발생하지 않거나 줄어드는 시스템* 을 보여준다.
- *복잡성이 작은 부분에 격리되면 추론이 쉬워진다.* 부분을 *추출·삭제·이동·ownership 변경* 하기 쉽다. 고전적 *"big ball of mud"* 를 막고, 코드베이스나 팀원과 *덜 싸운다*.

### § 결론.3 모바일만의 이야기가 아니다

> *솔직해질 시간* — 책은 모바일에 집중하지만 *대부분의 원칙은 플랫폼 경계를 초월* 한다.

- *DI 패턴은 데스크탑 앱에서, 도메인 모델링은 CLI 도구에, 모듈러 접근은 큰 코드베이스에* 적용된다. 재사용 뷰의 *명명 전술* 은 *치과 예약 시스템·Python CLI·Unity 게임* 어디든 통한다.
- *보편성은 의도적* — iOS/Android/Swift/Kotlin/SwiftUI/Compose 의 *어떤 버전이든 초월* 하게. *모바일은 이 개념을 배우는 훌륭한 렌즈* 일 뿐 — 그 제약(빠듯한 자원·다양한 기기·빠른 진화)이 *아키텍처 결정을 즉시 보이게* 하므로.
- > *가장 중요한 것 — 좋은 시스템 디자인은 패턴 암기가 아니다.* **문제를 더 깊은 수준에서 이해 + 빌드하면서 시스템을 유연하게 유지.**

### § 결론.4 다음으로 갈 곳 — 작은 실험부터

> 멘탈 모델은 *경직된 규칙이 아니라 — 자신의 컨텍스트에 적응할 수 있는 도구*. 모든 코드베이스는 *고유한 도전*(다루지 않은 edge case·팀 다이내믹·비즈니스 제약)을 갖는다 — 그건 *예상되고 건강한 일*.

- *현재 프로젝트의 작은 부분에 적용 시작* — *뷰 없이 어디까지 빌드 가능한가?*, 단일 뷰에서 재사용 컴포넌트 추출, tightly coupled 서비스에 DI 주입. *작은 실험이 더 큰 변경을 위한 자신감* 을 만든다.
- *경험이 쌓이면 직관 발달* — 언제 적용하고 언제 단순한 해법이 충분한지, *리팩터의 조기 경고 신호* 를 인식. 가장 중요하게 — *문제를 사후 해결이 아니라 예방* 하는 눈을 갖게 된다.

### § 결론.5 계속 빌드하라

> *모바일 풍경은 계속 진화* 한다 — 새 프레임워크, 플랫폼 변화, 다른 팀으로의 이동(iOS→Android, 그 반대도). *그러나 배운 근본 원칙은 적용 가능하게 유지* 된다. **특정 구문은 변해도 — 유지보수 가능·확장 가능·협업 가능 소프트웨어를 빌드하는 근본 문제는 일정하다.**

> *"이제 가서 주목할 만한 무언가를 빌드하세요(Now go build something remarkable)."*

---

## Android / Jetpack Compose 매핑

> Ch.10 의 dual-role · lateral · interface-vs-seam 논의는 Android 멀티모듈에서 *가장 첨예하게* 드러난다 — Gradle 이 의존 방향을 *컴파일 단계에서 강제* 하고, `internal` 이 카테고리 경계를 *공짜로 집행* 하기 때문이다. 아래는 Pillo(복약 앱: DrugBank·FDA·Pillo 다중 API 통합) 도메인으로 옮긴 매핑이다.

#### Dual-role = 두뇌는 `:core`, 화면은 `:feature` (§ 10.1)

책의 도메인 분할(Payments + PaymentsSDK)을 Android 로 옮기면 — **두뇌(SDK)는 공유 가능한 `:core`**, **화면(feature)은 `:feature`** 에 둔다. SDK 의 public API 만 노출하고 내부 구현은 `internal`.

```
              :app
          ┌────┴────┐
:feature:payments  :feature:course
          └────┬────┘ implementation
        :core:payments
        PaymentProcessor ◀ public   (Course 가 두뇌만 빌림)
        StripePayment... ◀ internal (밖에서 안 보임)

✅ feature→core    ❌ feature→feature
```

- **Android 철칙: feature → feature 의존 금지.** Course 가 `:feature:payments` 에 의존하면 규칙 위반(= "`:feature:payments` 가 이상하게 느껴지는" 이유). 두뇌를 `:core:payments` 로 빼면 `:feature:course → :core:payments`(feature→core, 허용)가 되어 해결.
- **소비자 구분** — *두뇌(SDK)는 여러 곳이 공유*, *화면(Payments UI)은 그 화면이 필요한 진입점만* 사용. Course 는 화면 안 쓰고 두뇌만 빌린다.
- **두 레이아웃** — NiA 평면형(`:core:payments` + `:feature:payments`, 카테고리가 최상위) vs feature 레이어형(`:payments:domain` + `:payments:ui`, dual-role 과 1:1 매핑이 깔끔하며 저자 의도에 더 가깝다). Pillo 예: `:core:reminders`(알림 엔진, UI 없음) ← `:feature:medication`·`:feature:reminders` 둘 다 의존.

#### 모듈 1→2단계 승격은 재설계가 아니라 *기계적 이사* (§ 10.1.2)

- 1단계(한 모듈 + 패키지 분할 + `internal`)에서 2단계(`:feature`+`:core` 분리)로의 전환 비용은 *일회성·기계적* 이다 — build.gradle 추가 + sdk 패키지 IDE refactor 이동 + 경계 넘던 `internal`→`public`. **호출부를 인터페이스(`PaymentProcessor`)에 대고 짰고 패키지명을 유지하면 import 조차 안 변한다.** 분리 시 뜨는 컴파일 에러는 혼란이 아니라 *"여기서 경계를 넘었다"* 는 체크리스트. → 전환 고통 ∝ 1단계에서 경계를 어긴 양.
- *"처음부터 2모듈"* 도 정당화 조건(외부 소비자 / 팀 분리 / `internal` 강제 / 충분한 규모 / 안정된 경계)이 맞으면 정답이다. 책의 1단계는 *조건이 아직 안 찼을 때* 모듈 비용을 미루는 절약책 — Android 는 `internal` 강제·빌드 성능 때문에 그 조건이 *조금 일찍* 온다.
- *데이터 위치* — "feature 에 두다 core 로 승격" 은 *feature-전용 두뇌* 에만. **범용 인프라(Retrofit/Room/DesignSystem)는 태생이 `:core`** 라 승격 대상이 아니다. NiA 는 여기서 더 나아가 *데이터 전부를 항상 core* 에 둬(`:core:network`+`:core:data`, feature 는 UI 만) *혼재를 아예 없애는* 균일성을 택하고 `:core:data` 비대를 감수한다 — 책의 수직 소유(feature 가 자기 데이터 품다 승격)와는 *다른 선택*.

#### Lateral 의존 = "이동한다 ≠ 상대를 안다" (§ 10.3)

`:feature:medication` 이 `:feature:interactions` 로 *가고 싶을 뿐* 그 화면 코드를 알 필요는 없다. 사이에 추상화를 끼운다.

- **패턴 A (route 계약)** — 각 feature 가 진입점을 route 로만 노출, `:app` 이 `NavGraph` 조립. 공유되는 건 화면 코드가 아니라 *route 타입* 뿐이며 그 정의만 `:core:navigation` 에.
- **패턴 B (Navigator 인터페이스 + Hilt DI)** — `interface InteractionsNavigator` 를 공유 모듈에, 구현은 `:feature:interactions` 의 `internal class`. Hilt 런타임 주입 → *컴파일 타임 화살표 `medication → interactions` 가 안 생긴다*(= § 10.3 "조정은 위로").
- **조정이 무거워지면 → orchestration 모듈**(§ 10.3.3). `:userhub` 같은 모듈을 *feature 바로 위* 에 두고 `:app` 이 얇아지게. **공유 로직이면 → 아래로 추출**(§ 10.3.6): moderation 을 `:support:moderation` 으로. 규칙 — *여러 feature 가 공유하는 순간 그건 더 이상 feature 가 아니다 → 아래로 내린다.* 단 *"비슷해 보인다 ≠ 공유해야 한다"* — 진짜 공통 추상이 확인될 때까진 *복제가 더 싸다*.

#### Interface 모듈 vs 모듈 내 seam — App 코드가 갈린다 (§ 10.4 / § 10.6)

같은 "swap·test 가능" 능력을 두 방식으로 얻는다. 차이는 *복잡성이 어디로 가느냐* 이고, Android 에선 그게 *재컴파일 모듈 수* 로 측정된다.

```
❌ Interface 모듈 (복잡성이 App으로)        ✅ 모듈 내 seam (복잡성이 모듈에)
   App  ← wiring/번역/분기 집중               App  ← 환경 "선택"만 (얇음)
    ├ wires NetworkImpl                       │ inject(Prod|Staging|Mock)
    ├ wires PersistenceImpl                   ▼
    └ 번역 cacheKey(lateral)            :core:network (한 팀 소유)
        ▲ implements                          ├ NetworkClient (concrete, public)
   NetworkInterface (빈 모듈)                  ├ NetworkTransport (seam)
   PersistenceInterface (빈 모듈)             ├ Production / Staging / Mock
        ▲ depends                             ▲ depends on NetworkTransport
   feature (Course/Marketplace)         feature (+ sample 앱이 Mock 재사용)
   모듈 6+ · 복잡성 ▲App(병목)           모듈 3 · 복잡성 ▼모듈(분산)
```

- feature 가 구현을 모르게 되면 *그 모른 만큼을 누군가 대신 알아야* 한다. Interface 모듈에선 그 책임이 *App(orchestrator)으로 누적*(구현 생성·lateral 번역·feature별 분기), seam 방식에선 *모듈 내부에 머문다*. → § 10.4 의 "decoupling 늘리려다 코드가 공유·느린 빌드 App 으로 올라가는 정반대 효과" 가 *바로 이 App 코드 차이* 로 드러난다.
- 핵심 구분: *"구현마다 모듈로 쪼개기"* 가 아니라 *"능력의 주인 모듈 한 곳에 추상화·구현·연결을 co-locate"*. Mock/Staging/Production 모두 `:core:network` 안에 두고 App 이 아니라 모듈이 소유. Mock 을 `main` 소스셋에 두면 sample·debug 빌드가 재사용, `test` 소스셋에 두면 단위 테스트 전용 — *어느 쪽이든 App 소유가 아니다*.

```kotlin
// ✅ seam — :core:network 가 자기 wiring 소유, :app 은 얇다
@Module @InstallIn(SingletonComponent::class)
object NetworkWiring {                                 // <-- :core:network 안 (app 아님)
    @Provides fun transport(ok: OkHttpClient): NetworkTransport = ProductionTransport(ok)
    @Provides fun client(t: NetworkTransport, c: NetworkCache) = NetworkClient(t, c)
}
@HiltAndroidApp class PilloApp : Application()         // :app 끝. wiring 코드 0

// 테스트/sample: 그 "한 점"만 교체
@TestInstallIn(components = [SingletonComponent::class], replaces = [NetworkWiring::class])
object FakeNetworkWiring { @Provides fun transport(): NetworkTransport = MockTransport() }
```

#### cacheKey 케이스 — § 10.4.3 "public API 비대" 의 실제 모습

오프라인 캐싱을 위해 요청에 `cacheKey` 가 필요해졌다. 이 키는 **Persistence(저장소) 개념** 이라 url 만으론 못 만들고 `keyFor(url)` 로만 나온다(테넌트 네임스페이스·스키마 버전·정규화 id 를 저장소가 소유). interface 모듈(설계 A)에선 § 10.4.3 의 *세 나쁜 옵션*(import / 타입 중복+번역 / god-module)만 남고, 무엇을 택하든 *App 이 조립·번역을 소유*(복잡성 위로).

> **자연 방향은 "아래로"** — Network 가 Persistence 키가 필요하면 *App 을 거치지 말고* `:core:network` 가 `:core:persistence` 를 **아래로(단방향) 직접 의존** 하고, 어댑터(`NetworkCache`)를 *모듈 내부* 에 둔다(설계 B). public `get(url)` 은 *시그니처조차 안 바뀌어* feature 도 App 도 불변.

```
설계 A: :network-api ABI 변경 → :network-impl·:feature×2·:app 재컴파일 = 5 모듈 (app=직렬 꼬리)
설계 B: :core:network 내부 변경(Client ABI 불변)   → :core:network 1 모듈, feature·app 스킵
```

| cacheKey 변경 시 | 설계 A (interface 모듈) | 설계 B (seam) |
|---|---|---|
| 재컴파일 모듈 | **5** | **1** |
| feature 재빌드 | 강제(공유 -api ABI) | 스킵 |
| :app·:sample | 바뀜(번역·wiring) | **불변** |
| 소유권 | `:network-api` @platform-team (교차팀 승인) | `:core:network` @network-team (단독 머지) |

> 요점 — 공유 능력(Persistence 키)이 필요하면 *"아래로(support 로) 의존"* 이 자연 방향인데, interface 모듈은 그걸 *peer 로 갈라* App 이 lateral 로 중개하게 만든다. 그 부자연스러운 방향이 *세 나쁜 옵션 + App·여러 팀·느린 빌드로 번지는 변경 증폭* 이다. seam 은 복잡성을 *주인 모듈 안에 가둬* App 을 지킨다.

#### Analytics interface — "문제적 모듈" 의 Android 판단 (§ 10.7)

§ 10.7 의 "불안정 third-party SDK 는 interface 뒤에" 는 Android 에서 *의존성 격리* 로 직결된다. `ThirdPartyAnalytics` 가 원격 의존성 20개를 끌고 오면, `:core:analytics` 가 `AnalyticsClient` interface 만 노출하고 third-party 를 `implementation`(전이 차단)으로 감추면 `:feature:*` 는 그 20개를 *클래스패스에서 안 본다*. 단 — *잘 디자인된 안정적 모듈에는 이 ceremony 가 순수 오버헤드* 이므로, 손 뻗기 전에 *"무엇이 문제적인가, 대신 고칠 수 있나"* 를 먼저 묻는다(§ 10.7 의 code-smell 경고).

---

## 정리 — *Ch.10 은 "복잡성은 사라지지 않는다" 는 챕터, Ch.11 은 "합성" 이라는 한 단어다*

- Ch.10 의 가장 큰 통찰은 **복잡성은 제거되지 않고 이동할 뿐** 이라는 것이다. dual-role 모듈에서는 도메인을 *분할* 해 복잡성을 한 모듈 안에 정돈하고, lateral 의존에서는 조정 로직을 *위로*·공유 로직을 *아래로* 보내 계층을 지키며, interface 모듈에서는 그 분리가 복잡성을 *앱으로 밀어 올린다는 비용* 을 직시한다. 그래서 default 는 *별도 interface 모듈이 아니라 모듈 내 seam* 이다 — 복잡성을 *주인 모듈 안에 가두는* 쪽이 앱을 얇게, 빌드를 빠르게, 변경 반경을 작게 유지한다.
- 한 줄 처방으로 압축하면 — *둘 다인 모듈은 도메인을 쪼개고(§10.1)*, *옆으로 가고 싶으면 위로 올리거나 아래로 내리고(§10.3)*, *추상화는 "비슷한 개념의 다른 구현" 에만 쓰되(§10.5) 그마저 모듈 내 seam 으로 충분하며(§10.6), 별도 interface 모듈은 "문제적" 의존성에만(§10.7)*. 그 바닥의 한 질문은 늘 같다 — **이 복잡성을 어디에 둘 것인가, 그리고 그 선택의 변경 반경은 얼마인가.**
- Ch.11 은 시리즈 3권 전체를 *하나의 철학* 으로 닫는다 — **간결하게 시작해, 강한 최소 API 의 작은 부품을 합성한다.** 이는 모바일만의 이야기가 아니라 *유지보수·확장·협업 가능한 소프트웨어* 의 보편 원칙이며, 멘탈 모델은 *규칙이 아니라 도구* 다. 구문은 진화해도 근본 문제는 일정하다.

> Fundamentals 가 *기초와 사고법*, UI 가 *재사용과 합성*, Scale 이 *확장과 모듈화* 였다 — 세 권을 관통한 단 하나의 문장은 *"독립적으로 잘 작동하는 것을 만들고, 그것들을 합성하라"* 이다. *Now go build something remarkable.*
