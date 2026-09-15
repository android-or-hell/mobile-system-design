# Delivering Reusable Views: The Art of Decomposing a Design

### UI Principle 5: Name a view after what it is, not how you use it

- 대부분의 뷰들은 기능에 대해 알 필요 없다.
- 흔히 저지르는 대표저인 실수는 뷰와 같은 타입에 너무 구체적인 이름을 붙이는 것이다.
- 이러한 사고방식 때문에 어떻게 사용되는지를 너무 과하게 의식하여 특정 유스케이스에만 최적화될 수 있다.

#### Making a reusable view

- 만약에 Tutor 정보를 포함하는 뷰를 TutorView라고 이름 짓는다면, 유스케이스에 너무 종속된다.
- TutorView를 사용하는 곳이 또 생기면 이름을 바꿀지, 동일한 뷰를 복제할지, 공통 뷰를 상속시킬지 고민하게 된다.
- 뷰의 이름에서 어떻게 보이는지를 제거하는 것이 가장 좋은 방법이다.

#### Type name versus instance name

- 문제는 뷰가 무엇인가와 뷰를 어떻게 사용하는가가 혼동되고 있다는 점이다.
- 프로그래밍 용어로 바꾸어 말하면, 타입 이름과 인스턴스 이름을 혼동하고 있는 것이다.
- 어떻게 사용되는지 명시되지 않는 이름을 떠올려야 어떤 상황에서든 사용할 수 있다.
- 뷰를 구체적으로 만들수록, 해당 뷰가 특정 기능에서만 사용될 가능성이 높아진다.

#### Expressing reusable views in code

- 코드에서는 인스턴스의 이름(타입 이름)이 아닌 tutorView나 profileView로 지어서 재사용 가능한 뷰를 표현한다.
- 목적이 들어나는 이름을 타입 자체가 아닌 타입의 인스턴스 이름으로 활용해라.

#### View primitives

```kotlin
// 1. 튜터 프로필용
val tutorView = ImageLabelView(
    image = avatar,
    title = "Caleb Davis",
    subtitle = "@CalebGuitar"
)

// 2. 일반 사용자 프로필용
val profileView = ImageLabelView(
    image = photo,
    title = "User Name",
    subtitle = "Bio"
)

// 3. 동영상 미리보기용 (subtitle이 없는 경우 null 전달 가능)
val videoPreview = ImageLabelView(
    image = videoIcon,
    title = "Guitar introduction",
    subtitle = null
)

// 4. 에러 안내용
val errorView = ImageLabelView(
    image = alertIcon,
    title = "Oops, something went wrong",
    subtitle = "Try again later"
)
```

- 용도마다 view primitive를 선언해서 사용할 수 있다.
- 프로젝트 내에서 이 뷰를 별도의 UI Library 폴더로 이동하고, 나중에는 독립된 모듈로 분리할 수 있다.

#### The road of becoming too specific

- 처음부터 TutorView나 ProfileView로 이름지었다면 파라미터도 displayName이나 handle처럼 특정 용도에 맞춘 속성이였을거고
- 다른 시나리오에서 사용하려면, 필드를 추가하여 뷰를 더 복잡하고 기능 중심적으로 만들게 된다.
- 이 과정에서 뷰의 독립성이 손상되어 타입과 호출자 사이에 암묵적인 결합이 생긴다.
- 더 좋은 점은 의존성 분리를 명확하게 해준다는 점이다.
    - UI 라이브러리에 있는 이 뷰에 Tutor 모델을 전달하는 것은 부자연스럽지만
    - TutorView와 Tutor 모델이 옆에 붙어있을 때는 둘을 연결하는 것이 이상해 보이지 않는다.

### Introducing abstractions without introducing new types

- TutorView라는 새로운 타입을 굳이 안 만들고, 팩토리 메서드의 커스텀 생성자를 사용하면 재사용성과 도메인 편의성을 모두 챙길 수 있다.

```kotlin
// tutorView는 ImageLabelView 타입이지만, displayName과 handle을 전달하여 생성(초기화)할 수 있습니다.
val tutorView: ImageLabelView = CourseUI.makeTutorView(
    avatar = avatar,
    displayName = "Caleb Guitar",
    handle = "@CalebGuitar"
)
// [코드 예시 2.3: 튜터 도메인 색깔이 입혀진 ImageLabelView를 생성하는 팩토리]
```

- Course 기능 모듈에 속해있음을 나타내기 위해 object를 확용한 네임스페이스를 사용할 수도 있다.

```kotlin
object CourseUI {

    // 튜터 도메인에 특화된 이름인 avatar, displayName, handle을 파라미터로 받습니다.
    fun makeTutorView(
        avatar: Painter,
        displayName: String,
        handle: String
    ): @Composable () -> Unit = {
        // 내부(Under the hood)에서는 여전히 범용 파라미터를 사용하여 ImageLabelView를 호출합니다.
        ImageLabelView(
            image = avatar,
            title = displayName,
            subtitle = handle
        )
    }
}
// [코드 예시 2.4: ImageLabelView를 반환하는 CourseUI 네임스페이스 팩토리]
```

#### Abstractions via aliasing

- typealias를 통해서 새로운 타입을 만들지 않고도 동일한 타입에 다른 이름을 붙일 수 있다.

```kotlin
// ImageLabelView에 TutorView라는 별칭(alias)을 부여합니다.
typealias TutorView = @Composable (modifier: Modifier) -> Unit
// 또는 데이터/뷰 매핑용 함수 타입 별칭
typealias TutorViewComposable = ImageLabelView
```

#### Making an extension

- 팩토리에서 한 번 더 나아가 확장 함수를 추가할 수도 있다.

```kotlin
// TutorView(ImageLabelView)를 확장(Extension)합니다.
// 범용 컴포저블 위에 도메인 맞춤형 오버로딩 함수를 제공하는 방식입니다.
@Composable
fun TutorView(
    avatar: Painter,
    displayName: String,
    handle: String,
    modifier: Modifier = Modifier
) {
    // 내부적으로는 더 범용적인 속성을 가진 ImageLabelView를 호출(초기화)합니다.
    ImageLabelView(
        image = avatar,
        title = displayName,
        subtitle = handle,
        modifier = modifier
    )
}
// [코드 예시 2.7: 도메인 맞춤형 TutorView 컴포저블 확장]
```

- 팩토리와 유사하지만 내부적으로 TutorView 자체에 직접 생성자를 호출하는 것처럼 해준다.
- 수많은 유스케이스를 수용하고 UI 라이브러리에 둘 수 있는 ImageLabelView는 그대로 유지되고
- 기능 모듈 내부에서는 TutorView라는 도메인 언어로 작업할 수 있다.
- 다른 팀이 해당 뷰가 포함된 UI 라이브러리를 소유/관리하고 있다면, 함부로 수정할 수 없도록 우리쪽 기능 도메인에 위치시켜야 한다.

### UI Principle 6: Don’t name a view after its styling

- 뷰가 말풍선이나 shortcut 처럼 생겼다고 Shoutout이나 TextBallon이라고 이름 붙이지 마라.
- CalloutView라는 더 범용적이고 사용 목적을 드러내지 않는 이름이 있다.
- 뷰의 설정 가능성은 정말 필요한 시점에만 확장하는 것을 권장한다.
- 필요 이상으로 뷰를 재사용 가능하게 만드느라 시간을 낭비하지 마라. 사용안해서 삭제할 수도 있다.

#### Alternative names

- 특정 이름으로 결정했다고 해서 더 나은 대안이 없다는 것은 아니다. SpeechBubble도 수용 가능한 이름이다.
- 하지만, SpeechBubble이라는 이름은 개발자에게 시각적인 말풍선을 얻게 될 것이라는 메시지를 준다. 따라서 UI 스타일링과 이름을 분리해야 한다.
- 미래에 디자인이 바뀌게 된다면, SpeechBubble이라는 이름을 선택하면, 실제로 풍선 모양이 아니기 때문에 거짓이 된다.

### UI Principle 7: Favor composition over smart views

- 여러 개의 라벨과 두 개 이상의 버튼이 있는 스케줄을 잡는 뷰가 있다고 가정해보자
- 해당 뷰는 특정 기능에 너무 지나치게 특화되어 있기 때문에, 다른 곳에서 사용할 가능성이 적다.
- 따라서 억지로 재사용 가능한 뷰로 만들지 않고 ScheduleView로 유지하되, 하위 컴포넌트를 분해하여 재사용 가능하게 한다.

#### Breaking apart the view

- 버튼을 떼더라도 목적이 드러나지 않도록 AccessoryButton으로 만들고, 버튼의 경우에도 텍스트만 사용할 것을 나타내도록 TextButton이라고 명명한다.
- 이제 ScheduleView는 해당 공용 컴포넌트들을 조립하여 사용한다.

#### Decomposing the todo list

- Todo list가 있을 때 이는 본질적으로 할 일 목록이 아닌 우리가 할 일 목록으로 쓰고 있는 것이다.
- 내 기능 영역에 갇혀 생각하면 기능에 종속적인 UI를 만들기 쉽다.
- 뷰를 더 일반적이고 추상적인 시각으로 바라보아 SelectionView라는 이름을 붙여줄 수 있다.

#### Domain naming versus view naming

- TodoList 도메인은 비즈니스 목적을 달성하기 위한 도구로 SelectionView를 사용할 뿐이다.
- 따라서, 비즈니스 도메인의 이름은 도메인 이름으로 그대로 유지하고, 뷰는 그대로 두어야 한다.

#### Handling inconsistent UI

- 일관성 없는 UI를 다룰 때 여러 옵션 파라미터로 집어넣어 설정 가능하도록 만들 수 있다.
- 하지만 이렇게 만들다 보면 온갖 복잡한 조건문 로직이 붙고 복잡성 및 유지보수 비용이 증가한다.
- 또한, 원래 의도와는 다르게 뷰를 사용할 수도 있고 불필요한 기능이 있다면 재사용성도 감소한다.

#### Avoid delivering for hypothetical scenarios

- 지금 당장에 필요하지 않은 기능까지 미리 구현해서 제공할 필요는 없다.
- 가상의 시나리오에 대비하느라 뷰를 복잡하게 만드는 것보다, 지금 단순하게 유지함으로써 얻는 이점이 훨씬 크다.

#### Button Views

- 테두리가 있는 버튼이라고 해서 BorderButton이라고 부를 수 있지만, 이는 시각적 외형을 이름에 담은 것이다.
- 외형보다 기능과 의미의 관점에서 생각하여 SubducedButton 같은 네이밍을 사용하자

#### What we covered

- UI 원칙 5 : 타입 이름은 본질에 따라 짓고, 인스턴스 이름은 용도에 따라 지어라
- UI 원칙 6 : 뷰의 시각적 스타일링을 기반으로 이름을 짓지 마라
- UI 원칙 7 : Smart View 대신 Composition을 지향하라
- View primitives : 특정 비즈니스 도메인이나 세부 지식을 모르는 단순하고 재사용 가능한 뷰들을 정의하라
- 매번 억지로 새로운 타입을 만들지 않고 팩토리 함수, typealias, 확장 함수를 통해 도메인에 맞는 사용성을 제공하라
- 뷰의 이름을 범용적으로 지을수록 공용 UI 라이브러리에 둘 수 있고, 특정 기능에 종속된 뷰는 도메인 모듈에 두어야 한다.
- 혹시 나중에 필요할지도 모른다는 생각해 불필요한 기능을 만드는 것보다 단순하게 유지하는 것이 이점이 크다.