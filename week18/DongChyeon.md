### Considering various approaches

- 화면을 사전에 분해했다고 해서, 언제나 실제 구현에 들어가기 전부터 모든 서브뷰들을 잘게 쪼개두어야 하는 것은 아니다.

#### A top-down approach

- 탑 다운 접근법은 화면을 한 덩어리로 구현한 뒤, 나중에 필요에 따라 하위 뷰들을 추출하는 방식
- 선언형 UI와 달리 명령형 UI 뷰에서는 서브 뷰를 추출하는 것이 더 복잡함
- 처음부터 한 덩어리로 구현하면, 비즈니스 로직과 UI 코드가 결합될 위험이 있음

#### A bottom-up approach

- 작은 view primitive 부터 구현하고, 최상위 뷰로 조립하는 방식
- 초기 작업에 드는 비용이 크지만 비즈니스 로직이 컴포넌트로 침투하는 것을 차단할 수 있음

#### A holistic approach

- 하위 뷰들을 먼저 선언해두되, 세부 구현없이 플레이스홀더로 대체
- 최상위 화면의 배치가 끝나면 하위 뷰들을 완성형 코드로 교체

#### A mixed approach

- 플레이스홀더를 사용하되, 러프한 형태로 구현하여 화면이 온전히 살아 움직이고 데이터가 올바르게 바인딩되는지 확인
- 미리 정의된 플랫폼 표준 폰트나 색상 등을 활용하여 빠르게 비슷하게 구현

### How we’ll load the data

- 뷰가 스스로 데이터를 로딩하는 대신, 외부에서 전달받은 모델을 렌더링하도록 지원

#### Starting with CourseView

- Compose에서는 외부 파라미터로 Course를 넘겨줌으로써 뷰를 그에 따라 표시

### Simplifying view code

- 선언형 뷰를 사용하면 하위 요소를 쉽게 떼어나고 이를 이전시킬 수 있음

### Implementing SelectionView

- SelectionView는 비즈니스 모델을 직접 알지 못하며, SelectionElement라는 인터페이스를 통해 데이터를 표현
- 이 인터페이스를 통해 SelectionView는 개별 행을 나타내는 SelectionViewItem 타입들의 목록을 렌더링
- 도메인 모델이 SelectionElement 를 준수하도록 선언되면, SelectionView는 아무런 수정 없이 TodoItem 배열을 렌더링 가능

#### Defining SelectionView

- Compose 에서는 UDF를 원칙으로 삼기 때문에 하위 뷰가 상위 상태 객체를 직접 수정하지 않고, 상태 호이스팅 방식 사용
- SwiftUI에서는 이를 @Binding 을 통해서 전달함

#### Compile-time versus runtime

- 단순 타입 (SharedElement) 로 넘기면 컴파일 타임에 구체 타입을 제대로 알지 못함
- 따라서 <Element: SelectionElement> 처럼 제네릭 제약 조건을 걸어서 해결할 수 있음

#### Implementing SelectionItemView

- Binding된 요소를 표시하고, 클로저를 주입함으로써 업데이트해주면 SelectionView까지 변경사항이 전파됨
- Compose도 비슷하게 UDF에 맞춰 불변 인터페이스와 상태 변경 람다 콜백 사용 가능

### Implementing the todo list

- Todo list가 주간 일정, 일일 일정 섹션 2가지로 나눠져 있더라도 SelectionView 인스턴스를 조합하는 것으로 구현할 수 있다
- 이를 TodoSection으로 분리하면 깔끔하게 유지 가능하다.

### Implementing view primitives

- 전체 구현을 위해 하위 뷰 프리미티브의 가능한 상태를 파악해서 파라미터로 전달받도록 구성
- 뷰 내부에서 null 처리와 같은 로직을 덜어내기 위함

#### View primitives in the UI Library

- 나머지 primitive들은 공용 디자인 시스템에 상주하는 초경량 컴포넌트

### Looking at our progress

- 여태까지의 구현으로 디자인과 완벽히 동일하지는 않지만, 러프한 구현을 확인할 수 있음

#### Preparing for navigation

- 특정 화면으로 이동시키는 지점은 개발 속도를 유지하기 위해, 실제 화면 전환 로직은 뒤로 미룰 수 있음
- 대신 이러한 전환 액션이 실행될때마다 호출할 메서드 이름, 파일, 라인 번호를 남기도록 둘 수 있음

### What we covered

#### Strategic implementation approaches

- Top-down 방식: 한 화면에 몰아서 짜고 서브뷰를 추출하기에 유리하지만, 서브뷰가 기능에 결합될 위험이 큼
- Bottom-up 방식: 하위 컴포넌트부터 작성하므로 기능과 서브뷰 사이에 API 분리가 용이하지만, 지엽적인 디테일 때문에 완성 속도를 늦출 수 있음
- 이 둘을 혼합하여, 하위 컴포넌트부터 구현하되 러프한 구현만 하도록 할 수 있음

#### Framework-agnostic patterns

- 패러다임이 다르더라도 명령한 컴포넌트와 선언형 컴포넌트는 유사한 구조로 설계할 수 있음
- 인터페이스 기반 설계 패턴을 특정 UI 프레임워크나 기능에 종속되지 않은 공통 컴포넌트를 만듦

#### Action-based architecture

- 뷰에 람다 콜백을 주입함으로써 비즈니스 로직과 화면과의 결합을 끊어내고 기능은 유지
- 선언형 코드는 오토 바인딩이 존재하지 않아 클로저로 공백을 메워야 함
- MVI처럼 통일된 액션 파이프라인 형태로 만들 수 있음

#### Pragmatic trade-offs

- 개발 모멘텀을 지키기 위해서, 기능 동작을 최우선으로 완성하고 나머지 디테일은 나중에 다듬어야 함
- 작동하는 기능을 전달하기 위해 속도와 완벽 주의 사이에 균형을 맞춰야 함
- 최상위 피처 뷰의 바디를 높은 추상화 수준으로 유지할 수록 화면 전체의 구조를 추론하기 쉬움
- 메인 뷰를 상위 수준으로 깔끔하게 유지하기 위해, 동일 파일 내의 로컬 private 서브뷰로 만드는 것을 적극적으로 고려해야 함
    - 이후, 더 성숙하거나 로직이 들어가면 독립된 뷰로 추출