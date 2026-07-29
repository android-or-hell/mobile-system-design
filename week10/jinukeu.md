# 05 시스템 전반 테스팅 정리 — 얼마나 낮은 곳에서 mock할 것인가

> Mobile System Design 시리즈 Book 0 (Fundamentals) 의 다섯 번째 챕터.
> 핵심 질문 — *"테스트를 위해 무엇을 가짜로 바꿔야 하는가, 그리고 그 경계를 어디에 그어야 하는가?"*

---

## 도입 — 시계가 시간을 잘 가리키는지만 확인하면 되는데

에피그래프: *"시계가 시간을 잘 가리키는지만 확인하면 되는데 왜 모든 톱니바퀴를 테스트하는가?"*

지난 챕터(§4)에서 플레이스홀더를 계층 아래로 밀어내며 **실제 네트워크 요청 직전까지** 내려왔고, 저자는 거기서 멈췄다. NetworkClient와 관련된 진짜 어려운 작업이 네트워크 코드 자체가 아니라 **의존성 주입과 테스트**이기 때문이다. 이 챕터가 그 절반(테스트)을 맡는다.

저자는 시작부터 두 가지를 못 박는다. 첫째, **은총알은 없다** — 테스팅은 의견이 극도로 분분한 분야이고 단일 챕터가 "가장 좋은 하나의 방법"을 정의할 수 있다고 생각하는 것은 터무니없다. 둘째, 이 챕터는 **unit test에만 집중**한다. 매뉴얼 테스트·UI 테스트·통합 테스트 중 unit test가 지금 개발 단계에서 개발자가 취할 다음 걸음과 가장 가깝기 때문이다.

그 위에서 챕터 전체를 관통하는 한 문장이 제시된다 — **가능한 한 적게 mocking하면서 가능한 한 큰 코드 표면을 테스트한다.**

> Chapter 5에서 다루는 것: 스택의 얼마나 높은 곳에서 테스트할지 파악하기 · 작은 unit을 테스트할 때의 단점 · unit test로 시스템 전반 테스트 작성하기 · 테스트 중 boilerplate 다루기 · system-wide test의 이점과 단점

---

## § 5.1 덜 세밀하게 테스트하기 — 방향 설정

업계에서 흔히 듣는 조언은 **"많은 인터페이스·mock·stub을 써서 코드를 테스트 가능하게 만들라"** 이다. 저자는 모바일 앱을 위해 정반대 길을 택한다 — **가능한 한 적게 mocking하면서 가능한 한 큰 코드 표면을 테스트한다.** 작은 unit이 아니라 코드를 **시스템으로서** 테스트하는, 사람들이 흔히 system testing이라 부를 만한 접근이다.

이 방향이 약속하는 것은 세 가지다.

| 얻는 것 | 내용 |
|---|---|
| **시간** | 테스트를 작성하는 데 시간을 **덜** 쓴다 — 많은 개발자가 고마워할 만한 점 |
| **품질 보장** | 최종 목표는 **코드를 머지하기 전에** 시스템이 전체로서 함께 작동한다는 많은 품질 보장을 갖는 것 |
| **인터페이스 감소** | 부수 효과로 **인터페이스(protocol)가 더 적어지고**, 이는 코드베이스에 대해 추론하기를 (논쟁의 여지 있게) 더 쉽게 만든다 |

위치 감각도 함께 정해둔다 — 이 접근은 **unit test와 통합 테스트 사이의 미세한 균형**이며, 모바일 환경에 특히 잘 맞는다. 궁극적으로 원하는 것은 **가능한 한 적은 테스트를 쓰면서도 버그의 위험을 낮추는 것**이다.

> **읽을 때 주의할 지점** — "큰 표면을 테스트한다"를 "테스트를 더 많이/길게 쓴다"로 읽으면 이 절 전체가 뒤집힌다. 저자의 주장은 **적게 쓰고 넓게 덮자**는 것이다. 조립 코드가 길어지는 문제는 §5.6.1에서 **단점으로** 따로 다루며, 곧바로 완화책이 뒤따른다.

---

## § 5.2 피해 통제와 피해 예방 — 왜 하필 모바일인가

테스트 기법으로 들어가기 전에, 저자는 **모바일 릴리스 프로세스의 구조적 제약**을 먼저 깔아둔다. 이 절이 챕터 전체의 동기이기 때문이다.

**웹·백엔드와 다른 두 가지 제약.**

- **롤백이 없다.** 깨진 변경을 되돌리는 것은 웹에서는 일상이지만, 우리는 사람들의 앱을 "un-update"할 수 없다. **오직 앞으로만 갈 수 있다.**
- **세분화된 릴리스가 없다.** 웹 개발자는 `/profile` 엔드포인트 뒤 기능만 갱신하고 나머지는 그대로 둘 수 있다. 모바일은 **한 번에 하나의 빌드**를 제출하며 그 빌드에는 **모든 팀의 모든 기능**이 담긴다. 화면 하나를 배포하려 해도 새 바이너리를 제출해야 한다.

여기에 릴리스 준비 자체의 무게가 얹힌다 — production 빌드 신중히 만들기, 수많은 검사와 테스트, 문자열 로컬라이즈, 마케팅 부서와의 조율, 어떤 회사에서는 **빌드 검증에만 일주일**. 제출 후에는 심사를 기다리고, 거부되면 이의를 제기하거나 다시 제출한다. 릴리스 준비가 끝나도 staged rollout으로 천천히 내보내며 크래시 수를 지켜본다. 저자의 논평이 뼈아프다 — **"그것이, 슬프게도, 최선의 시나리오이다."**

**§ 5.2.1 Damage control — 사고가 난 뒤에 할 수 있는 일.** 큰 앱에서 릴리스를 막는 버그는 어느 각도에서든 온다. 우리 팀이 준비한 거대한 온보딩 플로우가, **다른 팀이 결제 화면을 깨뜨렸다는 이유로** 지연될 수 있다. 릴리스 후 큰 버그가 터지면 이제 damage-control 모드다.

- **staged rollout 중단** — 시간이 맞다면 가능하지만, 크래시 보고가 초기에 경보를 울리지 않으면 이미 늦다.
- **hot-fix** — 모바일에서 hot-fix는 **진정으로 'hot'하지 않다.** 빠르게 머지해 즉시 배포할 수 없고, 새 빌드를 만들어 신속 검토를 거쳐야 한다. **쉽게 최소 하루**를 잡아먹는다.
- **feature flag** — 백엔드로 구동되는 플래그 뒤에 클라이언트 코드를 두어 사고 시 원격으로 기능을 끌 수 있다. 매우 권장되지만 **만능 계획은 아니다** — 플래그가 항상 잘 구현·유지되는 것도 아니고(큰 앱은 플래그가 아주 많다), **앱이 플래그를 가져오기도 전에 시작 시점에 크래시하면** 무력하다.

그래서 동전의 다른 면이 필요하다 — **damage prevention**, 라이브로 가기 전에 품질을 보호하는 것. 이것이 이 챕터의 목표이며, 우리가 테스트에 대해 추론하는 방식을 바꾼다.

> **이 절이 챕터에서 하는 일** — §5.3~§5.6의 모든 트레이드오프 판단은 결국 여기로 되돌아온다. "테스트가 좀 느려도 괜찮은가?", "조립 코드가 길어져도 감수할 만한가?" 같은 질문의 답이 **"릴리스 후에는 고칠 방법이 사실상 없다"** 는 제약에서 나오기 때문이다.

---

## § 5.3 스택 상위에서 모킹하기 — 통상적 방식과 그 대가

**시나리오 설정.** 학생이 진행 상황 영상을 업로드해 튜터에게 보내는 `VideoClient` 기능을 상상한다. 기타를 배우는 학생이 연주 영상을 올려 분석을 받거나, 스페인어 학습자가 발음이 어려운 구절을 찍어 보내 **1:1 통화를 기다리지 않고 비동기로 도움을 받는** 식이다.

이 기능이 예제로 선택된 이유는 **복잡한 로직을 요구하고 여러 컴포넌트에 의존하기 때문**이다 — 하드 디스크에서 미디어 읽기, 비트레이트 변환, 부분 업로드 재개, 백그라운드 업로딩. 어느 계층에서 mock하느냐에 따라 결과가 갈리는 상황을 보여주기에 적합하다.

**§ 5.3.1 의존성 계층.** Video 도메인의 그래프는 3단이다(Figure 5.1).

```
VideoClient ──> VideoAPI ──> NetworkClient
     │
     └────────> VideoCompressor
```

이후 모든 논의가 이 그래프 위에서 진행된다.

**§ 5.3.2 어디서 격리할 것인가.** VideoClient를 unit test하기로 했고, production 서버로의 네트워크 호출은 원치 않으니 무언가를 mock해야 한다. 통상적 선택은 **바로 아래 계층인 VideoAPI**다(Figure 5.2). 이유는 솔직하다 — **편리하고, VideoClient를 격리된 unit으로 빠르고 쉽게 초기화·테스트할 수 있기 때문**이다. 저자는 이 편의를 부정하지 않는다. 다만 **대가가 따른다**고 예고하고, 먼저 이 방식이 어떻게 작동하는지 끝까지 보여준다.

**§ 5.3.3 mock 도입 — 그리고 첫 번째 청구서.** 교체를 가능하게 하려면 인터페이스가 필요하다. 그것을 `VideoAPIProtocol`이라 부르고, 이를 따르는 `MockVideoAPI`를 테스트용으로 만든다(Figure 5.3).

```swift
// Listing 5.2
protocol VideoAPIProtocol {
    func uploadVideo(data: Data) async throws
}

final class VideoClient {
    private let videoAPI: VideoAPIProtocol           // 인터페이스에 의존
    private let videoCompressor = VideoCompressor()  // 교체 대상이 아니라 자체 생성

    init(videoAPI: VideoAPIProtocol) {
        self.videoAPI = videoAPI
    }
}
```

> **이미 드러나는 단점** — *"VideoAPI를 미러링하기 때문에 그 인터페이스의 이름을 짓기가 더 어려워지고, 게으르게 `Protocol` 접미사를 붙이는데, 이는 흔히 보는 모습이다. 타입의 이름을 짓는 것도 이미 어려운데, 비슷한 아이디어에 두 개의 이름을 짓는 것은 말할 것도 없다."*

§4.4.6의 "미러 인터페이스는 코드 스멜"이 여기서 구체적 사례로 되돌아온다. 이름을 못 지어 접미사를 붙이는 순간, 그 추상화는 **설계상 필요해서가 아니라 테스트를 위해 태어났다**는 자백이 된다.

**§ 5.3.4 테스트 환경 — 이 방식이 주는 유일한 선물.** `MockVideoAPI`는 NetworkClient에 의존하지도, 실제로 업로드하지도 않는다. 호출 여부만 기록한다.

```swift
// Listing 5.5 · 5.6
final class MockVideoAPI: VideoAPIProtocol {
    var didUploadVideo = false
    func uploadVideo(data: Data) async throws { didUploadVideo = true }
}

let mockVideoAPI = MockVideoAPI()
let videoClient = VideoClient(videoAPI: mockVideoAPI)   // 준비 끝
```

저자가 이 절에서 이점으로 꼽는 것은 **정확히 하나**다 — *"우리의 setup 코드는 작다 — MockVideoAPI 인스턴스를 만들고 그 외에는 아무것도 하지 않으면 된다."*

한편 VideoClient는 여전히 VideoCompressor에 의존한다. 그래서 저자는 **"완전히 격리되어 있다"는 라벨이 회색 영역**이 된다고 인정하면서, VideoCompressor는 **외부 의존성을 요구하지 않으므로 mock할 필요 없는 본질적(intrinsic) 의존성**으로 간주한다.

**§ 5.3.5 단점 — 무엇이 테스트 스위트에서 사라졌는가.**

> *"VideoAPI와 VideoClient 사이의 통합이 우리 테스트 스위트에서 완전히 누락되어 있다!"*

VideoClient는 테스트에서 **항상 완벽하게 통제된 네트워킹 환경**을 갖는다. 두 클래스를 각각 잘 테스트할 수는 있지만, **둘이 맞물려 동작한다는 자신감은 어디서도 얻지 못한다.** 저자가 드는 예는 전부 **호출 관계의 결함**이다.

- VideoClient가 VideoAPI의 upload를 **두 번 호출**할 수도 있다
- **잘못된 스레드**에서 반환할 수도 있다
- **업로드가 완전히 끝나기 전에 success를 반환**할 수도 있다 — 큰 파일을 재개 가능한 청크로 나눠 올릴 때 매우 마주치기 쉬운 시나리오

여기서 중요한 것은 **시점**이다. 이런 버그는 처음에는 잘 나타나지 않는다 — 개발자가 새 기능이 잘 도는지 직접 확인하기 때문이다. 문제는 **시간이 지나 유지보수하는 동안 슬그머니 들어오고**, unit test가 그것을 잡지 못해 위험이 조용히 누적된다는 것이다.

그래서 통합에 대한 자신감을 얻으려면 **다른 전술**에 기대야 한다 — 매 릴리스마다 수동으로 검증하거나, 내부 빌드를 배포하고 누군가 버그를 잡아주길 바라거나, 실제 네트워크 호출을 하는 통합 테스트에 투자하거나. 전부 **많은 추가 작업**이고, 그것도 겨우 두 클래스에 대한 이야기다.

> **최종 귀결** — *"스택 상위에서 mock했기 때문에 통합이 의도대로 작동한다는 것을 보장하기가 더 어렵고 시간이 많이 걸리게 만들고 있다. 이는 테스트를 릴리스에 더 가깝게 — 출시 직전의 수동 검사 같은 — 밀어붙이는데, 이는 모바일 개발에 해롭다."*

§5.2가 왜 먼저 나왔는지가 여기서 드러난다. 검증이 릴리스 직전으로 밀린다는 것은, **롤백도 부분 배포도 불가능한 환경에서 문제를 가장 늦게 발견한다**는 뜻이다.

---

## § 5.4 스택 하위에서 모킹하기 — 경계를 아래로 내리기

이번에는 **가능한 한 낮게** mock한다. 스택을 보면 NetworkClient가 mock할 수 있는 가장 낮은 클래스로 보인다(Figure 5.4). 이 선택만으로도 구성이 달라진다 — **VideoClient 테스트가 이제 실제 VideoAPI에 의존하게 되어, unit test가 실제로 production에 출시하는 것과 더 가깝게 닮는다.**

다만 저자는 곧바로 **남는 사각지대**를 명시한다. VideoAPI는 여전히 **mock 변형에 의존하는 NetworkClient**와 맞물리므로, 그 경계의 통합에는 unit test 단계에서 못 잡는 잔존 버그가 있을 수 있다.

> **이 챕터의 일반 원리** — 어디서 mock하든 **그 경계 바로 위 한 겹의 통합은 항상 검증에서 빠진다.** mock 지점을 아래로 내릴수록 그 사각지대가 얇아질 뿐이다. §5.3.5(VideoClient↔VideoAPI가 통째로 빠짐)와 §5.4(VideoAPI↔NetworkClient만 남음)의 차이가 정확히 이것이다.

그리고 여기서 멈추지 않는다 — **클래스를 통째로 mock해야 한다는 법은 없다.** 더 분해하면 테스트 표면을 더 키울 수 있다.

**§ 5.4.1 가장 작은 mock 표면 찾기.** 애초에 NetworkClient를 mock하려던 이유로 돌아간다 — **unit test 중 서버로 실제 네트워크 호출을 하지 않게 보장하는 것**, 오직 그것뿐이다. 그런데 NetworkClient는 그 이상을 한다.

| NetworkClient의 구성 | 비중 | 테스트에서 |
|---|---|---|
| 토큰 관리 · 재시도 메커니즘 · 검증 · 요청 설정 | 대부분 | **production 코드 그대로 실행되어야 마땅함** |
| 네트워크 의존성을 사용해 실제 호출을 하는 부분 | 약 10% | 이것만 교체하면 충분 |

그래서 NetworkClient를 통째로 바꾸는 대신 **네트워크 의존성만** 뽑아낸다. iOS에서는 `URLSession`이 그 대상이며, 이를 외부 의존성으로 만들어 교체 가능하게 한다. 결과적으로 **VideoClient보다 세 계층 아래에서** mock하게 된다(Figure 5.5).

**§ 5.4.2 배선 — 인터페이스를 깊은 곳에 둔다.** `NetworkTransport`라는 인터페이스를 NetworkClient **아래에** 도입한다. 이전 인터페이스(`VideoAPIProtocol`)와 달리 이것은 **스택 훨씬 깊은 곳에 산다**(Figure 5.6).

```swift
// Listing 5.7 · 5.8
protocol NetworkTransport {
    func data(for request: URLRequest) async throws -> (Data, URLResponse)
}

final class NetworkClient {
    private let transport: NetworkTransport
    init(transport: NetworkTransport) { self.transport = transport }
}

extension URLSession: NetworkTransport {}   // 메서드가 그대로 미러링되므로 구현 불필요
```

이어서 VideoAPI가 NetworkClient를 주입받도록 바꾸고, VideoClient는 **`VideoAPIProtocol` 대신 구체 타입 `VideoAPI`를 받도록 되돌린다**(Listing 5.9 · 5.10). §5.3에서 만들었던 인터페이스와 MockVideoAPI가 **통째로 사라진다** — §5.1이 예고한 "인터페이스가 더 적어진다"가 실제로 일어나는 지점이다.

**§ 5.4.3 테스트 조립 — production과 딱 한 줄만 다르다.**

```swift
// Listing 5.11 (production)              // Listing 5.12 (test)
let urlSession = URLSession.shared        let mockURLSession = MockTransport()      // ← 이 줄만 다름
NetworkClient(transport: urlSession)      NetworkClient(transport: mockURLSession)
VideoAPI(networkClient: networkClient)    VideoAPI(networkClient: networkClient)
VideoClient(videoAPI: videoAPI)           VideoClient(videoAPI: videoAPI)
```

가장 아래 전송 계층 한 줄만 갈아끼우고 **위쪽 배선은 production과 완전히 동일**하다. 그 결과 VideoAPI와 VideoClient의 unit test에서 **실제 NetworkClient를 쓰게 되고, NetworkClient는 여전히 90%가 production 코드다.**

교체가 가능한 근거는 구조 변경 그 자체다 — NetworkClient가 **구체 타입이 아니라 인터페이스에 의존**하도록 바뀌었고, `MockTransport`와 `URLSession` **둘 다 그 인터페이스를 따르기** 때문이다.

**§ 5.4.4 트레이드오프.** 공짜는 아니다.

- **인터페이스 미러링 부담** — NetworkClient가 인터페이스에만 의존하고 더 이상 `URLSession`의 메서드에 직접 도달할 수 없으므로, **쓰고 싶은 메서드를 protocol에 일일이 미러링해야 한다.**
- **조립 코드 증가** — VideoClient를 세우는 데 이제 4줄이 필요하다. 이 boilerplate 때문에 개발자가 이 접근을 망설일 수 있다(§5.6.1에서 다룸).

여기서 저자의 **설계 판단** 하나가 눈에 띈다. iOS라면 `URLSession`이 `URLProtocol` 형태의 자체 mocking 기능을 제공하므로 `NetworkTransport`를 도입하지 않고도 같은 목적을 달성할 수 있다. 그럼에도 그 길을 택하지 않은 이유는 **챕터를 플랫폼 독립적으로 유지하기 위해서** — 모든 솔루션이 자체 mocking 기능을 제공하는 것은 아니기 때문이다.

---

## § 5.5 비싼 연산 모킹하기 — 인터페이스 대신 함수

mock할 이유는 네트워킹만이 아니다. **비디오 압축은 시간이 걸린다.** system-wide 방식에서 VideoClient는 테스트 중 항상 실제 VideoCompressor에 의존하게 되는데, **모든 테스트에서 실제로 압축을 돌리는 것은 말이 되지 않는다** — 테스트 실행 시간이 분에서 시간으로 늘어날 수 있다.

**§5.4.1과 같은 기법을 다시 적용한다.** VideoCompressor는 여러 일을 한다 — 큰 파일을 다루기 위한 디스크 캐싱, 멀티스레딩. 하지만 **가장 느린 부분은 압축 알고리즘 자체**, 즉 바이트를 새로운 바이트 집합으로 변환하는 함수다. 그것이 **다른 모든 것을 그대로 두면서 교체할 수 있는 가장 작은 컴포넌트**다(Figure 5.7).

**§ 5.5.1 이번에는 인터페이스가 아니라 closure.** 압축은 "데이터 입력, 데이터 출력"의 문제이고, 그것은 함수 정의 하나로 표현된다.

```swift
// Listing 5.13 ~ 5.16
let compressionAlgorithm: (Data) -> Data

final class VideoCompressor {
    private let algorithm: @escaping (Data) -> Data
    init(algorithm: @escaping (Data) -> Data) { self.algorithm = algorithm }

    func compress(videoData: Data) -> Data {
        return algorithm(videoData)     // 디스크 캐싱·멀티스레딩은 그대로 남아 있다
    }
}

// production                          // test — 받은 데이터를 그대로 돌려주는 빈 알고리즘
VideoCompressor(algorithm: realAlgo)   VideoCompressor(algorithm: { data in data })
```

**인터페이스를 만들지 않은 근거는 단순하다** — 교체 대상이 **데이터를 받아 데이터를 돌려주는 단일 함수뿐**이기 때문이다. 타입 전체를 추상화할 이유가 없을 때 함수 자체를 값으로 넘기면 충분하다.

**§ 5.5.2 결과와 운용 방식.** 이제 **아주 적게 mock하면서 전체 도메인의 90%를 테스트**한다. 코드를 머지하기 전에 더 많은 안전 보장을 얻고, 이후의 모든 테스트가 VideoClient가 올바르게 작동하는지 검증한다.

운용은 균형이다 — **일부 테스트에서만 "production" 알고리즘을 사용**해 모든 것이 의도대로 작동하는지 확인하고, 나머지 스위트에는 가벼운 알고리즘을 써서 **스위트가 빠르게 돌게** 한다.

---

## § 5.6 시스템 전반 테스트 개선하기 — 단점을 정면으로 다루기

저자는 여기서 **내성(introspection)** 을 한다. 격리 unit test가 업계 표준이라 배웠다면 실제 의존성과 함께 테스트하는 것이 나쁜 아이디어처럼 느껴질 수 있다 — 결국 **의존성을 mock하는 것이 훨씬 쉽다.** 그래서 이 절은 **적게 mock하며 테스트하는 것을 실제로 무엇이 막고 있는지 정의한 뒤, 각각을 완화할 계획을 세우는** 진단 목록으로 시작한다.

| # | 장벽 |
|---|---|
| 1 | 네트워킹·파일 저장 같은 **I/O를 제대로 테스트하기 어렵다** |
| 2 | 테스트 중 **실제 디스크를 쓰는 것은 더 느리다** |
| 3 | UI·생체 인식·백그라운드 전환 같은 **외부 세계는 테스트가 어렵거나 불가능하다** |
| 4 | 모든 것을 초기화해야 하므로 **테스트 설정이 더 어렵고 손이 많이 간다** |
| 5 | vendor의 closed-source SDK처럼 **우리가 소유하지 않은 코드**가 있다 |
| 6 | 비디오 인코딩·이미지 처리 같은 **비싼 연산**이 스위트를 느리게 만든다 |
| 7 | production 코드에 더 의존하므로 **컴파일과 실행이 더 느려진다** |

항목들의 공통 성격은 분명하다 — 전부 **실제 의존성을 그대로 쓸 때 부딪히는 현실적 비용**이다. 그리고 이 목록은 포기의 근거가 아니라 **완화 계획의 입력**이다.

**§ 5.6.1 boilerplate — 문제의 구조.** VideoClient 하나를 테스트하려면 이렇게 된다.

```swift
// Listing 5.17
let mockCompressionAlgorithm: (Data) -> Data = { data in data }
let videoCompressor = VideoCompressor(algorithm: mockCompressionAlgorithm)
let mockURLSession = MockTransport()
let networkClient = NetworkClient(transport: mockURLSession)
let videoAPI = VideoAPI(networkClient: networkClient)
let videoClient = VideoClient(videoAPI: videoAPI, videoCompressor: videoCompressor)
```

길어지는 **구조적 원인**은 의존성이 계층적이라는 데 있다 — **모든 의존성이 만들어진 뒤에야** 최상위 타입을 초기화할 수 있으므로, 스택 가장 아래부터 위로 차례차례 조립해야 한다. 초기화 전에 5줄이 붙고, 그것도 전체 앱의 작은 부분일 뿐이다. 앱 전체라면 **수십 줄**이 된다.

저자의 태도는 회피가 아니다 — *"한 가지를 테스트하기 위해 많은 의식을 행하는 것은 **정당한 지적**이다. 그러나 약간의 작업으로 필요한 노력을 줄일 수 있다."*

**§ 5.6.2 production 쪽 줄이기 — 기본값.** 각 initializer에 default 값을 준다.

```swift
// Listing 5.18 · 5.19 · 5.20
init(transport: NetworkTransport = URLSession.shared) { ... }
init(networkClient: NetworkClient = NetworkClient()) { ... }
init(algorithm: @escaping (Data) -> Data = Algorithm.highCompression) { ... }
init(videoAPI: VideoAPI = VideoAPI(), videoCompressor: VideoCompressor = VideoCompressor()) { ... }

let videoClient = VideoClient()   // 조립이 한 줄로
```

**대가는 암묵성이다.** *"production 코드가 암묵적이다 — 이러한 의존성에 대해 모르고 테스트 중 '그냥' `VideoClient()`를 만들면, 그것은 암묵적으로 production 서버와 통신할 것이다."* 또는 누군가 여러 클래스에 mock을 주입하다 **하나를 잊어버려** 테스트 스위트가 실제 네트워크 호출을 트리거할 수도 있다. 이 접근은 **팀의 일정한 인식**을 요구한다. 대안으로, 환경이 허용한다면 **이 initializer를 production 코드에만 제공**하는 선택지도 제시된다.

**§ 5.6.3 테스트 쪽 줄이기 — 중앙화된 기본값.** production 기본값은 테스트에서 쓸 수 없으므로 테스트 쪽 boilerplate는 그대로 남는다. 해법은 **테스트용 기본 인스턴스를 한 곳에 모아두는 것**이다.

```swift
// Listing 5.21
extension NetworkClient  { static let `default` = NetworkClient(transport: URLSession.shared) }
extension VideoAPI       { static let `default` = VideoAPI(networkClient: .default) }
extension VideoCompressor{ static let `default` = VideoCompressor(algorithm: Algorithm.simple) }
extension VideoClient    { static let `default` = VideoClient(videoAPI: .default, videoCompressor: .default) }

let videoClient = VideoClient.default   // 각 테스트는 이것만 가져다 쓴다
```

*"설정하기 쉬우며, 그 자체로 boilerplate로 간주될 수 있다. 그러나 모든 default 값을 포함하는 **중앙화된 위치**를 갖는 것은 나머지 테스트 스위트의 많은 boilerplate를 제거하는 데 도움이 된다."* 환경이 허락하면 코드 생성(매크로)이나 구성 가능한 static 함수로 더 다듬을 수 있다.

**§ 5.6.4 느린 테스트 — 무엇과 비교해 느린가.** "30개 클래스가 함께 도니 느리다"는 반론에 저자는 네 갈래로 답한다.

- **관점 전환** — 머지 후에야 이슈를 놓치는 **빠른 테스트를 많이** 갖는 것보다, **더 적지만 시스템 전체가 의도대로 동작함을 보증하는** 테스트를 갖는 쪽이 나을 수 있다.
- **타협** — 기본적으로 느리거나 어려운 부분은 mock하되, **모든 것이 실제 타입과 함께 도는 테스트도 유지**한다. 속도와 안전을 모두 얻는 방식이다.
- **전체 최적화** — 테스트가 느려져도 **릴리스 프로세스 전체는 빨라질 수 있다.** 수동 테스트 의존이 줄고, 이미 system-wide로 덮고 있으니 통합 테스트 의존도 줄기 때문이다. 게다가 VideoClient를 테스트하면 VideoAPI·NetworkClient·VideoCompressor가 **간접적으로 함께 테스트된다.**
- **분리 실행** — 일상 개발용 "빠른 mock" 스위트와, 추가 검사를 위한 "무거운" 스위트를 **야간 CI**로 나눠 돌린다.

**§ 5.6.5 파일시스템 — 피하지 말고 써보라.** 개발자들은 디스크가 덜 신뢰할 수 있고, 가득 찰 수 있고, 느리고, 정리가 필요하다는 이유로 파일시스템을 메모리 저장소로 대체한다. 이해는 되지만 **그 가치를 고려하라**는 것이 저자의 권고다. 로컬 디스크 캐싱 라이브러리를 쓴다면, **일부 플랫폼은 경로에 후행 슬래시를 자동으로 붙이고 일부는 그러지 않는다**는 사실을 아는 편이 낫다. 이런 차이는 "항상 그냥 작동하던" 코드가 갑자기 폴더 관련 이상한 버그를 뿜기 시작하는 원인이 된다. **테스트가 더 느릴 것이라고 지레 가정하지 말고**, 느리더라도 코드가 작동한다는 보증이 그 값을 할 수 있다.

**§ 5.6.6 테스트 불가능한 코드 — 현실 인정.** 이 책은 그린필드를 가정하지만 실생활은 다르다 — 상태와 I/O에 깊이 얽힌 코드, 레거시 의존, **모든 것이 모든 것에 의존하는 스파게티**. 그런 시나리오에서는 이 접근을 항상 취할 수 없다. 선택지는 둘이다.

1. **더 적은 인터페이스로 일부분만 외과적으로 mock**한다
2. **고통을 직면하고** 그 어려운 컴포넌트를 테스트 가능하게 만들어 이 챕터의 이점을 얻는다

그리고 실천적 조언으로 닫는다 — **새 기능을 만들 때는 원하는 대로 설계할 자율성이 크니, 그때 이 접근을 시도해 자신에게 맞는지 확인해보라.**

**§ 5.6.7 테스트와 코드 사이의 거리.** VideoClient를 테스트하면 VideoAPI·VideoCompressor·NetworkClient가 간접적으로 테스트된다. 그런데 **NetworkClient의 버그를 고칠 때 우리가 수정하는 것은 VideoAPI 쪽 테스트**다.

여기서 오해하기 쉬운 지점을 저자가 못 박는다 — *"이는 NetworkClient가 그 버그에 대해 **테스트되지 않은 것처럼 보이는** 약간의 거리를 만든다. **그러나 그것은 테스트되고 있다** — 단지 VideoAPI를 통해 간접적으로."*

문제는 검증 여부가 아니라 **테스트가 놓인 위치**다. *"클래스와 관련된 테스트가 그 클래스 근처에서 사용 가능하지 않다. 이는 간접적인 의존성을 만들고 **이식성(portability)을 해칠 수 있다.** VideoClient는 추출하지 않고 NetworkClient만 추출하고 싶다면, NetworkClient는 일부 관련 테스트를 잃는다."*

> **모듈 분리 시나리오로 옮기면** — 네트워킹 코드의 재시도 로직 버그를 잡는 회귀 테스트가 상위 클라이언트의 테스트 파일 안에 들어가 있다고 하자. 6개월 뒤 팀이 네트워킹을 별도 모듈로 추출하면, **코드는 새 모듈로 옮겨가지만 그것을 검증하던 테스트는 원래 모듈에 남는다.** 새 모듈은 테스트가 하나도 없는 상태로 출발한다. 코드는 이동하는데 검증 자산은 이동하지 않는다는 것 — 이것이 "이식성을 해친다"의 실체다.
>
> ```
> [분리 전]                        [분리 후 — 네트워크 모듈 추출]
> :video 모듈                      :video 모듈            :network 모듈 (신규)
> ├─ VideoClient                   ├─ VideoClient         ├─ NetworkClient
> ├─ VideoAPI                      ├─ VideoAPI            └─ NetworkTransport
> ├─ NetworkClient  ─────┐         └─ VideoClientTests       (테스트 0개!)
> └─ VideoClientTests ◄──┘            (재시도 로직 테스트가
>    (재시도 로직도 여기서             이 안에 파묻힌 채 남음)
>     같이 검증되고 있었음)
> ```

절의 마무리는 균형이다 — *"다른 경우에는 버그를 고치기 위해 깊이 파고들어야 하는 것을 피할 수 없다. 때로는 깊이 파고들어 unit을 격리해 테스트해야 한다. **system-wide 접근은 은총알이 아니다.** 당신에게 맞는 좋은 균형을 찾으려고 노력하라."* (테스트 거리는 다음 도메인 테스팅 챕터에서 더 깊이 다뤄진다.)

---

## § 5.7 단위(Unit)란 무엇인가, 도대체

"격리해서 테스트하지 않으면 unit test의 요점에 반하는 것 아닌가"라는 반문에 대한 답이다. 저자의 질문이 날카롭다 — ***"누가 이 소위 'unit'을 정의하는가? '이 코드는 unit이지만 다른 클래스의 메서드를 호출하면 아니다'라고 말하는 비밀 위원회가 있는가?"***

논증은 사다리를 타고 올라간다.

- 작은 **순수 함수** 하나 — unit이라 부를 수 있다
- 그 함수가 **다른 함수를 호출**한다면? 여전히 unit인가?
- 함수들을 **클래스의 메서드**로 묶으면 — 그 클래스를 unit으로 볼 수 있다
- 모든 것을 하는 **god 클래스**를 써도 — **뚱뚱한 unit일 뿐 여전히 단일 unit**이다
- struct를 enum으로 쪼개 파일에 펼쳐놓고 각각을 unit이라 부른 뒤, 그것들을 모아 더 큰 컴포넌트에 끼워 넣으면 — **앱 관점에서 그 컴포넌트가 하나의 unit**이다

결론은 **컨텍스트 의존성**이다. *"앱의 관점에서 Course는 하나의 unit이다. 그러나 Course의 관점에서 CourseService나 TodoList가 하나의 unit이다."* 소프트웨어는 컴포지션 위에 세워져 있고, 프로그램을 해체하면 어셈블리·비트·논리 게이트·전기 펄스까지 내려가는 **컴포지트된 함수의 계층**이다. 그러니 **임의의 수준에서 무언가가 unit이라고, 또는 아니라고 자신을 설득할 수 있다.**

> *"'unit'을 테스트하는 데 맹목적으로 집중하지 말고 unit을 정의하기 위해 임의의 선을 긋지 말자. 대신 **버그를 출시하지 않도록 보장하기 위해 무엇을 테스트해야 하는지** 생각하라. 그게 다이다."*

단일 책임 원칙 역시 주관적이라는 각주가 붙는다. **unit 격리에 종교적이지 말 것** — 회색 영역임을 인정하고 자신에게 가장 잘 맞는 unit을 쓰라는 것이 이 절의 요구다.

---

## § 5.8 챕터 요약

| 축 | 핵심 |
|---|---|
| **왜 이 방식인가** | 모바일에서 **릴리스와 damage control은 더 제한적이고 어렵다.** 항상 빠르게 릴리스할 수 없으므로 **damage prevention이 큰 역할**을 한다. 작은 unit을 격리해 테스트하면 **머지할 때까지 전체가 얼마나 잘 맞물리는지 알 수 없는 위험**이 생긴다. 더 작은 unit이 아닌 **시스템으로서 코드베이스를 테스트해 더 많은 품질 보장**을 얻을 수 있다 |
| **어떻게 mock하는가** | **스택에서 더 깊이 mock**해 코드를 시스템으로서 테스트하라. **상위에서 테스트해 하위 컴포넌트를 간접적으로** 테스트하라. 더 세밀한 통제가 필요하면 **인터페이스 대신 closure**를 쓰라. **"unit"은 공식적으로 정의되지 않으니** 자신에게 맞는 unit을 정하라 |
| **대가와 처방** | **boilerplate가 늘어난다** → default 매개변수·default 속성으로 최소화하라. **스위트가 느려지고 컴파일이 길어진다** → 느린 테스트를 별도로 또는 주기적으로 실행하라. **더 높은 추상화에서 테스트하므로 실제 코드와 테스트 코드 사이에 거리가 생긴다.** **일부 코드는 이 방식에 맞지 않아 어차피 mock해야 한다** — closed-source 소프트웨어가 그런 예다 |

> **요약이 릴리스 이야기로 시작하는 이유** — 이 챕터의 논지는 "도구를 바꾸자"가 아니라 **"unit test라는 도구는 그대로 두되 그 사정권을 시스템 전체로 넓히자"** 이다. 통합 테스트·UI 테스트·수동 검증은 이 챕터가 **줄이려는 대상**이지 대안이 아니다 — §5.3.5에서 그것들은 상위 mocking이 치르는 **대가**로 지목됐다.

---

## 나누고 싶은 이야기

### 1. Compose/Kotlin으로 옮기면 — `extension URLSession: NetworkTransport {}`가 안 되는 언어에서

- **§5.4.2의 `extension URLSession: NetworkTransport {}`는 Kotlin에 등가물이 없다.** Swift의 **retroactive conformance**(이미 존재하는 남의 타입을 나중에 내 프로토콜에 맞추는 것)는 Kotlin에 아예 없는 기능이다. 확장 함수는 멤버를 **추가**할 뿐 **인터페이스를 구현하게 만들지는 못한다.** 그래서 Kotlin에서는 어댑터를 한 겹 직접 써야 한다.

  ```kotlin
  interface NetworkTransport {
      suspend fun data(request: Request): Response
  }

  // Swift는 extension 한 줄이면 끝나지만, Kotlin은 감싸는 클래스가 필요하다
  class OkHttpTransport(private val client: OkHttpClient) : NetworkTransport {
      override suspend fun data(request: Request): Response = client.newCall(request).execute()
  }
  ```

  **원리는 전혀 손상되지 않는다** — "가장 낮은 전송 계층만 교체 가능하게 만든다"는 목표는 그대로이고, 다만 그 지점을 만드는 문법적 비용이 한 클래스만큼 더 든다. 이 차이 때문에 §5.4.2 코드가 유독 낯설게 읽힌다면, 그것은 개념을 놓친 것이 아니라 **언어 기능의 부재**를 감지한 것이다.

- **§5.4.4의 `URLProtocol` 논의 → 안드로이드에서는 MockWebServer.** 저자가 "iOS에는 URLSession 자체 mocking이 있지만 플랫폼 독립성을 위해 쓰지 않겠다"고 한 그 자리에, 안드로이드에는 **OkHttp의 `MockWebServer`** 와 **`Interceptor`** 가 있다. MockWebServer를 쓰면 `NetworkTransport` 인터페이스를 도입하지 않고도 **실제 OkHttp 스택 전체를 production 코드 그대로 태운 채** 응답만 가짜로 줄 수 있다 — §5.4.1의 "가장 작은 mock 표면"을 **인터페이스 추가 없이** 달성하는 셈이라, 어떤 의미로는 이 챕터의 이상에 더 가깝다. 다만 저자의 판단(플랫폼 특화 기능에 기대지 않고 일반 기법을 보여준다)이 왜 나왔는지도 함께 기억할 만하다.

- **§5.5의 closure 의존성 → Kotlin 함수 타입, `@escaping`은 불필요.** `(Data) -> Data`는 `(ByteArray) -> ByteArray`가 되고, Swift가 저장 시 요구하는 `@escaping` 표기는 Kotlin에 없다(함수 타입 프로퍼티 저장이 기본).

  ```kotlin
  class VideoCompressor(private val algorithm: (ByteArray) -> ByteArray) {
      fun compress(videoData: ByteArray): ByteArray = algorithm(videoData)
  }

  // test — 받은 것을 그대로 돌려주는 알고리즘
  VideoCompressor { data -> data }
  ```

  "교체 대상이 함수 하나면 인터페이스를 만들지 마라"는 §5.5.1의 판단은 Kotlin에서 더 자연스럽다. `fun interface`(SAM)까지 갈 것도 없이 함수 타입으로 충분하다.

- **§5.6.2의 default 값 → Kotlin 기본 인자, 그리고 같은 함정.** `class NetworkClient(private val transport: NetworkTransport = OkHttpTransport())` 처럼 그대로 옮겨진다. **암묵성의 위험도 똑같이 따라온다** — 여기에 Hilt를 섞으면 위험이 한 겹 더해진다. `@Inject` 생성자에 기본값을 두면 DI 그래프가 그 값을 쓰지 않지만, 테스트에서 수동으로 객체를 만들 때는 기본값이 살아나기 때문이다. **"테스트인 줄 알았는데 실서버를 때린다"** 는 저자의 경고가 그대로 재현될 수 있는 조합이다.

- **§5.6.3의 `static let default` → `companion object`.**

  ```kotlin
  // test source set에 두면 production 바이너리를 오염시키지 않는다 — 저자가 각주로 제안한 대안과 같은 효과
  fun defaultVideoClient(
      transport: NetworkTransport = FakeTransport(),
      algorithm: (ByteArray) -> ByteArray = { it },
  ) = VideoClient(VideoApi(NetworkClient(transport)), VideoCompressor(algorithm))
  ```

  기본 인자를 가진 **테스트 팩토리 함수**가 Kotlin에서는 `companion object` 상수보다 낫다. 대부분의 테스트는 인자 없이 부르고, 한 축만 바꿔야 하는 테스트는 그 인자만 넘기면 되기 때문이다.

- **§5.6.4의 스위트 분리 → Gradle이 잘하는 일.** JUnit5 `@Tag`, Gradle의 별도 `test` 태스크, 혹은 `src/slowTest` 소스셋으로 나누고 CI에서 nightly 잡으로 돌리는 구성이 그대로 대응한다. 안드로이드에서는 여기에 **`test/`(JVM) vs `androidTest/`(계측)** 라는 축이 하나 더 있어 분리가 오히려 쉽다.

- **§5.6.5의 파일시스템 → `TemporaryFolder`와 경로 차이.** JUnit의 `@get:Rule val tmp = TemporaryFolder()` 또는 `kotlin.io.path.createTempDirectory()`로 실제 디스크를 쓸 수 있다. 저자가 든 "플랫폼마다 후행 슬래시 처리가 다르다"는 예는 안드로이드에서 **`filesDir` vs `cacheDir` vs `externalFilesDir`, API 레벨별 scoped storage 동작 차이**로 번역된다 — 메모리 fake로는 절대 드러나지 않는 종류의 버그다.

- **§5.3.3의 `VideoAPIProtocol` = 안드로이드의 `XxxRepositoryImpl`.** §4에서 이미 붙었던 논쟁이 여기서 구체적 증거를 얻는다. `interface UserRepository` + `class UserRepositoryImpl`은 저자 기준으로 **미러 인터페이스**이며, 이름을 못 지어 `Impl`을 붙이는 순간 "테스트/DI 때문에 만든 추상화"라는 자백이 된다. 주목할 것은 §5.4에서 **`VideoAPIProtocol`이 실제로 삭제된다**는 사실이다 — mock 지점을 아래로 내리자 그 인터페이스의 **존재 이유 자체가 사라졌다.** "이 인터페이스는 설계상 필요한가, 아니면 mock 지점이 잘못된 곳에 있어서 필요한가"를 묻는 것이 이 챕터가 주는 실전 도구다.

### 2. 오늘의 멘탈 모델 업데이트 — 사각지대는 데이터가 아니라 연결에 생긴다

이 챕터를 하나로 관통하는 문장은 §5.4의 **"어디서 mock하든, 그 경계 바로 위 한 겹의 통합은 검증에서 빠진다"** 이다. 저자는 이것을 세 번 반복해서 보여준다.

| mock 지점 | 검증에서 빠지는 것 | 남는 사각지대의 두께 |
|---|---|---|
| VideoAPI (§5.3) | VideoClient ↔ VideoAPI **전체 통합** | 두껍다 — 도메인 로직 간 연결이 통째로 없다 |
| URLSession (§5.4) | VideoAPI ↔ (mock을 문 NetworkClient) | 얇다 — 전송 계층 한 겹만 |
| 압축 알고리즘 (§5.5) | 알고리즘 자체의 정확성 | 가장 얇다 — 캐싱·스레딩은 그대로 검증됨 |

**퀴즈에서 반복해 걸렸던 지점들도 결국 이 프레임에서 재구성된다.**

- **§5.3.5 · §5.4 본문 · §5.4.3 (세 번 연속 같은 방향으로 오답)** — 세 문제 모두 **"서버 응답 스키마/파싱 정합성"** 쪽을 골랐다. 그러나 이 챕터에서 mock이 만드는 구멍은 **데이터가 틀리는 문제가 아니다.** 저자가 든 예는 전부 호출 관계다 — upload를 두 번 호출한다, 잘못된 스레드로 반환한다, 완료 전에 success를 돌려준다. 스키마 문제는 애초에 **실제 서버 호출이 없는 unit test의 사정권 밖**이라 저자의 논의 대상이 아니다. **"mock은 데이터를 가짜로 만드는 게 아니라 연결을 끊는 것"** 으로 프레임을 바꾸면 세 문제가 한 번에 풀린다.
- **§5.3.4 (두 번 오답)** — "테스트 배선이 production과 같아진다"를 **상위** mocking의 이점으로 골랐는데, 그 서술이 성립하는 자리는 **§5.4.3(하위 mocking)** 이다. 상위에서 mock하면 배선은 오히려 production과 **달라진다** — 테스트에 VideoAPI도 NetworkClient도 등장하지 않기 때문이다. 상위 mocking이 주는 것은 오직 **작은 setup 코드** 하나뿐이고, 배선 동일성은 §5.3.5에서 **통합 누락**이라는 청구서로 돌아온다.
- **§5.1 · §5.8 (챕터의 방향 감각)** — 각각 "테스트 코드가 길어진다"와 "통합·UI 테스트 중심으로"를 골랐는데, 둘 다 챕터가 **반대하는** 방향이다. mock을 줄이면 **인터페이스가 줄고**(§5.4에서 `VideoAPIProtocol`이 실제로 삭제된다), 목표는 **적게 쓰면서 넓게 덮는 것**이다. 통합 테스트·UI 테스트·수동 검증은 §5.3.5가 **대가로 지목한 것들**이지 대안이 아니다.

§4가 "플레이스홀더를 아래로 밀어내는 절차"였다면, §5는 **mock 경계를 아래로 내리는 절차**다. 두 챕터의 동작이 같은 모양이라는 점이 흥미롭다 — 위 계층에서 가짜를 걷어내고 그 가짜를 한 칸 아래에 다시 세우면, 위 계층은 그만큼 진짜가 된다. §4에서는 그것이 "이 타입은 끝났다"를 뜻했고, §5에서는 "이 계층까지는 진짜로 검증됐다"를 뜻한다.

그리고 §5.7이 그 위에 마지막 못을 박는다. **"unit"에는 공식 정의가 없다.** 그러니 "이건 unit test가 아니다"라는 지적에 방어할 필요도 없고, 격리의 순수성을 지킬 이유도 없다. 물어야 할 질문은 하나다 — **버그를 출시하지 않으려면 무엇을 테스트해야 하는가.**
