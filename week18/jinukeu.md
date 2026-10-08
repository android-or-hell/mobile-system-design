# UI 04 실용적 UI 구현 정리: 거칠어도 되는 것과 거칠면 안 되는 것

> Mobile System Design 시리즈 Book 1 (Modular UI & UI Architectures) 의 네 번째 챕터.
> 핵심 질문: *원칙이 모두 세워진 뒤에, 무엇을 먼저 만들고 어느 수준에서 멈춘 채 다음 단계로 넘어갈 것인가?*

---

## 도입: 이론이 끝나고 구현이 시작된다

챕터는 이 장에서 얻을 것을 네 가지로 먼저 밝힌다.

> **(p.71)** *"이 챕터에서:"*
> - *"세 가지 구현 경로(top-down, bottom-up, holistic)와 빠른 UI 전달을 위해 각각을 언제 선택할지"*
> - *"프레임워크 변화에 견디는 인터페이스 주도 패턴으로 UI를 데이터에서 분리"*
> - *"속도 해킹: 거칠게 UI를 그리고, 재작성 없이 다듬기"*
> - *"선언형이든 명령형이든 프레임워크 전반에 걸친 한 사고방식으로 구현 접근하기"*
>
> *"**프레임워크는 왔다 가지만, 꾸준한 접근은 그 모두를 능가합니다.**"*

이 챕터는 번호가 붙은 새 원칙을 세우지 않는다. UI-03 정리 문서에서 확인했듯이 UI 원칙 목록은 12번에서 닫혔고, 이 챕터는 그 열두 원칙을 `CourseView`라는 실제 화면에 적용해 보는 챕터다. 형식도 직전 챕터와 정반대다.

> **(p.72)** *"이전 챕터와 달리, 이 챕터는 코드가 많습니다. 그 목표는 솔루션의 완전성과 구현 수준에서의 더 깊은 고려를 보여주는 것입니다."*
>
> **(p.72) Note**: *"따라할 수 있도록 소스 코드를 가져오는 것을 권장합니다. https://github.com/tjeerdintveen/mobile-system-design-code 를 참조하세요."*

저자는 독자가 무엇을 아는 상태에서 이 챕터를 읽는지도 가정해 둔다.

> **(p.71)** *"이 챕터는 여러분이 이미 정기적으로 UI를 구현한다고 가정합니다. 그 초점은 *어떻게*가 아니라 결정을 내릴 때 *왜*에 대한 근거에 있습니다. 이 챕터의 주요 초점은 UI를 *빠르게* 전달하는 법을 보여주는 것입니다. 종종 디테일에 집중하기 전에 무언가 작동하게 만드는 것이 더 유용하기 때문입니다."*
>
> *"대부분의 경우, UI를 구현하는 데 SwiftUI를 사용할 것입니다. 그러나 깨진 레코드처럼 들릴 위험을 무릅쓰고, SwiftUI나 UIKit, Flutter, Jetpack Compose를 사용하는지는 너무 중요하지 않습니다. 이 결정 뒤에 있는 *개념*을 이해하는 것이 더 중요합니다."*
>
> *"경쟁 우선순위를 균형 있게 다루는 법을 보게 될 것입니다: 속도 vs 완벽성, 결합 vs 유연성, 즉각적 필요 vs 장기 유지보수성."*

이 정리 문서도 같은 태도를 따른다. 책의 코드를 전부 옮기지 않고, 결정이 갈리는 지점의 코드만 골라 싣는다. 안드로이드 개발자가 읽는 문서이므로 SwiftUI 문법 자체보다 그 코드가 내리는 결정을 Compose 코드로 다시 쓰는 일에 비중을 둔다. 저자가 Jetpack Compose를 직접 이름으로 언급한 만큼, 이 챕터는 안드로이드에서도 그대로 적용되는 부분이 많다.

---

## 이 챕터가 내리는 결정 한눈에 보기

챕터는 `CourseView` 하나를 완성하는 과정에서 열 가지 결정을 차례로 내린다. 각 결정이 어느 절에 있고, 책이 어떤 근거를 댔는지 먼저 표로 정리해 둔다.

| 절 | 결정 | 책이 댄 근거 |
|---|---|---|
| §4.1 | 내비게이션 크롬은 빼고 화면 본체만 구현한다 | 구현 대상을 기능 뷰로 한정한다 |
| §4.2 | 서브뷰는 placeholder 구현으로 먼저 만들고, 외형은 거칠게 둔다 (mixed) | 화면을 먼저 작동하게 만드는 편이 픽셀 퍼펙트보다 중요하다 |
| §4.3 | `Course`는 밖에서 전달받는다. 로딩은 다음 챕터로 미룬다 | 어디서 로드했든 `Course` 모델로 작동해야 한다 |
| §4.4 | SwiftUI 어휘를 먼저 합의한다 | 개념이 코드 디테일보다 중요하다 |
| §4.5 | `@Binding var course: Course`로 받고 `body`에서 서브뷰를 조립한다 | 렌더링도 못 하면 로딩을 지원할 수 없다 |
| §4.6 | `body`가 복잡해지면 `private` 뷰 속성으로 뺀다 | `body`가 섹션을 높은 수준에서 보여 준다 |
| §4.7 | `SelectionElement` 프로토콜과 제네릭으로 `SelectionView`를 만든다 | 뷰가 `TodoItem`을 모른 채 재사용 가능해야 한다 |
| §4.8 | `SelectionView` 두 개를 합성하고, 제목과 리셋 버튼은 `CourseView`가 맡는다 | `SelectionView`를 단순하고 재사용 가능하게 유지한다 |
| §4.9 | `ScheduleView`는 옵셔널 `date` 하나로 두 상태를 표현한다 | 옵셔널 해제는 분기 지점에서 한 번만 한다 |
| §4.11 | 내비게이션 액션은 콘솔 출력 스텁으로 남긴다 | 모멘텀을 유지한다 |

표의 맨 오른쪽 칸을 읽으면 챕터 전체가 한 가지 태도로 정리된다. **지금 해결하지 않아도 되는 문제는 미루되, 나중에 되돌리기 어려운 결정은 미루지 않는다.** 로딩과 내비게이션, 스타일링은 미루고, 어떤 뷰가 무엇을 아는가는 처음부터 정한다.

---

## § 4.1 구현의 시작

> **(p.73)** *"시작하려면, 기억을 새롭게 하기 위해 디자인을 다시 가져옵시다."* (Figure 4.1)
>
> *"우리 목표는 이 UI와 앞서 정의한 관련 서브뷰(`TextButton`이나 `SelectionView` 등)를 구현하는 것입니다. 그 이유로, 우리는 *기능 뷰*로 분류한 이 화면 자체에 집중할 것입니다. 그것 주변의 *크롬*을 제외하고 말입니다."*
>
> **(p.74) Note**: *"UI에서 '크롬(chrome)'이라는 용어는 콘텐츠나 화면을 둘러싸는 사용자 인터페이스입니다. 모바일에서 탭바나 내비게이션 바 같은 것입니다."*

첫 문장부터 Ch3에서 세운 분류 어휘가 그대로 쓰인다. 구현 대상을 "기능 뷰"라고 부르는 순간, 이 뷰가 비즈니스 로직을 아는 유일한 자리라는 전제가 함께 따라온다. 구현 범위를 화면 본체로 한정했기 때문에, 화면 바깥으로 나가는 이동(내비게이션)은 §4.11에서 스텁으로 남겨 둘 수 있다. 두 결정은 서로 맞물려 있다.

---

## § 4.2 구현 접근 비교

Ch2에서 `CourseView`를 여러 서브뷰로 분해해 두었으므로, 구현에 쓸 부품의 목록은 이미 있다.

> **(p.74)** *"이전 챕터에서, 우리는 `CourseView`를 구성하는 데 필요한 서브뷰를 정의해 왔습니다. `CourseView`를 분해함으로써, 많은 뷰로 끝나며, 일부는 기능 자체 가까이 또는 UI 라이브러리에 살 수 있습니다."* (Figure 4.2)
>
> *"그러나 그것이 항상 우리가 뷰를 구현 *전에* 분해해야 한다는 의미는 아닙니다."*

분해와 구현의 순서는 별개의 문제라는 뜻이다. 저자는 순서를 정하는 방법으로 네 가지를 비교한다.

### § 4.2.1 top-down 접근

> **(p.75)** *"우리는 top-down 접근을 취할 수 있습니다. 거기서 먼저 `CourseView`를 구현하고, 그 다음 필요에 따라 서브뷰를 *추출*합니다."*
>
> *"특히 선언형 UI에서는, 뷰 코드를 자르고 붙여넣음으로써 뷰를 자체 서브뷰로 (해)분해할 수 있어 비교적 부담이 없습니다."*
>
> *"반대로, 명령형 코드에서는 더 큰 뷰에서 뷰를 추출하기가 일반적으로 훨씬 어렵습니다. 뷰를 자유롭게 자르고 붙여넣을 수 없기 때문입니다. 이 이유로, 명령형 UI를 위해서는 적어도 서브뷰를 *미리 정의*(반드시 구현하지는 않음)할 것을 권장합니다."*

단점은 결합이다.

> **(p.75)** *"top-down 접근의 한 함정은 뷰를 비즈니스 로직과 너무 단단히 결합시킬 수 있다는 것입니다. 예를 들어 `SelectionView`가 직접 `TodoItem`에 의존하게 만드는 것입니다. 뷰를 추출할 가능성을 따져 보기도 전에 모든 뷰 코드를 서로 얽어 놓게 됩니다."*
>
> *"그저 명심해야 할 것입니다. 강한 결합을 막을 강한 장벽이 없기 때문입니다. 별도 뷰가 아직 추출되지 않았기 때문입니다."*
>
> *"그래서 뷰를 *기능 뷰*, *뷰 컴포넌트*, *뷰 프리미티브*로 분류하면 그것들이 얼마나 많은 책임을 가져야 하는지 결정하는 데 도움이 됩니다. 이는 여러분이 우연히 뷰를 프리미티브에서 컴포넌트로, 또는 비즈니스 로직을 아는 기능 뷰로 승격/강등시키지 않는지 인지하게 합니다."*

마지막 문단이 Ch3와 이 챕터를 잇는 연결 고리다. Ch3의 분류는 모듈을 나눌 때 쓰는 명명법에 그치지 않고, **구현 도중에 뷰가 슬그머니 다른 분류로 넘어가는 것을 알아채는 점검 도구**로도 쓰인다. 분류는 코드가 아니라 의존으로 결정되므로, 코드를 자르고 붙이는 사이에 도메인 타입 하나가 매개변수로 끼어들기만 해도 프리미티브가 기능 뷰가 된다.

### § 4.2.2 bottom-up 접근

> **(p.75)** *"둘째, 우리는 bottom-up 접근을 취할 수 있습니다. 거기서 우리 UI 라이브러리의 *뷰 프리미티브* 같은 서브뷰를 `CourseView` *전에* 구현합니다. `CourseView`는 계층의 *최상*에 자리합니다."*
>
> *"코드에서 모든 서브뷰를 미리 정의하면 강한 장벽을 만들고 우연히 데이터를 결합하지 않도록 보장할 수 있습니다(예: `Course` 데이터를 작은 재사용 뷰에). 이는 같은 UI에서 코워커와 작업할 때 특히 유용한 접근입니다. 뷰 사이의 API 장벽을 만들기 때문입니다."*
>
> *"단점은 사전 작업이 더 번거롭다는 것입니다. 먼저 무언가를 만들고 서브뷰를 나중에 어떻게 추출할지 알아내는 top-down 접근과 대비됩니다."*
>
> *"우리는 `CourseView` 구현을 우선시하는 것과 반대로 디테일에 너무 집중할 수 있습니다. 우리 목표를 위해, `CourseView`를 기능적 상태로 만드는 것이 더 중요합니다. 우리가 구현하는 메인 화면이기 때문입니다. 그 서브뷰를 완전히 끝내는 것은 덜 중요합니다."*

top-down에는 장벽이 없고 bottom-up에는 장벽이 있다. 대신 bottom-up은 디테일에 시간을 쏟기 쉽다. 두 접근의 장단점이 정확히 맞물려 있다는 점이 다음 접근의 동기가 된다.

### § 4.2.3 holistic 접근

> **(p.76)** *"셋째, 우리는 이를 holistic하게 접근할 수 있습니다. 거기서 우리는 *서브뷰를 미리 만듭니다*. 그러나 단지 그것들을 위한 placeholder 구현, 직사각형이나 라벨 같은 것만 사용합니다. 그런 다음 placeholder를 포함하는 서브뷰를 사용해 `CourseView`의 실제 구현을 작성할 것입니다."*
>
> *"`CourseView`를 끝낸 후, 우리는 한 추상화 낮춰서 서브뷰의 placeholder 구현을 삭제하고 실제 구현으로 교체합니다."*
>
> *"이 접근은 필요한 모든 뷰를 설정하는 것과 `CourseView` 구현에 집중을 유지하는 것 사이의 미묘한 균형을 잡습니다."*

이 접근의 뿌리는 Book 0의 Holistic-Driven Development(HDD)다. `.docs/03-HDD-Plan-To-Code-정리.md`에서 본 것처럼, HDD는 `Course.placeholder` 같은 값으로 컴파일을 유지하면서 전체 구조를 먼저 확정하고 구현 디테일은 미루는 방법이었다. 이 챕터는 같은 기법을 데이터가 아니라 **뷰**에 적용한다. 서브뷰의 이름과 매개변수(API)는 먼저 확정하고, 내부 구현만 나중에 채운다.

### § 4.2.4 mixed 접근

> **(p.76)** *"앞서 언급한 모든 접근은 유효하고 유용합니다. 여러분에게 맞는 것을 사용하세요."*
>
> *"이 챕터에서, 우리는 mixed 접근을 가정할 것입니다. 서브뷰는 이미 만들어졌지만 holistic하게 placeholder 구현을 사용해 구현됩니다. 그래서 `CourseView`에 먼저 집중할 수 있습니다."*
>
> *"그러나 트위스트가 있습니다. 모든 UI는 *대략* 디자인처럼 보이겠지만, 정확한 마진, 폰트, 간격을 사용하지 않을 것입니다."*
>
> *"우리는 placeholder 뷰를 사용하지만, 이 placeholder들은 단순한 모양 이상입니다. 디자인처럼 대략 보일 것입니다. 이렇게 하면, 이 뷰들은 UI 디테일을 만지작거리는 데 많은 시간을 쓰지 않고도 UI 구현의 좋은 느낌을 주기에 충분히 보일 것입니다."*
>
> *"UI를 거칠게 구현하는 이유는 테두리, 마진, 화려한 그림자에 집중하는 것과 반대로, 이 화면이 먼저 작동하게 만드는 것이 더 중요하기 때문입니다. 그렇게 하면 모든 것을 '픽셀-퍼펙트'로 만드는 데 산만해질 것입니다."*
>
> *"거친 구현을 사용하면 `CourseView`를 구현하는 데 *holistically*하게 집중할 수 있습니다. 우리는 뷰를 결합하고, 데이터에 연결하고, 적절한 기능성으로 이 화면을 우리 앱에 끼우는 데 시간을 집중할 것입니다."*

네 접근을 한 표로 비교하면 다음과 같다.

| 접근 | 먼저 만드는 것 | 서브뷰 사이의 경계 | 책이 꼽은 위험 | 책이 꼽은 적합한 상황 |
|---|---|---|---|---|
| **top-down** | `CourseView` | 추출하기 전까지 없다 | 뷰와 비즈니스 로직이 우연히 단단히 결합한다 | 선언형 UI에서 서브뷰를 추출할 때 |
| **bottom-up** | 서브뷰 (프리미티브·컴포넌트) | 코드로 먼저 정의되어 강하다 | 사전 작업이 번거롭고 디테일에 쏠린다 | 동료와 같은 UI를 나누어 작업할 때 |
| **holistic** | 서브뷰의 placeholder 구현, 그다음 `CourseView` | placeholder의 API가 경계가 된다 | 책이 따로 꼽지 않는다 | 필요한 뷰의 설정과 `CourseView`에 대한 집중을 함께 얻을 때 |
| **mixed** | holistic에 "대략 디자인처럼 보이는 외형"을 더한 것 | holistic과 같다 | 외형 디테일에 시간을 쓰는 것 | 이 챕터 |

> **읽을 때 주의할 지점**: mixed 접근에서 "거칠게"라는 말이 가리키는 범위를 정확히 읽어야 한다. 책이 거칠게 두겠다고 한 것은 **마진, 폰트, 간격 같은 시각적 외형**이다. 반대로 이 챕터의 코드가 거칠게 두지 않은 것은 뷰 사이의 **의존 구조**다. §4.7에서 보게 되듯 `SelectionView`는 제네릭과 바인딩까지 갖춘 온전한 API로 구현되고, `ScheduleView`는 `Course`를 모른 채 `Date?` 하나만 받는다. 즉 mixed 접근에서 낮춘 것은 외형의 정밀도이고, 낮추지 않은 것은 경계의 정확도다. 외형은 나중에 다듬어도 다른 코드가 깨지지 않지만, 경계는 나중에 고치려면 호출부까지 모두 바꿔야 한다. 그래서 이 구분이 "무엇을 미루고 무엇을 미루지 않는가"의 기준이 된다.

> **(p.76) Note**: *"책 후반에서, 우리는 컬러 시스템 정의에 *깊이* 들어갈 것입니다. 컬러를 정의하는 법에 대한 더 많은 정보는 'UI Library Fundamentals, Part I: Typography and Colors' 챕터를 확인하세요."*
>
> **(p.76)** *"UI 코드 안에서, `.brandingPurple` 같은 컬러 표기를 발견하실 것입니다. 이 컬러들은 코드 *외부*에 자산 라이브러리에서 프로젝트에 추가됩니다. 우리 에디터인 Xcode는 `BrandingPurple` 같은 컬러를 컴파일 타임에 체크되는 `.brandingPurple` 같은 속성으로 이름을 바꿀 것입니다."* (Figure 4.3)

색상이 코드 밖의 자산으로 정의되고 컴파일 타임에 이름이 검사된다는 설명은 §4.5 이후의 코드를 읽을 때 필요한 배경이다. 색 이름이 `.brandingPurple`처럼 역할 이름이라는 점은 Ch2의 원칙 6(스타일링으로 이름 짓지 말라)과 같은 방향이다.

---

## § 4.3 데이터는 전달받는다

> **(p.77)** *"이 챕터에서, `CourseView`가 *전달받은* 코스를 지원하게 만들 것입니다. `CourseView`가 코스 자체를 로드하는 것과 반대로 말입니다. 이는 `CourseView`가 어디서 로드되었는지에 관계없이 `Course` 모델로 작업해야 하기 때문이며, 그래서 단순히 그것을 전달하는 것을 우선시합니다."*
>
> *"모멘텀을 유지하기 위해, 아바타가 하드코딩된 방식으로 로드되도록 보장할 것이며, 지금은 네트워크 로딩을 옆으로 비켜 갈 것입니다."*
>
> *"화면이 `Course` 모델로 작동한 후, 그것을 *어디에서* 로드할지 알아낼 수 있습니다. 다음 챕터에서, 이 UI를 자기 충족적으로 만들어 자체 로드할 수 있게 하는 다중 방법을 탐구할 것입니다."*

Ch3 §3.7은 `CourseView`가 `CourseService`에서 `Course`를 **어떻게** 받아올지 아직 정하지 않았다고 말했다. 이 절은 그 질문에 답하지 않고 순서를 정한다. 먼저 받은 `Course`로 그릴 수 있게 만들고, 어디서 가져올지는 그다음에 정한다. 이 순서에는 실질적인 이점이 있다. 화면이 전달받은 값만으로 작동하면 데이터 출처가 서버든 캐시든 샘플 값이든 화면 코드가 달라지지 않는다. HDD에서 만든 `Course.placeholder`를 그대로 넣어 화면을 그려 볼 수 있다는 뜻이기도 하다.

모델 로직에 관해서는 한 가지 전제가 붙는다.

> **(p.77)** *"Holistic-Driven 챕터에서 우리가 모델 로직의 *모든* 측면을 완전히 완성하지 않았다는 점에 주목할 가치가 있습니다. 특히 todo 리스트의 정렬과 필터링에 관해서는 의도적으로 연습으로 남겨두었습니다."*
>
> *"그럼에도 불구하고, 이 챕터에서 우리는 디자인과 정렬되도록 모델 로직이 완료된 것처럼 진행할 것입니다. 'lorem ipsum' 같은 placeholder나 todo 항목 관련 이상한 동작을 피합니다."*

HDD 정리 문서의 `course.schedule.weekly` / `.daily`가 날짜 필터링 없이 `self`를 그대로 돌려주던 placeholder였다는 설명과 같은 이야기다. 이 챕터는 그 부분이 완성된 것으로 간주하고 UI에 집중한다.

---

## § 4.4 SwiftUI 크래시 코스

> **(p.78)** *"이 책은 SwiftUI 튜토리얼이 아닙니다. 그러나 우리가 같은 언어로 말하고 있는지 확인하기 위해, 우리가 사용할 중요한 SwiftUI 요소를 설명하기 위한 작은 크래시 코스를 가져 봅시다."*
>
> *"일상 업무에서 SwiftUI를 사용하지 않는다면, 구체사항에 대해 너무 걱정하지 마세요. 우리 접근 뒤에 있는 개념이 코드 디테일보다 더 중요합니다."*

이 절이 나열하는 요소 가운데 개념으로 이해해야 하는 것은 상태 관련 둘뿐이다. 레이아웃 뷰(`ScrollView`, `VStack`, `HStack`, `ZStack`, `Group`, `Spacer`, `Padding`, `Divider`)는 Compose의 `Column`, `Row`, `Box` 등에 대응하므로 뒤의 "안드로이드로 옮기면"에서 표로 정리한다.

> **(p.78)** *"`@State`는 뷰에 속한 데이터 조각을 선언합니다. 그것이 변하면, SwiftUI는 자동으로 뷰를 다시 그립니다. UI 업데이트를 트리거하는 로컬 변수처럼 생각하세요."*
>
> *"`@Binding`은 다른 곳(종종 부모 뷰나 observable 모델)에 소유된 데이터에 양방향 연결을 만듭니다. 그것에 쓰면 그 진실 공급원을 업데이트하고, 그 상태를 읽는 뷰들이 새로고침됩니다."*

`@State`는 **뷰가 소유한** 상태이고, `@Binding`은 **다른 곳이 소유한** 상태에 대한 읽기·쓰기 통로다. 이 챕터의 코드는 거의 전부 `@Binding`으로 쓰인다. 뷰들이 상태를 소유하지 않고 위에서 받아 쓰는 구조이기 때문이다. 이 구분은 Compose에서 상태 호이스팅을 이해하는 방식과 직접 이어진다.

> **(p.78)** *"*actions*라고 불리는 것도 보게 될 것입니다. 이들은 사용자가 버튼을 누를 때 호출되는 메서드입니다. 액션은 이 챕터 후반에 집중할 것입니다. 먼저 화면에 무언가 렌더링하는지 확인해야 하기 때문입니다."*

이 절은 앞으로 쓸 커스텀 뷰 다섯 개도 소개한다. 여기에 `SelectionView`와 `CourseView`를 더해, 각 뷰의 분류와 위치를 한 표로 모아 두면 이후의 코드를 읽기 쉽다.

| 뷰 | 하는 일 | Ch3 분류 | 위치 |
|---|---|---|---|
| `ThumbDescriptionView` | 강사의 아바타, 이름, 핸들 | 뷰 프리미티브 | UI 라이브러리 |
| `SubduedButton` | 강사에게 메시지를 보내는 작은 버튼 | 뷰 프리미티브 | UI 라이브러리 |
| `TextButton` | 텍스트 기반 버튼 | 뷰 프리미티브 | UI 라이브러리 |
| `CalloutView` | 강사의 긍정적 강화가 담긴 말풍선 | 뷰 프리미티브 | UI 라이브러리 |
| `ScheduleView` | 다음 1:1 일정 | 뷰 프리미티브 | Course 기능 안 (원칙 10) |
| `SelectionView` | 선택 가능한 항목 목록 | 뷰 컴포넌트 | UI 라이브러리 |
| `CourseView` | 화면 본체 | 기능 뷰 | Course 기능 |

> **읽을 때 주의할 지점**: Ch2와 Ch3의 도해에서 이 역할의 뷰는 `ImageLabelView`라는 이름이었는데, 이 챕터의 코드에서는 `ThumbDescriptionView`로 나온다. `CalloutView`도 도해에서는 `Callout`이었다. 매개변수의 모양(이미지, 제목, 설명)이 같으므로 같은 역할의 뷰로 읽으면 되고, 책은 이름이 바뀐 이유를 따로 설명하지 않는다. 이 정리 문서는 코드에 나온 이름을 그대로 쓴다.

---

## § 4.5 CourseView 첫 구현

> **(p.79)** *"다가오는 `CourseView` 구현을 보면, todo 리스트를 제외한 앞서 언급한 모든 뷰를 볼 수 있습니다. 더 복잡하므로 마지막을 위해 남겨둘 것입니다."*
>
> *"우리는 SwiftUI에서 `@Binding var course: Course`를 사용합니다. `@Binding` 키워드는 *외부* 소스가 `Course`를 이 뷰에 전달할 것임을 함의합니다. 바인딩이므로, 우리가 `CourseView`의 `Course` 모델을 업데이트하면, 이 변경은 `CourseView`*뿐 아니라* 그것의 소유자(부모)에서도 `Course`를 변형시킵니다."*
>
> *"또는, `Course`를 반환하는 `CourseService`를 전달할 수 있습니다. 그러면 외부 소스가 대신 `CourseService`를 전달합니다. 그러나 우리 우선순위는 지금 어떻게 검색되든 `Course` 인스턴스로 이 화면이 작동하게 만드는 것입니다. 코스를 렌더링조차 못한다면 코스 로딩을 지원할 수 없습니다. 그래서 먼저 우리가 *전달하는* `Course`를 지원하는 데 집중합시다."*

```swift
struct CourseView: View {
    @Binding var course: Course

    var body: some View {
        ScrollView {
            VStack {
                // Tutor info, laid out horizontally.
                HStack(alignment: .top) {
                    ThumbDescriptionView(imageName: "Avatar",
                                         title: course.tutor.displayName,
                                         description: course.tutor.handle)
                    Spacer()
                    SubduedButton(action: openMessageInbox, label: "Message")
                }.padding([.horizontal, .top], 20)

                // Only show callout when there is a message from the tutor
                if case let .message(message) = course.tutor.callOut {
                    CalloutView(message: message, action: dismissCallout)
                        .padding(.vertical, 10)
                }

                Divider().padding(.horizontal, 20)

                // The scheduled info for a 1-on-1 call
                ScheduleView(date: course.calendarEvent?.date,
                             actionJoinCall: joinCall,
                             actionReschedule: openScheduler)
                    .padding(.vertical, 10)
            }
        }
    }
    // .. Rest is omitted
}
```
*Listing 4.1: CourseView 구현 (주석 일부 생략)*

> **(p.80) Note**: *"SwiftUI에서, `some View`는 '이 함수는 뷰를 반환하지만, 정확한 타입은 숨겨져 있다'는 의미입니다. SwiftUI가 다양한 레이아웃 구성이 그들의 구체 타입을 노출하지 않고 모두 '뷰를 반환'한다고만 말할 수 있도록 이를 필요로 합니다."*

이 코드에서 Ch3의 원칙들이 어떻게 나타나는지를 읽어야 한다. `CourseView`는 `Course` 모델을 아는 유일한 뷰이고, 서브뷰에는 도메인 타입 대신 **원시 값과 클로저**만 넘긴다. 서브뷰마다 무엇을 받고, `CourseView`가 그 값을 어떻게 만들어 내는지를 표로 정리하면 다음과 같다.

| 서브뷰 | 받는 값 | `CourseView`가 넘기는 식 |
|---|---|---|
| `ThumbDescriptionView` | 이미지 이름, 제목, 설명 (문자열) | `"Avatar"`, `course.tutor.displayName`, `course.tutor.handle` |
| `SubduedButton` | 라벨, 액션 | `"Message"`, `openMessageInbox` |
| `CalloutView` | 메시지(문자열), 액션 | `if case let .message(message) = course.tutor.callOut`로 꺼낸 문자열, `dismissCallout` |
| `ScheduleView` | `Date?`, 클로저 둘 | `course.calendarEvent?.date`, `joinCall`, `openScheduler` |
| `TextButton` | 라벨, 액션 | `"Reset all"`, `resetAllTodoItems` (§4.8) |
| `SelectionView` | 요소 배열(바인딩), 액션 | `$course.schedule.weekly` 또는 `.daily`, `openDetails(todoItem:)` (§4.8) |

표에서 확인되는 사실은 두 가지다. 첫째, **`Course` 모델을 직접 받는 서브뷰는 하나도 없다.** 말풍선이 어떤 enum의 어떤 케이스에서 오는지, 일정 날짜가 어떤 이벤트 객체에서 오는지는 `CourseView`만 안다. Ch3의 결론이었던 "비즈니스 로직에 대한 앎은 기능 뷰 한 곳에 모은다"는 문장이 코드에서는 "도메인 값을 원시 값으로 푸는 작업이 `CourseView`의 `body`에서만 일어난다"는 모양으로 나타난다. 둘째, 도메인 타입의 값을 받는 서브뷰는 `SelectionView`뿐이고, 그것도 `Element: SelectionElement`라는 제약을 통해 인터페이스로만 받는다. 이 예외가 §4.7의 주제다.

바인딩의 구조도 Ch3의 그림과 맞아떨어진다. Figure 3.10에서 `CourseView`에서 `Course Model`로 향하는 굵은 화살표 하나가 기능 뷰를 기능 뷰로 만든다고 했는데, Listing 4.1의 첫 줄 `@Binding var course: Course`가 바로 그 화살표를 코드로 옮긴 것이다.

---

## § 4.6 `body`를 높은 수준으로 유지하기

> **(p.80)** *"`body` 변수를 보면, 강사 섹션이 약간 꼬여있는 것을 제외하고 각 뷰가 꽤 가독성 있다고 주장할 수 있습니다."*
>
> *"선언형 뷰 덕분에, 우리는 뷰를 쉽게 옮길 수 있습니다. 한 번 그러면 많은 리팩토링이 필요하지 않기 때문입니다. `tutorView`라는 로컬 속성을 만들고 관련 강사 코드를 거기로 옮길 것입니다."*

```swift
struct CourseView: View {
    @Binding var course: Course

    var body: some View {
        ScrollView {
            VStack {
                tutorView          // We use a new tutorView property
                // ... snip
            }
        }
    }

    // We introduce a private view property called tutorView
    private var tutorView: some View {
        HStack(alignment: .top) {
            ThumbDescriptionView(imageName: "Avatar",
                                 title: course.tutor.displayName,
                                 description: course.tutor.handle)
            Spacer()
            SubduedButton(action: openMessageInbox, label: "Message")
        }.padding([.horizontal, .top], 20)
    }
}
```
*Listing 4.2: tutorView 추출 (일부 생략)*

> **(p.81)** *"이제 `body`는 *섹션*을 높은 수준에서 보여줍니다."*
>
> *"새 `tutorView` 속성을 `private`으로 표시한다는 점에 주목하세요. 다른 어떤 뷰도 이 `tutorView`에 대해 알 필요가 없기 때문입니다. 그 위에, 우리는 이를 공유, 재사용 가능 뷰로 만들지 않도록 보장합니다. `tutorView`가 `CourseView`에 매우 특정하기 때문입니다."*
>
> **(p.81) Note**: *"또는, `tutorView`를 자체 (private) `TutorView` 타입으로 추출할 수 있습니다. 그러나 그것은 약간 더 오버헤드가 있습니다. 로컬 뷰가 더 복잡해지면(예: 더 많은 바인딩이 필요), 추출하는 것이 더 의미가 있습니다. 그러나 네임스페이스를 약간 더 오염시킬 수 있다는 점에 주목하세요. 추출한다면, 외부인에게 `CourseView`의 API를 더 작고 단순하게 유지하기 위해 private으로 만드는 것을 고려하세요."*

이 절의 선택지는 세 단계로 정리된다.

| 선택 | 비용 | 쓰는 시점 |
|---|---|---|
| `body` 안에 그대로 둔다 | 없음 | 섹션이 단순할 때 |
| `private` 뷰 속성으로 뺀다 | 거의 없음 | `body`를 훑어 읽기 어려워질 때 (이 절) |
| `private` 타입으로 뺀다 | 타입 정의, 바인딩 전달 | 로컬 뷰가 더 복잡해지거나 더 많은 바인딩이 필요할 때 |

뒤의 두 선택은 모두 **`private`**이라는 점이 공통이다. 이 결정은 Ch3의 원칙 10과 같은 계열이다. `tutorView`는 `CourseView`에 매우 특정하므로 공유 가능한 뷰로 만들지 않고, 필요해지는 순간에 공개 범위를 넓히면 된다. §3.4에서 본 "가설적 시나리오를 위해 재사용 가능하게 만들지 말라"는 경험 법칙이 이 절에서는 가시성 한정자 하나로 구현된다.

---

## § 4.7 SelectionView 구현: 원칙 9를 코드로 옮기기

이 절은 챕터에서 가장 길고, Ch3의 원칙 9(컴포넌트는 비즈니스 로직을 모른다)가 실제 코드가 되는 자리다.

> **(p.82)** *"이전 챕터에서 논의한 대로, 우리는 데이터 모델을 표현하기 위해 인터페이스(프로토콜) `SelectionElement`를 사용하는 `SelectionView`를 가집니다. 그런 다음 그 인터페이스를 사용하여 `SelectionView`는 `SelectionViewItem` 타입의 리스트를 렌더링할 수 있습니다. 이는 `SelectionView`가 한 특정 기능에 묶이는 것과 반대로 재사용 가능한 뷰 컴포넌트로 남을 수 있게 합니다."*
>
> *"마지막으로, `TodoItem`을 `SelectionElement`에 conform하게 만들어 `SelectionView`가 `TodoItem` 배열을 렌더링하게 만들 것입니다."* (Figure 4.5)
>
> **(p.82) Note**: *"SwiftUI를 사용하지 않더라도, 우리는 UIKit 같은 다른 패러다임에도 정확히 이 접근을 취할 수 있습니다. UI는 다르게 구현되겠지만, 개념은 동일합니다!"*

### § 4.7.1 프로토콜과 `SelectionView`의 정의

> **(p.82)** *"`SelectionElement`는 작은 인터페이스가 될 수 있습니다. id, 타이틀, 체크 여부(완료 여부)를 나타내는 boolean만 필요하기 때문입니다."*
>
> **(p.83)** *"주목할 한 가지 Swift 및 SwiftUI 특정 사항은 `SelectionElement` 타입이 *또한* `Hashable`과 `Identifiable` 프로토콜에 conform하도록 보장한다는 것입니다. 단순히 말해, 이는 SwiftUI가 `SelectionElement`가 상태를 변경할 때 UI를 다시 그릴지 결정하기 위해 이 요소들의 고유성을 확인할 수 있도록 보장합니다."*
>
> **(p.83) Note**: *"`title`과 `id`가 `get` 키워드로 보여지듯이 읽기 전용 속성임에 주목하세요. 그러나 `completed` 속성은 사용자가 작업을 완료할 때 변형되므로 `set`도 가능합니다."*

```swift
protocol SelectionElement: Hashable, Identifiable {
    var title: String { get }
    var id: UUID { get }
    var completed: Bool { get set } // This property can be mutated, as shown by the 'set' keyword.
}

extension TodoItem: SelectionElement {}
```
*Listing 4.3, 4.4: 바인딩 친화적 계약을 가진 `SelectionElement`, 그리고 `TodoItem`의 conform*

> **(p.83)** *"`TodoItem`이 `SelectionElement`와 같은 이름의 속성들을 사용하므로, 추가 작업을 수행할 필요가 없습니다. 따라서 extension의 본문은 비어 있습니다."*

두 가지를 읽어 둘 만하다. 첫째, 프로토콜은 `Course` 도메인이 아니라 **UI 라이브러리 쪽**에 있고, `extension TodoItem: SelectionElement {}`는 도메인 쪽에서 UI 쪽 인터페이스에 맞춰 들어오는 한 줄이다. Ch3의 Figure 3.3에서 `TodoItem → SelectionElement` 화살표가 도메인에서 라이브러리 방향이었던 것과 같다. 둘째, `completed`만 `get set`이라는 점은 이 인터페이스가 **읽기 전용 계약이 아니라 변경까지 포함한 계약**이라는 뜻이다. 이것이 뒤에서 바인딩이 동작하는 전제가 된다.

이어서 `SelectionView`의 속성을 정의한다. 책은 속성이 두 개뿐이라는 점과, 그 둘이 서로 다른 방향의 소통을 맡는다는 점을 설명한다.

> **(p.83)** *"`@Binding`을 사용하여 요소 배열이 전달되어야 하고, 우리가 이 요소 중 하나를 *업데이트*하면 그 소유자, 우리 경우 `Course`도 업데이트될 것임을 나타냅니다."*
>
> *"두 번째 속성은 사용자가 액세서리 버튼을 탭할 때 트리거될 액션입니다. 그 정의는 `(SelectionElement) -> Void`입니다."*
>
> **(p.84)** *"`TodoItem`의 완료 상태를 업데이트할 액션을 전달할 필요가 없다는 점에 주목하세요. 곧 보시겠지만, 대신 바인딩을 사용할 수 있습니다."*

> **읽을 때 주의할 지점**: Ch3 §3.2의 정리 문서에서 "양방향 바인딩의 두 방향은 서로 다른 도구로 끊는다"고 적었는데, 이 코드가 그 설명의 실례다. 한 컴포넌트 안에서 **상태 변경**(완료 토글)은 `@Binding`이 맡고, **의도의 전달**(액세서리 버튼을 눌렀다는 사실)은 클로저가 맡는다. 두 도구가 나뉜 이유는 뷰가 그 결과를 해석하는지로 읽을 수 있다(책이 명시한 기준은 아니다). 완료 토글은 뷰가 `element.completed.toggle()`로 값을 직접 바꾸고, 그 결과가 소유자까지 전파되면 끝난다. 반대로 액세서리 버튼은 눌린 뒤에 무엇이 일어나는지(상세 화면을 열 것인가)를 뷰가 전혀 모르고 알 필요도 없으므로, 호출만 하고 끝나는 클로저가 어울린다. Compose에는 `@Binding`에 해당하는 도구가 없어서 두 방향 모두 콜백이 된다. 이 차이는 "안드로이드로 옮기면"에서 다시 다룬다.

본문이 이어서 보이는 `SelectionView`의 `body`는 단순하다.

```swift
ForEach($elements) { element in
    // Inside the body of ForEach, return a SelectionItemView for each element.
    SelectionItemView(element: element, action: action)
        .padding(.vertical, 4)
        .padding(.horizontal, 10)
}
```
*Listing 4.6: `SelectionItemView` 행을 렌더링하는 `SelectionView` body (일부 생략)*

> **(p.85) Note**: *"요소를 `ForEach`에 전달하는 대신 달러 사인을 접두사로 한 `$elements`를 전달하는 것을 알아챘을 수 있습니다. 그것은 SwiftUI의 바인딩을 전달하는 방식입니다."*

### § 4.7.2 컴파일 타임과 런타임

> **(p.85)** *"거의 완성에 가까워지고 있습니다. 그러나 안타깝게도 `SelectionView`는 우리가 방금 정의한 방식으로 `SelectionElement`와 작동할 수 *없습니다*. Swift 컴파일러는 이 특정 상황에 대해 각 `SelectionElement`가 무엇을 나타내는지 *컴파일 타임*에 알아야 합니다. 현재, 우리는 `SelectionElement`를 *동적으로*, 런타임에 사용하고 있습니다."*
>
> *"이를 해결하기 위해, *제네릭(generics)*을 사용해 컴파일 단계 동안 `SelectionElement` 타입을 다루고 있다고 Swift 컴파일러에게 알려야 합니다."*
>
> **(p.85) Note**: *"제네릭이 이해하기 어렵다면, 괜찮습니다. 이해해야 할 핵심 개념은 `SelectionView`가 `SelectionElement` 타입과 작동한다는 것입니다."*

```swift
// SelectionView uses Element instead of SelectionElement
struct SelectionView<Element: SelectionElement>: View {
    @Binding var elements: [Element]
    var action: (Element) -> Void
    // ... snip
}
```
*Listing 4.7: 제네릭 `Element`를 사용하는 `SelectionView` 바인딩 (일부 생략)*

이 문제는 Swift의 프로토콜과 제네릭 규칙에서 나오는 제약이므로 세부 규칙은 외울 필요가 없다. 책도 같은 입장이다. 이해할 것은 결과 하나다. **인터페이스로 끊으려고 했더니 타입 시스템이 구체 타입 정보를 요구했고, 그 값을 제네릭이라는 형태로 지불했다.**

> **(p.86)** *"이 복잡함은 `SelectionView`를 재사용 가능하게 만드는 데 우리가 지불하는 가격입니다. 한편으로, 제네릭은 복잡하며 신참자가 이해하기 어렵게 만든다고 진술할 수 있습니다. 다른 한편으로, SwiftUI 뷰를 만드는 데 익숙해지면 이것이 작업하는 흔한 방식임을 알게 될 것입니다."*

> **읽을 때 주의할 지점**: Ch3 §3.4.1은 `SelectionView`를 재사용 가능하게 만드는 비용을 "속성 두세 개짜리 프로토콜 하나를 추가할 뿐이고, 뷰의 핵심 구현은 바뀌지 않는다"고 추정했다. 이 절의 실제 비용은 그보다 크다. 프로토콜은 `Hashable`과 `Identifiable`까지 요구하고, `SelectionView`와 `SelectionItemView` 두 곳에 제네릭 매개변수가 붙고, 바인딩 전달 방식도 바뀐다. 그렇다고 Ch3의 판단이 틀렸다는 말은 아니다. 책은 이 비용을 숨기지 않고 "가격"이라고 부르며 감수할 만하다고 판단한다. 다만 위험 기반 휴리스틱을 적용할 때 **언어와 프레임워크가 청구하는 추가 비용까지 계산에 넣어야 한다**는 교훈은 분명하다. 같은 인터페이스 도입이 Swift에서는 제네릭으로, Kotlin 멀티모듈에서는 모듈 의존의 방향 문제로 청구된다.

### § 4.7.3 `SelectionItemView` 구현

> **(p.86)** *"우리는 *그저* `SelectionItemView`에 라벨을 채울 타이틀을 전달할 수 있습니다. 그러나 그 상태를 변경해도 `TodoItem`이 업데이트되지 않을 것입니다. 그것을 작동시키려면 더 많은 풀칠이 필요할 것입니다."*
>
> *"대신, `@Binding`을 사용해 `SelectionElement`를 다시 전달할 수 있습니다. 이렇게 하면, 토글에 의해 요소를 업데이트하면, 변경이 자동으로 `SelectionView`로, 그리고 `Course` 인스턴스까지 모든 길로 전파됩니다. 이는 그런 다음 `CourseView`에서 UI 업데이트를 트리거할 것입니다."*
>
> *"그러나 반복하면, `SelectionItemView`는 `TodoItem`이나 코스를 *인지하지 않습니다*."*

```swift
struct SelectionItemView<Element: SelectionElement>: View {
    @Binding var element: Element
    var action: (Element) -> Void

    var body: some View {
        HStack {
            // The first button toggles the element.
            Button {
                element.completed.toggle()
            } label: {
                Image(systemName: element.completed ? "checkmark.circle.fill" : "checkmark.circle")
                    .foregroundColor(element.completed ? Color(.brandingGreen) : Color(.black))
                Text(element.title)
                    // ... frame, padding, color omitted
                Spacer()
            }

            // The second button triggers the action method. (When a user taps the accessory icon)
            Button {
                action(element)
            } label: {
                Image(systemName: "arrow.right.circle")
                    .foregroundColor(Color(.brandingPurple))
            }
        }.padding(.horizontal, 10)
    }
}
```
*Listing 4.8: `SelectionItemView` 구현 (일부 생략)*

토글 한 번이 거치는 경로를 따라가 보면 `@Binding`이 무엇을 대신해 주는지 분명해진다.

1. 사용자가 체크 버튼을 누르면 `SelectionItemView`가 `element.completed.toggle()`로 값을 바꾼다.
2. 그 바인딩은 `SelectionView`의 `elements` 배열의 해당 항목에 이어져 있으므로, 배열이 갱신된다.
3. 그 배열은 `CourseView`가 넘긴 `$course.schedule.weekly`(또는 `.daily`)에 이어져 있으므로, `course` 모델이 갱신된다.
4. `course`가 바뀌었으므로 `CourseView`가 다시 그려지고, 체크 표시가 바뀐 화면이 나온다.

네 단계 가운데 `Course`와 `TodoItem`을 아는 뷰는 `CourseView` 하나뿐이다. `SelectionItemView`는 `Element: SelectionElement`라는 사실만 알고 변경을 일으키며, 그 변경이 어디까지 올라가는지는 모른다.

> **읽을 때 주의할 지점**: Ch3의 Figure 3.3은 `SelectionItemView`를 아래층의 **View Primitives('Dumb', Simple views)**에 넣었다. 그런데 이 절의 구현은 `@Binding`을 받고 `element.completed.toggle()`이라는 로직을 직접 실행한다. Ch3의 원칙 8은 "로직이나 바인딩을 가진 뷰는 프리미티브가 아니라 컴포넌트"라고 정의했으므로, 이 구현은 정의로 따지면 **뷰 컴포넌트**에 가깝다. 책은 이 불일치를 따로 짚지 않는다. 이것을 오류로 읽을 필요는 없지만, 분류가 도해의 인상이 아니라 **실제 코드가 하는 일**로 결정된다는 점을 보여 주는 사례로는 쓸모가 있다. 도해 단계에서 프리미티브라고 부른 뷰가 구현 단계에서 컴포넌트로 승격되는 일은 §4.2.1이 경고한 "우연한 승격"의 무해한 형태다. 이 뷰도 비즈니스 로직은 모르므로 UI 라이브러리에 두는 결정에는 영향이 없다.

---

## § 4.8 todo 리스트 합성

> **(p.88)** *"이제 `SelectionView`로 표현되는 todo 리스트를 구현할 것입니다. 그러나 두 섹션을 가진다는 점에 주목하세요: 주간 일정용 하나와 일일 일정용 하나. 또한 타이틀 'This week's schedule', 섹션 헤더 'Every day', 그리고 'Reset all' 버튼을 표시할 것입니다."* (Figure 4.7)
>
> **(p.88) Note**: *"`SelectionView`가 풍부한 데이터를 포함하지 않는 이유는 'Delivering Reusable Views' 챕터에서 다룬 것처럼 그것이 더 단순하고 재사용 가능하게 만들기 때문입니다."*
>
> *"운 좋게도, 두 `SelectionView` 인스턴스를 쉽게 합성하여 하나의 todo 리스트를 만들 수 있습니다. `CourseView`가 타이틀과 리셋 버튼을 추가하는 역할을 맡을 것입니다."*

```swift
private var todoListView: some View {
    VStack {
        // The section title and button
        HStack {
            Text("This week's schedule").font(.title2).fontWeight(.semibold)
            Spacer()
            TextButton(label: "Reset all", action: resetAllTodoItems)
        }.padding([.horizontal, .top], 20).padding(.bottom, 10)

        // The weekly todo list
        SelectionView(elements: $course.schedule.weekly, action: openDetails(todoItem:))

        Text("Every day").font(.title3).fontWeight(.semibold)
            // ... frame, padding omitted

        // The daily todo list
        SelectionView(elements: $course.schedule.daily, action: openDetails(todoItem:))
        Spacer()
    }.background(Color(.brandingGrey))
}
```
*Listing 4.9: todo 리스트 추가 (일부 생략)*

Ch2 §2.5.5에서 제목과 리셋 버튼을 `SelectionView` 밖으로 밀어낸 결정이 여기서 효과를 낸다. `SelectionView`가 제목과 헤더까지 품고 있었다면 주간 목록과 일일 목록에 서로 다른 제목을 붙이기 위해 매개변수가 늘어났을 것이다. 지금은 같은 뷰를 두 번 쓰고 그 사이에 `Text`를 끼우면 끝난다. 책이 "합성하여"라고 표현한 것이 이 구조다.

또 하나 눈여겨볼 점은 `$course.schedule.weekly`처럼 `Course`의 **하위 속성에 대한 바인딩**을 곧바로 만들어 넘긴다는 것이다. `SelectionView`는 자신이 받은 배열이 주간 목록인지 일일 목록인지 알지 못한다. 어느 목록을 그릴지 고르는 것은 전적으로 `CourseView`의 몫이다.

> **(p.89)** *"이전에 `tutorView`로 한 것처럼, 정의의 대부분을 body 밖으로 옮기면 사람들이 화면이 어떻게 구성되는지 이해하기 쉬워집니다. 그러나 뷰가 어떻게 만들어지는지 디테일에 들어가고 싶다면, 디테일을 보기 위해 아래로 스크롤할 수 있습니다."*

이 절의 마지막 문장은 §4.6에서 만든 `private` 속성 패턴의 효용을 요약한다. `body`는 위에서 아래로 훑어 읽고, 자세한 내용이 궁금하면 파일을 내려가며 읽는다.

---

## § 4.9 뷰 프리미티브 구현

> **(p.90)** *"구현을 마무리하기 위해, 뷰 프리미티브가 어떻게 구현되는지 살펴봅시다."*
>
> *"먼저, `Course`에 로컬한 뷰 프리미티브, 즉 `ScheduleView`부터 시작합시다. 다른 프리미티브보다 더 정교하기 때문입니다."*

`ScheduleView`는 Ch3의 원칙 10이 말한 "기능 폴더에 살지만 기능을 모르는 뷰"의 실제 구현이다. 책은 이 뷰의 상태가 날짜 유무로 갈린다는 점에서 출발한다.

> **(p.90)** *"주목할 한 가지는 `ScheduleView`에 date 속성이 있다는 것입니다. 날짜가 설정되었는지가 뷰 상태를 결정합니다. 날짜가 없으면 다른 상태를 보여줄 것입니다."* (Figure 4.8, 4.9)
>
> *"코드에서 이 상태를 `date` 속성을 옵셔널로 만듦으로써 반영합니다. 그것은 전달되며, 상태에 따라 `filledView` 또는 `emptyView`를 보여줍니다."*

```swift
struct ScheduleView: View {
    let date: Date?
    let actionJoinCall: () -> Void
    let actionReschedule: () -> Void

    var body: some View {
        VStack {
            Text("Next 1-on-1") // ... frame, padding, font omitted
            HStack {
                Image(systemName: "calendar")
                if let date {
                    filledView(date: date)   // We unwrap the date, and pass it to filledView
                } else {
                    emptyView                // There is no date, show an empty view.
                }
            }.padding(.horizontal, 20).padding(.bottom, 1)

            TextButton(label: date == nil ? "Schedule" : "Reschedule",
                       action: actionReschedule)
                // ... frame, padding omitted
        }
    }
}
```
*Listing 4.10: filled 또는 empty 상태를 보여주는 `ScheduleView` (일부 생략)*

> **(p.91)** *"그러나 `filledView`는 자신을 채우기 위해 unwrap된 `date`가 필요합니다. 이를 해결하는 한 방법은 `filledView`를 뷰 속성으로 만드는 것과 반대로, `date` 매개변수를 받는 함수로 만드는 것입니다."*
>
> *"왜 `filledView`가 `ScheduleView`의 옵셔널 `date`를 사용하지 않는지 궁금하다면, 옵셔널 값을 다뤄야 하기 때문입니다. 그것은 *filled* 뷰에 거의 의미가 없습니다. 반면 *전달*하면 그 메서드에 대해 `date`는 *항상* 사용 가능하거나 '채워져' 있습니다."*

```swift
// We'll use ~Group~, as opposed to ~VStack~, so we can return a list of views without imposing a layout.
private func filledView(date: Date) -> some View {
    Group {
        Text(date, format: Date.FormatStyle(date: .abbreviated, time: .shortened))
        Spacer()
        Button(action: actionJoinCall, label: {
            Text("Join call").foregroundColor(Color(.brandingPurple))
            Image(systemName: "arrow.right.circle").foregroundColor(Color(.brandingPurple))
        })
    }
}

private var emptyView: some View {
    Text("Not set").frame(maxWidth: .infinity, alignment: .leading) // Left align
}
```
*Listing 4.11: filled와 empty 서브뷰를 가진 `ScheduleView` 헬퍼 (일부 생략)*

이 절이 가르치는 요령은 **옵셔널을 한 번만, 분기 지점에서만 벗긴다**는 것이다. `date`의 옵셔널성은 "일정이 아직 없다"는 도메인 사실이므로 `ScheduleView`의 최상단에서 한 번 처리하고, 그 안쪽의 `filledView`는 값이 있다는 전제만 받는다. 옵셔널이 코드 안쪽까지 퍼지지 않도록 타입으로 막는 방식이다.

`ScheduleView`가 받는 값이 `Course`나 `CalendarEvent`가 아니라 `Date?` 하나라는 점도 놓치지 말아야 한다. §4.5의 표에서 본 대로 `CourseView`가 `course.calendarEvent?.date`로 값을 꺼내 넘기므로, `ScheduleView`는 `Course` 폴더에 살면서도 `Course`를 모르는 상태를 유지한다. 이 뷰를 나중에 UI 라이브러리로 옮겨야 할 때 이동 비용이 거의 없다는 원칙 10의 요구를 코드가 충족한다.

### § 4.9.1 UI 라이브러리의 뷰 프리미티브

> **(p.92)** *"남은 뷰들은 UI 라이브러리에 사는 뷰 프리미티브입니다. 완전성을 위해 살펴보겠지만, 작은 정의이므로 깊이 들어가지는 않을 것입니다."*

`ThumbDescriptionView`, `TextButton`, `SubduedButton`, `CalloutView`는 각각 Listing 4.12~4.15에 구현되어 있다. 모두 문자열과 클로저만 받는 짧은 뷰이므로 코드를 옮기지 않고 특징만 정리한다.

| 뷰 | 받는 값 | 구현 특징 |
|---|---|---|
| `ThumbDescriptionView` | 이미지 이름, 제목, 설명 | `HStack` 안에 이미지와 두 줄 텍스트를 쌓는다 |
| `TextButton` | 라벨, 액션 | `Button` 하나에 작은 글씨와 브랜드 색을 입힌다 |
| `SubduedButton` | 액션, 라벨 | 위와 같다 |
| `CalloutView` | 메시지, 액션 | `ZStack`으로 둥근 사각형 위에 닫기 아이콘을 겹치고 오프셋으로 모서리에 걸친다 |

> **(p.93) Note**: *"여기 우리가 이전에 사용하지 않았던 새 SwiftUI 요소가 하나 있습니다: `ZStack`. 요소를 서로 위에 쌓을 수 있게 해줍니다."*

> **읽을 때 주의할 지점**: Listing 4.13의 `TextButton`과 Listing 4.14의 `SubduedButton`은 본문이 서로 같다. 둘 다 `font(.footnote)`에 브랜드 보라색이고, 달라진 것은 매개변수의 선언 순서(`label, action`과 `action, label`)뿐이다. 책은 이 점을 설명하지 않는다. 가장 자연스러운 해석은 §4.2.4의 mixed 접근과 §4.12의 "기본적인 스타일링"이다. 두 버튼은 디자인상으로는 서로 다른 역할이지만, 스타일링을 뒤로 미룬 지금은 같은 외형으로 구현되어 있을 뿐이다. Ch2의 원칙 6이 말한 대로 **이름은 현재의 외형이 아니라 역할을 가리키므로**, 외형이 같아도 이름을 합치지 않고 두는 것이 맞다. 디자이너가 나중에 두 버튼의 외형을 갈라놓는 순간, 이름을 바꿀 필요 없이 구현만 고치면 된다.

---

## § 4.10 진척 점검

> **(p.95)** *"시뮬레이터를 열면, 꽤 작동 가능한 화면을 볼 수 있습니다!"* (Figure 4.10)
>
> **(p.96)** *"이는 픽셀 퍼펙트가 아닙니다. 네, 디자이너가 이 거친 화면을 보면 움찔할 것입니다. 그러나 작동합니다! UI 라이브러리와 기능 화면을 가지고 있습니다. 단지 아직 다듬어지지 않았을 뿐이며, 이는 우리가 실제 타입으로 계속 진행할 수 있게 해줍니다."*
>
> *"남은 것은 이 화면을 내비게이션을 위해 준비하는 것입니다. 챕터를 마무리하기 위해 다음으로 그것을 합시다."*

마지막 문장 "실제 타입으로 계속 진행할 수 있게 해준다"가 이 절의 핵심이다. 거친 화면이 가치 있는 이유는 보기에 좋아서가 아니라, 이 화면이 이미 **진짜 `Course` 타입과 진짜 서브뷰 API로 연결되어 있기 때문**이다. 외형을 다듬는 일은 이 연결을 건드리지 않고 나중에 할 수 있다.

---

## § 4.11 내비게이션 준비

> **(p.96)** *"어떤 액션들, 즉 `openMessageInbox`, `openScheduler`, `joinCall`, `openDetails`은 모두 플로우의 한 지점을 트리거합니다."*
>
> *"모멘텀을 유지하기 위해, 이 액션들을 나중에 구현하기로 결정할 수 있습니다. 우리가 *할 수 있는* 것은 우리가 이 액션을 트리거할 때마다 함수, 파일, 줄을 콘솔에 출력할 작은 placeholder를 추가하는 것입니다."*
>
> **(p.96) Note**: *"우리는 'Reusing Views Across Flows' 챕터에서 내비게이션을 다룰 것입니다."*

```swift
// Navigation-specific.
private func joinCall() {
    print("Implement \(#function) on \(#file), line: \(#line)")
}
// openScheduler(), openMessageInbox(), openDetails(todoItem:) 도 같은 본문이다.
```
*Listing 4.16: 내비게이션 훅을 위한 `CourseView` 준비 (일부 생략)*

> *"내비게이션 관련 액션의 경우, 트리거할 때 콘솔이 우리가 그것을 구현할 수 있는 줄을 출력할 것입니다."*
> ```
> Implement joinCall() on /Users/tjeerdintveen/workspace/TutorApp/CourseView.swift, line: 134
> ```

이 스텁의 설계 의도는 **빈 함수가 아니라 자기 위치를 알려 주는 함수**라는 데 있다. 빈 함수는 눌러도 아무 일이 없어서 구현을 잊기 쉽지만, 이 스텁은 누를 때마다 "여기를 구현해야 한다"는 메시지를 파일과 줄 번호까지 붙여 출력한다. 구현을 미루면서도 미뤄 둔 일의 목록이 잊히지 않게 하는 장치다.

> **읽을 때 주의할 지점**: 책은 §4.5와 §4.8의 코드에서 `dismissCallout`과 `resetAllTodoItems`도 액션으로 참조하지만, 이 절의 스텁 목록에는 내비게이션 액션 네 개만 있다. 두 함수의 정의는 `// .. Rest is omitted`에 가려져 있다. 이 구분이 의미를 가진다. 네 개는 **화면 바깥으로 이동하는 액션**이라 다른 챕터로 미룰 수 있다. 나머지 둘은 이름으로 미루어 `Course` 모델의 상태를 바꾸는 액션으로 보이며(책은 정의를 보여 주지 않는다), 그렇다면 `CourseView`가 `@Binding`을 통해 직접 처리할 수 있는 종류다. 즉 이 챕터가 미루는 것은 "이동"이고, 모델 변경은 바인딩으로 해결할 수 있는 것으로 보인다.
>
> 또 하나, 책의 Listing 4.17에는 "내비게이션 와이어링을 위한 액션 컨테이너 구조체"라는 캡션이 붙어 있지만 실제로는 위의 콘솔 출력 블록에 붙어 있고, 구조체 코드는 이 챕터 어디에도 없다. 캡션이 잘못 붙은 것으로 보인다.

---

## § 4.12 결론

> **(p.97)** *"이 단계에서 `CourseView`가 충분히 끝났다고 간주할 수 있습니다. 작동하고, UI를 데이터에 연결하며, 적절한 서브뷰를 사용합니다."*
>
> *"필요한 뷰를 정의했지만, 그것들에 기본적인 스타일링을 주었습니다. 이 책에서 우리는 UI 스타일링을 다루지 않을 것이지만, 실제 프로젝트에서도 스타일링을 더 낮은 우선순위로 두는 것을 고려하세요."*
>
> *"비슷하게, 모든 UI 컴포넌트를 연결하고 작동하는 애플리케이션을 제공함으로써 '더 큰 그림'에 집중하면, 처음에는 덜 다듬어 보일지라도 모멘텀을 유지할 수 있습니다."*

이 챕터가 "끝났다"고 판정하는 기준이 세 가지로 적혀 있다. 작동하는가, UI가 데이터에 연결되어 있는가, 적절한 서브뷰를 쓰는가. 이 중 외형에 대한 기준은 하나도 없다.

저자는 이 기준을 디자이너와의 관계에서 변호하기도 한다.

> **(p.97)** *"이는 디자이너를 행복하게 하지 않을 수 있습니다. 그러나 예쁜 앱보다 *기능적인* 앱을 갖는 것이 더 중요합니다! 그러면 모두가 이미 백엔드 통합을 테스트하고, 놀고, 변경이 필요한지 결정할 수 있습니다. 실제 통합을 테스트하면 UI가 변할 가능성이 큽니다."*
>
> *"또는, `CourseView`를 픽셀 퍼펙트로 만들어 디자이너를 행복하게 할 *수도* 있지만, 그러면 모든 기능 테스트를 미루게 됩니다."*

여기서 "실제 통합을 테스트하면 UI가 변할 가능성이 크다"는 문장은 거친 구현을 정당화하는 가장 실용적인 근거다. 외형을 먼저 다듬으면 그 노력이 통합 테스트 이후의 변경으로 버려질 수 있다. 반대로 외형을 미루면 노력을 버릴 일이 없다.

다만 저자는 거친 구현이 **끝이 아니라는 점**도 명시한다.

> **(p.97)** *"스타일링 외에, 우리 뷰는 접근성 지원, 우-좌 텍스트 지원, 다크(나이트) 모드, 적절한 동적 폰트 크기 조정, 매우 작거나 큰 디바이스와의 호환성 등 보조 요구사항을 포함해 더 많은 작업이 필요하다는 점에 주목하세요."*
>
> *"그러나 `CourseView`를 기능적 수준에서 끝났다고 간주하려면, *누가* 코스를 로드하는지 알아내야 *하고* 구현 placeholder를 포함하는 마지막 메서드를 끝내야 합니다. 특히 내비게이션 관련 액션입니다. 다가오는 챕터에서 두 주제를 모두 다룰 것입니다."*

남은 일은 두 갈래로 나뉜다. 하나는 접근성, RTL, 다크 모드, 동적 폰트, 기기 크기 같은 **품질 요구사항**이고, 다른 하나는 데이터 로딩 주체와 내비게이션 액션이라는 **기능 요구사항**이다. 책은 후자를 "기능적 수준에서 끝났다고 간주하기 위한 조건"으로 분명히 적었다. 따라서 `CourseView`는 이 챕터가 끝난 시점에서 화면으로는 완성되었지만, 기능으로는 두 가지가 남아 있다.

---

## § 4.13 챕터 요약

> **(p.98)**
> **전략적 구현 접근**
> - top-down 접근은 서브뷰를 추출하는 데 유용하다.
> - top-down 접근의 한 함정은 기능을 서브뷰와 우연히 너무 단단히 결합하기 쉽다는 것이다.
> - 기능 뷰(또는 화면)를 구현하기 위한 bottom-up 접근은 기능 뷰와 그 서브뷰 사이에 강한 장벽을 만드는 데 유용하다.
> - bottom-up 접근의 한 함정은 서브뷰의 디테일에 산만해지기 쉽다는 것이다.
> - 강한 결합을 피하기 위해 서브뷰를 미리 정의하지만 완전히 구현하지는 않는 holistic 접근 같은 접근의 혼합을 사용할 수 있다.
>
> **프레임워크 무관 아키텍처 패턴**
> - 다른 패러다임에도 불구하고, 선언형과 비슷한 명령형 컴포넌트를 구현할 수 있다.
> - SwiftUI, UIKit, Flutter, React Native 또는 다른 프레임워크를 사용하든 같은 아키텍처 원리가 적용된다.
> - 인터페이스 기반 디자인 패턴(`SelectionElement` 같은)은 특정 프레임워크를 초월하는 재사용 가능한 컴포넌트를 만든다.
>
> **액션 기반 아키텍처**
> - 기능성을 유지하면서 뷰를 비즈니스 로직에서 분리하기 위해 액션을 전달하는 것.
> - 명령형 코드는 클로저를 사용해 자동 바인딩의 부족을 보상해야 한다.
> - 다중 클로저에서 통합된 액션 패턴으로의 진화.
>
> **실용적 트레이드오프**
> - 모멘텀을 유지하려면, UI를 "기능적으로" 먼저 구현하는 것이 중요하다. 화면을 기능적으로 만든 후에 정확한 마진과 레이아웃을 저장하라.
> - 작동하는 기능을 전달할 때 속도와 완벽성의 균형.
> - 기능 뷰의 body를 더 최상위로 유지하면 추론하기 쉬워진다.
> - 메인 뷰(body)를 더 상위 수준으로 유지하기 위해 로컬 private 서브뷰를 만드는 것을 고려하라.
>   - 더 성숙해지거나 더 많은 로직이 필요해지면 로컬 뷰를 자체 뷰로 추출하라.

요약의 네 묶음은 각각 §4.2(구현 접근), §4.7(인터페이스 기반 컴포넌트), §4.11(액션), §4.6과 §4.12(트레이드오프)에 대응한다. 한 가지 주의할 점은 **"액션 기반 아키텍처" 묶음의 마지막 줄**이다. "다중 클로저에서 통합된 액션 패턴으로의 진화"는 이 챕터 본문에 코드로 나오지 않는다. 본문이 보여 주는 것은 `SelectionView`의 클로저 하나(`action`)와 `CourseView`의 내비게이션 스텁 네 개뿐이다. 이 줄은 Ch9에서 다중 클로저가 `CourseActions`라는 인터페이스로 합쳐지는 과정을 미리 가리키는 문장으로 읽는 것이 맞다.

이 챕터 전체를 한 문장으로 압축하면 다음과 같다. **서브뷰의 경계(누가 무엇을 아는가)는 처음부터 정확하게 긋고, 외형과 로딩과 내비게이션은 작동하는 화면을 얻은 뒤에 채운다.**

---

## 안드로이드로 옮기면

이 챕터는 코드가 많아서 안드로이드 개발자가 가장 직접적으로 옮겨 볼 수 있는 챕터다. 다만 SwiftUI의 `@Binding`과 Compose의 상태 모델은 설계 철학이 달라서, 그대로 옮기면 책이 칭찬한 단순함이 사라지는 부분이 있다. 아래에서는 대응 관계를 먼저 정리하고, 달라지는 지점을 따로 짚는다.

### SwiftUI 요소의 Compose 대응

| SwiftUI | Compose | 비고 |
|---|---|---|
| `ScrollView` | `Column(Modifier.verticalScroll(rememberScrollState()))` | 항목이 많아 가상화가 필요하면 `LazyColumn` |
| `VStack` / `HStack` / `ZStack` | `Column` / `Row` / `Box` | |
| `Group` | 대응하는 컨테이너가 없다 | 컴포저블 함수는 호출한 위치의 스코프에 바로 내용을 내보낸다. 스코프 멤버(`weight` 등)가 필요하면 `RowScope.` 확장 함수로 선언한다 |
| `Spacer` | `Spacer`, 또는 `Modifier.weight(1f)` | 남는 공간을 채울 때는 `weight`가 더 흔하다 |
| `padding` | `Modifier.padding` | |
| `Divider` | `HorizontalDivider` | Material3 1.2 이상 |
| `@State` | `remember { mutableStateOf(...) }` | 뷰가 소유하는 상태 |
| `@Binding` | **대응하는 도구가 없다** | 상태 호이스팅(`value` + 변경 콜백)으로 대체한다 |
| `ForEach` | `forEach` + `key(id)`, 또는 `LazyColumn`의 `items(key = ...)` | `Identifiable`의 `id`가 `key`에 해당한다 |
| `some View` | 대응하는 개념이 없다 | 컴포저블은 `Unit`을 반환하는 함수다 |

### 접근 선택(§4.2): 모듈 경계가 만드는 "강한 장벽"

책은 top-down의 위험을 "별도 뷰가 아직 추출되지 않았으므로 강한 결합을 막을 장벽이 없다"고 설명했다. 안드로이드 멀티모듈 프로젝트에서는 이 장벽의 일부를 컴파일러가 만들어 준다. 서브뷰를 `:core:ui` 모듈로 옮기는 순간, 그 컴포저블이 `Course`나 `TodoItem` 같은 도메인 타입을 참조하고 있다면 **컴파일이 실패**한다. 그래서 top-down으로 시작하더라도 추출 시점에 결합이 드러난다. 추출한 함수의 **매개변수 목록이 곧 의존 목록**이라는 점도 기억해 둘 만하다. 매개변수에 도메인 타입이 하나라도 보이면, 그 컴포저블은 §4.2.1이 경고한 "우연한 승격"을 당한 것이다.

### 기능 뷰: 도메인 값을 원시 값으로 푸는 자리(§4.5, §4.6, §4.8)

§4.5의 표(`CourseView`가 서브뷰마다 넘기는 값)를 Compose로 옮기면 다음 모양이 된다. 두 가지가 책의 코드와 다르다. 첫째, `@Binding var course` 대신 **값 하나와 이벤트 콜백**을 받는다. 둘째, SwiftUI의 `private var tutorView`는 `self.course`를 암묵적으로 읽었지만, Compose의 `private` 컴포저블은 필요한 값을 **매개변수로 명시**해야 한다.

```kotlin
// 기능 뷰: Course를 아는 유일한 자리. 도메인 값을 원시 값으로 풀어서 서브뷰에 넘긴다.
@Composable
fun CourseScreen(
    course: Course,
    onToggleTodo: (TodoItem) -> Unit,
    onResetAll: () -> Unit,
    onDismissCallout: () -> Unit,
    onOpenMessageInbox: () -> Unit,
    onOpenScheduler: () -> Unit,
    onJoinCall: () -> Unit,
    onOpenDetails: (TodoItem) -> Unit,
    modifier: Modifier = Modifier,
) {
    Column(modifier.verticalScroll(rememberScrollState())) {
        TutorSection(tutor = course.tutor, onMessageClick = onOpenMessageInbox)

        // 책의 `if case let .message(message) = course.tutor.callOut` 에 해당한다.
        val callOut = course.tutor.callOut
        if (callOut is CallOut.Message) {
            Callout(message = callOut.text, onDismiss = onDismissCallout)
        }

        HorizontalDivider(Modifier.padding(horizontal = 20.dp))

        ScheduleCard(
            date = course.calendarEvent?.date,   // ScheduleCard는 Course를 모른다.
            onJoinCall = onJoinCall,
            onReschedule = onOpenScheduler,
        )

        TodoListSection(
            schedule = course.schedule,
            onToggle = onToggleTodo,
            onOpenDetails = onOpenDetails,
            onResetAll = onResetAll,
        )
    }
}

// §4.6의 private tutorView. 가시성이 private이라 다른 곳에서 공유되지 않는다.
// SwiftUI는 self.course를 암묵적으로 읽지만, Compose는 필요한 값을 매개변수로 선언한다.
@Composable
private fun TutorSection(tutor: Tutor, onMessageClick: () -> Unit) {
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .padding(start = 20.dp, end = 20.dp, top = 20.dp),
        verticalAlignment = Alignment.Top,
    ) {
        ImageLabelRow(                      // 책의 ThumbDescriptionView
            imageRes = R.drawable.avatar,
            title = tutor.displayName,
            description = tutor.handle,
            modifier = Modifier.weight(1f),
        )
        SubduedButton(label = "Message", onClick = onMessageClick)
    }
}
```

`TodoListSection`은 §4.8의 `todoListView`에 해당한다. `Column` 안에 제목 행, `SelectionList`, "Every day" 헤더, `SelectionList`를 차례로 두는 구성이므로 본문은 생략한다.

콜백이 일곱 개나 되어 부담스럽게 보인다. 책도 같은 문제를 알고 있고, 요약의 "다중 클로저에서 통합된 액션 패턴으로의 진화"가 그 해결을 예고한다(Ch9에서 `CourseActions` 인터페이스로 합쳐진다). 이 챕터의 단계에서는 콜백을 나열하는 것이 책의 코드와 같은 위치이므로 그대로 둔다.

`CourseScreen`은 전달받은 값만으로 그려지므로 `@Preview`와 UI 테스트가 이 함수 하나만 대상으로 삼을 수 있다. §4.3에서 `Course`를 전달받기로 한 결정의 이점이 여기서 드러난다. `CourseScreen(course = Course.placeholder, ...)`처럼 HDD에서 만든 placeholder를 그대로 넣어 에뮬레이터 없이도 화면을 확인할 수 있다.

### 원칙 9의 구현(§4.7): 인터페이스는 같고, 제네릭의 이유는 다르다

책의 `SelectionElement`를 Kotlin으로 옮기면 훨씬 짧아진다. `Hashable`과 `Identifiable`은 필요 없고(Compose에서는 `key(id)`가 같은 역할을 한다), 변경 가능한 `completed`도 필요 없다(상태는 호이스팅되므로).

```kotlin
// :core:ui. UI 라이브러리가 인터페이스를 소유한다.
interface SelectionElement {
    val id: String
    val title: String
    val completed: Boolean
}

// 제네릭 E가 필요한 이유는 Swift와 다르다.
// Swift: 컴파일러가 구체 타입을 요구한다.
// Kotlin: 없어도 컴파일되지만, 콜백이 SelectionElement를 돌려주면 호출부가 TodoItem으로 캐스팅해야 한다.
//         E를 두면 onToggle과 onAccessoryClick이 호출부의 타입(TodoItem)을 그대로 돌려준다.
@Composable
fun <E : SelectionElement> SelectionList(
    elements: List<E>,
    onToggle: (E) -> Unit,
    onAccessoryClick: (E) -> Unit,
    modifier: Modifier = Modifier,
) {
    // 이 챕터의 SelectionView는 CourseView의 스크롤 컨테이너 안에 놓인다.
    // verticalScroll 안에 LazyColumn을 넣으면 높이 제약이 무한대라 예외가 나므로 Column을 쓴다.
    Column(modifier) {
        elements.forEach { element ->
            key(element.id) {
                SelectionItemRow(
                    title = element.title,
                    completed = element.completed,
                    onToggle = { onToggle(element) },
                    onAccessoryClick = { onAccessoryClick(element) },
                    modifier = Modifier.padding(horizontal = 10.dp, vertical = 4.dp),
                )
            }
        }
    }
}
```

UI-03 정리 문서의 `SelectionList`는 `LazyColumn`이었지만, 이 챕터처럼 스크롤 컨테이너 안에서 쓸 때는 `Column`으로 바꿔야 한다. 스크롤되는 부모 안에 `LazyColumn`을 넣으면 "무한 최대 높이로 측정되었다"는 예외가 나기 때문이다. 같은 컴포넌트라도 **놓이는 위치에 따라 구현이 달라질 수 있다**는 점을 확인할 수 있다.

`SelectionItemRow`의 본문은 `Row` 안에 토글 영역(`Modifier.clickable(onClick = onToggle)`을 단 아이콘과 `Text`)과 액세서리 `IconButton`을 나란히 두면 되므로 생략한다. 책의 `SelectionItemView`가 버튼 두 개를 `HStack`에 둔 구조와 같다.

### `@Binding`이 없는 세계: 변경도 이벤트가 된다

여기가 이 챕터를 안드로이드로 옮길 때 가장 크게 달라지는 지점이다. 책은 `SelectionView`의 완료 토글을 `@Binding`으로, 액세서리 탭을 클로저로 나누었다. Compose에는 `@Binding`이 없으므로 두 방향이 모두 콜백이 된다.

```kotlin
// 책(SwiftUI): 완료 토글이 바인딩을 타고 Course까지 자동으로 올라간다.
//   SelectionView(elements: $course.schedule.weekly, action: openDetails(todoItem:))
//
// Compose: 변경은 "토글해 달라"는 이벤트로 올라가고, 새 상태가 다시 내려온다.
SelectionList(
    elements = schedule.weekly,
    onToggle = onToggle,               // 위로: 이 항목을 토글해 달라는 요청
    onAccessoryClick = onOpenDetails,  // 위로: 상세를 열어 달라는 요청
)
```

이 차이가 낳는 결과는 두 가지다. 첫째, 책이 칭찬한 "`TodoItem`의 완료 상태를 업데이트할 액션을 전달할 필요가 없다"는 단순함이 사라진다. 완료 토글에도 콜백이 하나 필요하다. 둘째, 변경을 **반영하는 일**이 뷰 밖으로 완전히 나간다. `onToggle`을 받은 쪽(ViewModel이나 상태를 소유한 곳)이 불변 객체에 `copy`를 적용해 새 `Course`를 만들고, 그것이 다시 `CourseScreen`의 `course` 매개변수로 내려온다. 책의 §4.7.3에서 본 네 단계 경로가 Compose에서는 "이벤트가 위로 올라가고, 새 값이 아래로 내려온다"는 단방향 흐름으로 바뀌는 셈이다.

`MutableState`를 매개변수로 직접 넘기면 `@Binding`과 비슷한 효과를 낼 수 있지만, 모델이 Compose 런타임에 묶이고 변경 지점이 흩어지므로 권장되지 않는다. 변경을 실제로 어디서 반영하는가는 이후 챕터에서 다시 나온다. Ch6 §6.6은 todo 토글이 `courseService.update(course:)` 호출로 이어지는 과정과, 뷰가 진실 공급원이 되는 것의 한계를 다룬다.

`SelectionElement` 자체의 의존 방향에는 Ch3 정리 문서에서 짚은 제약이 그대로 남아 있다. Kotlin에는 소급 준수가 없으므로, `data class TodoItem(...) : SelectionElement`라고 쓰면 도메인 모듈이 `:core:ui`에 의존하게 된다. 위 예시는 책의 구조를 그대로 옮기는 단일 모듈 기준이고, 멀티모듈에서는 Ch3 정리 문서의 "소급 준수의 부재" 절에서 설명한 변형(기능 UI 모듈에서 도메인을 `SelectionItem`으로 변환하고, 콜백은 `(id: String) -> Unit`로 받는 방식)을 따른다. 그 변형에서는 콜백이 타입이 아니라 `id`를 돌려주므로 제네릭도 필요 없어진다.

### 옵셔널을 한 번만 벗기기(§4.9)

`ScheduleView`의 설계 요령(옵셔널을 분기 지점에서만 벗기기)은 Kotlin의 스마트 캐스트로 같은 모양이 된다. `Group`이 하던 일(레이아웃을 강요하지 않고 여러 뷰를 내보내기)은 `RowScope.` 확장 컴포저블이 맡는다.

```kotlin
// :feature:course 모듈. 기능 폴더에 살지만 Course를 모른다. 원칙 10.
@Composable
internal fun ScheduleCard(
    date: LocalDateTime?,
    onJoinCall: () -> Unit,
    onReschedule: () -> Unit,
    modifier: Modifier = Modifier,
) {
    Column(modifier.padding(vertical = 10.dp)) {
        Text(
            "Next 1-on-1",
            style = MaterialTheme.typography.bodySmall,
            modifier = Modifier.padding(start = 20.dp, bottom = 2.dp),
        )
        Row(
            modifier = Modifier.padding(horizontal = 20.dp),
            verticalAlignment = Alignment.CenterVertically,
        ) {
            Icon(Icons.Default.DateRange, contentDescription = null)
            // `if (date != null)` 분기 안에서 date는 non-null로 스마트 캐스트된다.
            if (date != null) FilledContent(date, onJoinCall) else EmptyContent()
        }
        TextButton(onClick = onReschedule) {
            Text(if (date == null) "Schedule" else "Reschedule")
        }
    }
}

// 책의 filledView(date:). 값이 있다는 전제만 받는다.
// RowScope 확장이라 호출한 Row의 weight를 쓸 수 있다. 책의 Group + Spacer에 해당한다.
// scheduleFormatter는 DateTimeFormatter.ofLocalizedDateTime(FormatStyle.MEDIUM, FormatStyle.SHORT)로
// 정의했다고 가정한다 (책의 .abbreviated, .shortened에 해당).
@Composable
private fun RowScope.FilledContent(date: LocalDateTime, onJoinCall: () -> Unit) {
    Text(date.format(scheduleFormatter), Modifier.weight(1f))
    TextButton(onClick = onJoinCall) { Text("Join call") }
}

@Composable
private fun RowScope.EmptyContent() {
    Text("Not set", Modifier.weight(1f))
}
```

`internal` 한정자가 원칙 10의 결정("이 뷰는 이 기능 모듈 안에서만 쓴다")을 컴파일러가 강제하는 규칙으로 바꿔 준다는 점은 Ch3 정리 문서에서 이미 다뤘다. `CalloutView`의 `ZStack`은 `Box`에 `Modifier.align(Alignment.TopEnd).offset(x = 10.dp, y = (-10).dp)`를 단 닫기 아이콘으로 옮기면 된다.

### 거친 구현과 스텁(§4.2.4, §4.11)

책의 색상 자산(`.brandingPurple`)은 Compose에서 `Color` 상수나 `MaterialTheme.colorScheme`의 항목에 해당하고, 이 역시 이름이 컴파일 타임에 검사된다. 색 체계를 제대로 정의하는 일은 책이 약속한 대로 Scale 권의 UI 라이브러리 기초 챕터에서 다룬다. 그 전까지는 Material3의 기본 컴포넌트와 기본 테마로 충분히 "대략 디자인처럼 보이는" 화면을 만들 수 있다.

내비게이션 스텁은 Kotlin에서 한 가지 주의가 필요하다.

```kotlin
// 주의: TODO()는 NotImplementedError를 던지므로 버튼을 누르는 순간 앱이 종료된다.
// 책의 스텁처럼 "작동하는 화면"을 유지하려면 로그만 남겨야 한다.

// 책의 print("Implement \(#function) on \(#file), line: \(#line)") 에 해당한다.
// Kotlin에는 #function 같은 매크로가 없으므로 스택 트레이스에서 호출 위치를 읽는다.
fun todoStub() {
    val caller = Throwable().stackTrace[1]
    Log.d("TodoStub", "Implement ${caller.methodName} on ${caller.fileName}:${caller.lineNumber}")
}

// 사용: 람다 안에서 호출해야 호출 위치(파일, 줄)가 정확히 찍힌다.
CourseScreen(
    course = course,
    onJoinCall = { todoStub() },
    onOpenScheduler = { todoStub() },
    /* ... */
)
```

람다에서 호출하면 `methodName`은 `...$lambda$3`처럼 컴파일러가 만든 이름이 되지만, 파일 이름과 줄 번호는 정확하다. 스텁의 목적은 "어디를 구현해야 하는지 알려 주는 것"이므로 파일과 줄만 정확하면 충분하다.

### 다만 조금 다른 것

**`@Binding`의 부재가 변경의 책임 소재를 코드에 드러낸다.** 책의 코드에서는 `@Binding`이 변경을 `Course`까지 자동으로 이어 주므로, 변경을 받아 반영하는 주체가 코드에 보이지 않는다. Compose에서는 변경이 이벤트로 올라오므로, 그 이벤트를 받아 새 상태를 만드는 주체(ViewModel이나 상태 홀더)가 코드에 명시적으로 나타난다. UI-03 정리 문서에서 안드로이드의 ViewModel은 관심사 분리보다 라이프사이클과 구성 변경 대응 때문에 둔다고 정리했는데, 그 ViewModel이 바로 이 주체의 자리에 놓인다. 따라서 이 챕터의 `CourseView`도 Compose에서는 `CourseScreen`(책의 `CourseView`)과 그 위의 `CourseRoute`(상태를 모아 주는 쪽)로 나누는 UI-03 정리 문서의 구성을 그대로 따르는 것이 자연스럽다.

**제네릭이 해결하는 문제의 종류가 다르다.** Swift에서 제네릭은 컴파일러의 요구로 어쩔 수 없이 도입한 비용이었지만, Kotlin에서는 호출부 타입을 보존하는 편의 기능이다. 따라서 같은 코드를 옮길 때 "복잡함을 가격으로 치른다"는 책의 평가가 안드로이드에서는 훨씬 가볍게 적용된다. 반대로 Kotlin에는 소급 준수가 없다는 제약이 Swift의 제네릭 문제를 대신하는 비용이 된다.

---

## Ch3와의 관계, 그리고 Ch5로

**Ch3가 그린 화살표를 Ch4가 코드로 옮긴다.** UI-03 정리 문서는 이 챕터를 "Figure 3.10의 모든 화살표를 실제 코드의 의존으로 옮기는 챕터"로 예고했다. 실제로 화살표 하나하나가 코드의 한 줄에 대응한다.

| Ch3의 화살표 | Ch4의 코드 |
|---|---|
| `CourseView` → `Course Model` (굵은 화살표) | `@Binding var course: Course` (Listing 4.1) |
| `CourseView` → UI 라이브러리의 프리미티브 | `ThumbDescriptionView`, `SubduedButton`, `CalloutView`, `TextButton` 호출 |
| `CourseView` → `ScheduleView` (기능 안의 로컬 프리미티브) | `ScheduleView(date: course.calendarEvent?.date, ...)` |
| `CourseView` → `SelectionView` (컴포넌트) | `SelectionView(elements: $course.schedule.weekly, ...)` |
| `TodoItem` → `SelectionElement` | `extension TodoItem: SelectionElement {}` (Listing 4.4) |
| UI 라이브러리 → 모델 레이어: **화살표 없음** | `SelectionView`는 `Element: SelectionElement` 제약으로만 타입을 안다 |

마지막 행이 중요하다. "UI 라이브러리에서 모델 레이어로 향하는 화살표는 하나도 없다"는 Ch3의 조건이 코드에서는 "제네릭 제약 외에 도메인을 가리키는 `import`나 타입 이름이 없다"는 모양으로 지켜졌다.

**Ch3의 분류가 구현 중의 점검 도구가 된다.** Ch3는 분류의 목적을 "모듈을 나눌 때 어느 뷰가 어느 쪽으로 가야 하는지 미리 판단하는 것"이라고 했다. 이 챕터는 거기에 용도를 하나 더한다. §4.2.1이 말하듯 분류는 **구현하면서 뷰가 우연히 다른 분류로 넘어가지 않았는지 확인하는 기준**이기도 하다. §4.7.3에서 `SelectionItemView`가 Figure 3.3의 분류와 어긋나는 사례가 나왔듯, 분류는 도해로 한 번 정하고 끝나는 것이 아니라 코드가 바뀔 때마다 다시 대조해야 한다.

**Ch2와의 관계.** Ch2의 분해가 이 챕터에서 실제 코드가 된다. §2.5.5가 `SelectionView`에서 제목과 리셋 버튼을 밖으로 밀어낸 결정이 §4.8의 "`SelectionView` 두 개 합성"을 가능하게 했고, §2.5.1의 `TextButton`과 §2.5.7의 `SubduedButton`이 §4.9.1의 두 구현으로 이어진다. §4.9의 프리미티브들은 Ch2가 분해한 결과를 실제 코드로 옮긴 것이다.

**Book 0과의 관계.** §4.2.3의 holistic 접근은 Book 0의 HDD를 뷰에 적용한 것이고, §4.3에서 모델 로직을 완료된 것으로 가정하는 태도는 HDD가 `course.schedule.weekly`를 placeholder로 남겨 둔 결정과 짝을 이룬다. 두 챕터에서 같은 사고방식이 데이터 계층과 UI 계층에 각각 쓰인 것이다.

**Ch5로 넘어가며.** 이 챕터는 `CourseView`가 "기능적 수준에서 끝났다"고 간주하기 위한 조건을 스스로 두 가지 남겼다. **누가 코스를 로드하는가**와 **내비게이션 액션을 어떻게 구현하는가**다. 첫 번째는 바로 다음 챕터(Ch5 자기 충족 기능)에서 `CourseView`가 `Course`를 전달받는 방식에서 스스로 로드하는 방식으로 바뀌는 과정으로 이어지고, 두 번째는 Ch9의 클로저와 `CourseActions` 인터페이스로 이어진다. 이 챕터가 확정한 것은 **`CourseView`가 `Course` 모델만 있으면 작동한다**는 사실이고, 다음 챕터들이 다룰 것은 **그 `Course`를 누가, 어떤 경로로 건네주는가**다.
