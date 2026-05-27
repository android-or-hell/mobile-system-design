# 3. UI Library Fundamentals, Part II: Spacing, Icons, and Shadows 요약

## 3.1 Spacing

### 문제: Magic number와 spacing inconsistency

- `spacing`은 UI 요소 사이의 간격을 의미한다.
- 개발자는 디자인을 보고 spacing 값을 직접 측정하는 경우가 많다.
- 이때 `12`, `14`, `15` 같은 임의의 값을 그대로 코드에 넣기 쉽다.
- 이런 값은 흔히 `magic number`가 된다.
- 단일 화면에서는 magic number가 큰 문제처럼 보이지 않을 수 있다.
- 하지만 앱 전체로 확장되면 불일치가 빠르게 쌓인다.
- 어떤 개발자는 `12pt`를 쓰고, 다른 개발자는 `14pt`를 쓸 수 있다.
- 같은 개발자도 상황에 따라 서로 다른 값을 사용할 수 있다.
- 디자이너의 미세한 실수나 측정 오차도 spacing 불일치를 만든다.
- 작은 차이가 반복되면 UI가 덜 polished하게 느껴진다.
- 유지보수도 어려워진다.
- 디자이너가 전체 spacing을 조금 더 넓히고 싶어 할 수 있다.
- 그 경우 코드 전체에서 `12`, `10`, `15` 같은 값을 찾아 수정해야 한다.
- 이 방식은 지루하고 오류가 나기 쉽다.
- hard-coded spacing은 디바이스 크기 변화에도 약하다.
- `12px`은 휴대폰에서는 괜찮아 보일 수 있다.
- 하지만 태블릿에서는 너무 좁아 보일 수 있다.
- 매 layout마다 spacing 값을 새로 결정하는 것은 비효율적이다.
- 눈대중이나 pixel 측정에 의존하는 방식은 장기적으로 취약하다.
- 더 나은 방식은 spacing system을 만드는 것이다.

### 3.1.1 Spacing primitives

- spacing 문제를 해결하기 위해 spacing primitive를 정의한다.
- spacing primitive는 design system에서 사용하는 원시 간격 값이다.
- 예를 들어 다음과 같은 scale을 만들 수 있다.

```plain text
4, 8, 16, 24, 32, 40, 48, 64, 80
```

- 이 scale은 앱 전체에서 허용되는 spacing 후보가 된다.
- primitive scale은 arbitrary decision을 줄인다.
- 개발자는 매번 새 값을 만들지 않고 정해진 scale에서 선택한다.
- spacing scale은 보통 grid system과 잘 맞는 값으로 구성한다.
- `4`, `8`, `16` 같은 값은 UI layout에서 자주 쓰이는 단위다.
- 값이 커질수록 아주 세밀한 차이는 덜 중요해진다.
- 큰 간격에서는 proportionality와 balance가 더 중요하다.
- `10`이나 `12` 같은 중간 값을 추가할 수도 있다.
- 하지만 너무 촘촘한 scale은 선택을 어렵게 만든다.
- `10`과 `12`의 차이는 UI에서 명확히 구분하기 어렵다.
- `8`과 `16`의 차이는 상대적으로 더 명확하다.
- 너무 작은 incremental difference는 visual clutter를 유발할 수 있다.
- spacing scale은 디자이너와 함께 정해야 한다.
- 개발자가 임의로 scale을 정하기보다 design system의 의도를 반영해야 한다.
- 큰 spacing 값은 항상 primitive로 필요하지 않을 수 있다.
- 매우 큰 간격은 container, centering, layout structure로 해결할 수 있다.
- 즉 layout 자체가 자연스럽게 공간을 만들 수도 있다.

### 3.1.2 Spacing primitives 적용

- spacing primitive는 코드에서 상수 목록으로 정의할 수 있다.
- Swift에서는 `Space` 같은 struct를 만들 수 있다.
- `s4`, `s8`, `s16`처럼 static value를 둘 수 있다.
- 숫자로 property name을 만들 수 없으므로 `s` prefix를 붙인다.

```plain text
struct Space {
    static let s4: CGFloat = 4
    static let s8: CGFloat = 8
    static let s16: CGFloat = 16
    static let s24: CGFloat = 24
    static let s32: CGFloat = 32
    static let s40: CGFloat = 40
    static let s48: CGFloat = 48
    static let s64: CGFloat = 64
    static let s80: CGFloat = 80
}
```

- 이렇게 하면 `Space.s24`처럼 spacing 값을 참조할 수 있다.
- 핵심은 직접 `24`를 쓰지 않고 named constant를 쓰는 것이다.
- primitive를 쓰면 허용되지 않은 값이 드러난다.
- 예를 들어 디자인에서 `12pt`가 측정되었는데 scale에 `12`가 없을 수 있다.
- 이 경우 `12`는 시스템상 allowed value가 아니다.
- 개발자는 가까운 값인 `Space.s8` 또는 `Space.s16` 중 선택해야 한다.
- 선택이 애매하면 디자이너와 확인하는 것이 좋다.
- `20pt`가 필요하면 `Space.s16` 또는 `Space.s24` 중 선택할 수 있다.
- 시각적 hierarchy상 더 큰 값이 필요하면 `Space.s24`를 선택한다.
- 이런 방식은 spacing consistency를 강화한다.
- padding, margin, gap에 같은 primitive scale을 적용할 수 있다.
- 예를 들어 `14pt`를 측정했다면 scale에 없는 값임을 알 수 있다.
- 이때 근접한 `Space.s16`을 쓰는 식으로 정리할 수 있다.
- primitive는 spacing의 shared foundation 역할을 한다.
- 하지만 primitive만으로는 의미를 충분히 표현하지 못한다.
- 그래서 semantic spacing이 필요하다.

### 3.1.3 Semantic spacing

- primitive를 직접 사용하는 방식에는 한계가 있다.
- `Space.s16`은 단순한 값일 뿐이다.
- 이 값이 왜 사용되는지 설명하지 않는다.
- 색상에서도 `Palette.red`는 의미가 부족하다.
- typography에서도 font를 직접 지정하면 consistency가 깨지기 쉽다.
- spacing도 마찬가지로 semantic layer가 필요하다.
- semantic spacing은 primitive spacing에 의미 있는 이름을 붙인다.
- 예를 들어 `xxs`, `xs`, `sm`, `md`, `lg`, `xl` 같은 이름을 사용할 수 있다.

```plain text
struct Spacing {
    static let xxs = Space.s4
    static let xs  = Space.s8
    static let sm  = Space.s16
    static let md  = Space.s24
    static let lg  = Space.s32
    static let xl  = Space.s40
    static let xxl = Space.s48
    static let x3l = Space.s64
    static let x4l = Space.s80
}
```

- `Spacing.md`는 단순히 `24`라는 값보다 더 의미가 있다.
- semantic name은 code readability를 높인다.
- call site에서 “중간 크기의 spacing”이라는 의도가 드러난다.
- primitive 값은 내부 구현으로 숨길 수 있다.
- 예를 들어 `Spacing.md = Space.s24`처럼 매핑한다.
- 나중에 `md` 값을 바꾸고 싶으면 한 곳만 수정하면 된다.
- call site 전체를 찾아다닐 필요가 줄어든다.
- semantic spacing은 유지보수를 쉽게 만든다.
- 디자인 변경이 앱 전체에 더 안전하게 전파된다.
- primitive는 raw value의 안정성을 제공한다.
- semantic은 의미와 변경 용이성을 제공한다.
- 둘을 함께 쓰면 consistency와 flexibility를 동시에 얻을 수 있다.

### 3.1.4 Dynamic spacing

- static semantic spacing에도 한계가 있다.
- `Spacing.md`가 항상 하나의 값으로만 resolve되면 상황 대응이 어렵다.
- 휴대폰과 태블릿은 같은 spacing이 다르게 느껴질 수 있다.
- 버튼 사이 spacing은 휴대폰에서는 좁게, 태블릿에서는 넓게 둘 수 있다.
- 이를 위해 dynamic spacing이 필요하다.
- iOS에서는 size class를 기준으로 spacing을 바꿀 수 있다.
- `compact`는 보통 작은 width 환경을 의미한다.
- `regular`는 더 넓은 화면 공간을 의미한다.
- SwiftUI에서는 environment에서 `horizontalSizeClass`를 읽을 수 있다.
- size class에 따라 padding 값을 다르게 반환할 수 있다.
- view 안에서 직접 분기하면 동작은 가능하다.
- 하지만 앱 전체에서 반복되면 중복이 많아진다.
- 각 view가 breakpoint logic을 직접 갖는 것은 부담스럽다.
- 그래서 dynamic spacing logic을 library 내부로 넣는 것이 낫다.
- UI library가 환경에 맞는 spacing을 반환하게 만들 수 있다.
- 그러면 view는 spacing policy를 몰라도 된다.
- view는 단지 `Spacing.md.dynamic(...)`처럼 사용할 수 있다.
- dynamic spacing은 adaptive UI를 쉽게 만든다.
- 특히 mobile과 tablet을 동시에 고려할 때 유용하다.
- spacing system은 static value뿐 아니라 adaptive behavior도 포함할 수 있다.

### 3.1.5 Dynamic spacing 구현

- dynamic spacing은 `dynamic(for:)` 같은 method로 구현할 수 있다.
- 이 method는 현재 size class를 인자로 받는다.
- size class가 `regular`이면 더 큰 spacing을 반환할 수 있다.
- size class가 `compact`이면 기본 spacing을 반환한다.
- 각 semantic spacing은 기본 value와 offset을 가질 수 있다.
- 예를 들어 `Spacing(value: Space.s24, offset: Space.s8)`처럼 정의한다.
- regular 환경에서는 `value + offset`을 사용한다.
- compact 환경에서는 `value`만 사용한다.
- 이렇게 하면 spacing마다 device adaptation rule을 가질 수 있다.
- 작은 spacing은 작은 offset을 가질 수 있다.
- 큰 spacing은 더 큰 offset을 가질 수 있다.
- 이 방식은 static과 dynamic spacing을 모두 지원한다.
- 필요한 곳에서는 `.value`로 static value를 사용할 수 있다.
- 필요한 곳에서는 `.dynamic(for:)`로 adaptive value를 사용할 수 있다.
- 다만 매번 size class를 넘기는 것은 번거로울 수 있다.
- 이를 줄이기 위해 SwiftUI extension을 만들 수 있다.
- 예를 들어 custom padding modifier를 제공할 수 있다.
- extension 내부에서 environment를 읽고 dynamic spacing을 적용한다.
- 그러면 call site는 더 간단해진다.
- 결과적으로 spacing rule은 중앙화되고 view code는 깔끔해진다.

## 3.2 Icons

### 문제: Raw asset 직접 사용

- 다음 주제는 icons다.
- 앱에서는 icon asset을 raw name으로 직접 참조하는 경우가 많다.
- 예를 들어 `arrow` 같은 asset name을 그대로 사용할 수 있다.
- 이런 방식은 처음에는 간단하다.
- 하지만 앱이 커질수록 유지보수 문제가 생긴다.
- raw asset name은 사용 목적을 충분히 설명하지 않는다.
- 같은 `arrow`가 여러 맥락에서 쓰일 수 있다.
- navigation back button인지 disclosure indicator인지 알기 어렵다.
- asset 자체를 바꾸면 모든 사용처가 함께 바뀔 수 있다.
- 특정 맥락에서만 icon을 바꾸고 싶을 때 문제가 된다.
- asset 이름을 바꾸면 app-wide find and replace가 필요할 수 있다.
- raw asset은 variant를 표현하기도 어렵다.
- 예를 들어 accessibility 상황에서 다른 icon을 쓰고 싶을 수 있다.
- primitive icon과 semantic icon을 구분하면 이런 문제를 줄일 수 있다.
- primitive icon은 asset의 appearance를 기준으로 이름 붙인다.
- 예를 들어 `lightBulb`처럼 생김새를 기준으로 명명한다.
- `userHelpInfo`처럼 사용처를 기준으로 primitive asset을 명명하면 재사용성이 낮아진다.
- appearance-based naming은 asset reuse를 돕는다.
- semantic icon은 사용 목적을 기준으로 이름 붙인다.
- 예를 들어 `dismissIcon`, `warning`, `navigationBarBackButton` 같은 이름이 가능하다.

### 3.2.1 Semantic icons

- semantic icon은 icon의 meaning과 purpose를 추상화한다.
- call site는 어떤 asset이 쓰이는지 몰라도 된다.
- call site는 의미 있는 icon API만 사용한다.
- 예를 들어 warning 상황에서는 `Icons.warning`을 사용할 수 있다.
- 내부적으로는 triangle, exclamation mark, 다른 asset을 매핑할 수 있다.
- 나중에 warning icon을 바꿔도 call site를 수정할 필요가 적다.
- semantic icon은 icon replacement를 안전하게 만든다.
- 특정 목적의 icon을 앱 전체에서 일관되게 유지할 수 있다.
- semantic icon은 dynamic behavior도 지원하기 쉽다.
- 예를 들어 accessibility setting에 따라 다른 icon variant를 제공할 수 있다.
- light/dark mode나 platform 차이에 따른 icon 선택도 가능하다.
- semantic icon은 design system의 의도를 코드에 반영한다.
- primitive icon은 asset catalog의 raw material에 가깝다.
- semantic icon은 product/UI context에 가까운 API다.
- 둘을 구분하면 asset 관리와 UI 의도 표현이 분리된다.
- 아이콘도 색상이나 spacing처럼 추상화할 수 있다.
- 다만 모든 icon use case를 미리 알 수는 없다.
- 그래서 semantic icon만 강제하는 것은 현실적이지 않을 수 있다.
- 초기에는 primitive icon 접근도 필요할 수 있다.
- 핵심은 무분별한 raw asset 사용을 줄이는 것이다.

### 3.2.2 Semantic icon과 primitive icon 혼용

- semantic icon과 primitive icon은 함께 사용할 수 있다.
- 모든 icon 사용처를 upfront로 semantic하게 정의하기는 어렵다.
- 새로운 feature에서는 아직 의미가 확정되지 않은 icon이 있을 수 있다.
- 이런 경우 primitive icon을 직접 사용할 수 있게 하는 escape hatch가 필요하다.
- 다만 재사용되거나 의미가 분명해지면 semantic icon으로 승격하는 것이 좋다.
- primitive icon은 flexibility를 제공한다.
- semantic icon은 maintainability를 제공한다.
- 둘의 균형이 중요하다.
- icon asset 이름은 appearance 기준으로 유지하는 것이 좋다.
- semantic API 이름은 use case 기준으로 짓는 것이 좋다.
- 예를 들어 asset은 `arrowLeft`일 수 있다.
- semantic name은 `navigationBack`일 수 있다.
- 이렇게 하면 같은 asset을 여러 semantic context에서 재사용할 수 있다.
- 나중에 특정 semantic context만 다른 asset으로 바꿀 수도 있다.
- 이 구조는 icon 변경의 blast radius를 줄인다.
- 코드 리뷰에서도 icon 사용 의도가 더 명확해진다.
- raw string asset name 사용은 줄이는 것이 좋다.
- library를 통해 icon을 formalize하는 것이 유지보수에 유리하다.
- icon system은 단순한 asset wrapper가 아니라 semantic mapping layer다.
- 좋은 icon abstraction은 consistency와 adaptability를 동시에 제공한다.

## 3.3 Shadows

### 문제: Shadow inconsistency

- shadows는 UI depth와 emphasis를 표현하는 데 사용된다.
- 하지만 shadow도 임의로 쓰면 inconsistency가 생긴다.
- 어떤 개발자는 radius `4`를 쓰고, 다른 개발자는 radius `8`을 쓸 수 있다.
- offset, opacity, color가 제각각이면 UI가 산만해진다.
- shadows도 앱 전체에서 표준화할 필요가 있다.
- 대부분의 경우 `small`, `medium`, `large` 정도의 shadow variant면 충분하다.
- `ShadowStyle` 같은 struct로 shadow style을 정의할 수 있다.
- shadow style은 radius, x offset, y offset, opacity 등을 포함할 수 있다.
- 중요한 것은 exact implementation보다 standardization이다.
- 일관된 shadow set을 제공하면 developer choice가 줄어든다.
- view에서는 custom `applyShadow()` 같은 method를 사용할 수 있다.
- 이 method는 `ShadowStyle`을 받아 실제 SwiftUI `shadow()` modifier에 적용한다.
- 이렇게 하면 shadow 사용 API가 통일된다.
- 다만 SwiftUI의 기본 `.shadow()`를 직접 사용할 수 있다는 문제가 남는다.
- 개발자가 custom shadow system을 우회할 수 있기 때문이다.
- 이를 막기 위해 linting을 사용할 수 있다.
- linter는 `.shadow()` 사용 시 warning이나 error를 낼 수 있다.
- 대신 `applyShadow()`를 쓰도록 안내할 수 있다.
- 이는 code review에서 반복적인 nitpick을 줄인다.
- shadow consistency를 시스템적으로 강제할 수 있다.

### 3.3.1 Custom shadow method

- `applyShadow()`는 View extension으로 만들 수 있다.
- 이 method는 내부적으로 shadow style의 속성을 읽는다.
- radius, x, y, opacity를 native shadow modifier에 전달한다.
- 개발자는 raw shadow parameter를 직접 지정하지 않아도 된다.
- `applyShadow(.small)`처럼 의미 있는 API를 사용할 수 있다.
- custom method는 shadow usage를 더 readable하게 만든다.
- small, medium, large 같은 semantic shadow가 기본값이 된다.
- custom values가 필요하면 예외적으로 `ShadowStyle`을 직접 초기화할 수 있다.
- 그래도 built-in `.shadow()`보다 custom method를 쓰는 것이 좋다.
- custom method를 통하면 미래의 shadow policy 변경도 적용되기 때문이다.

### 3.3.2 Shadows와 dark mode

- shadow를 중앙화하면 dark mode 대응도 쉬워진다.
- dark mode에서는 일반적인 shadow가 어색할 수 있다.
- 어두운 배경 위에서는 shadow가 잘 보이지 않거나 의미가 약해진다.
- 경우에 따라 subtle glow를 쓸 수도 있다.
- 하지만 대부분은 dark mode에서 shadow를 생략하는 편이 더 나을 수 있다.
- 모든 사용처에서 shadow를 수동 제거하는 것은 비효율적이다.
- 중앙화된 `applyShadow()` 안에서 dark mode 여부를 판단하면 된다.
- dark mode면 shadow를 렌더링하지 않는다.
- light mode면 기존 shadow를 렌더링한다.
- SwiftUI에서는 environment를 사용해 color scheme을 확인할 수 있다.
- `applyShadow()` 자체가 environment를 직접 받지 못하면 ViewModifier를 만들 수 있다.
- `ShadowModifier` 같은 modifier가 dark mode logic을 담당한다.
- custom API는 이런 내부 구현을 call site에서 숨겨준다.
- call site는 여전히 `applyShadow()`만 사용하면 된다.
- 즉 semantic shadow는 mode-specific behavior를 흡수할 수 있다.

### 3.3.3 Shadows에는 primitive가 꼭 필요하지 않음

- 이 장에서는 여러 UI 요소를 primitive와 semantic으로 나누었다.
- 하지만 shadow는 꼭 primitive layer를 별도로 노출할 필요가 없다.
- 개발자가 offset, blur, opacity를 직접 자주 다루는 경우는 많지 않다.
- 오히려 `small`, `medium`, `large` 같은 semantic shadow가 더 유용하다.
- shadow primitive를 세부적으로 노출하면 불필요한 flexibility가 생길 수 있다.
- 그 flexibility는 inconsistency로 이어질 수 있다.
- 그래서 shadow는 semantic 중심 API만 제공해도 충분하다.
- 예외가 필요하면 custom `ShadowStyle` initializer를 escape hatch로 제공한다.
- 그래도 기본 경로는 semantic shadow를 사용하도록 유도한다.
- 이 판단은 UI 요소마다 달라질 수 있다.
- 더 많은 제어가 필요하면 primitive를 제공한다.
- 더 강한 일관성이 필요하면 semantic UI element만 제공한다.
- 이 heuristic은 다른 UI 요소에도 적용할 수 있다.
- 예를 들어 border, radius, opacity, elevation에도 같은 판단이 필요하다.
- 핵심은 추상화 수준을 목적에 맞게 선택하는 것이다.

## 3.4 Conclusion

- fonts, colors, spacing, icons, shadows는 UI의 큰 부분을 차지한다.
- 하지만 UI library가 다룰 수 있는 요소는 이것으로 끝나지 않는다.
- borders도 system화할 수 있다.
- corner radius도 system화할 수 있다.
- opacity도 system화할 수 있다.
- elevation, blur, grid, layout도 관리 대상이 될 수 있다.
- z-index와 motion animation도 design system 관점에서 다룰 수 있다.
- 대부분의 UI 요소는 primitive와 semantic이라는 두 관점으로 생각할 수 있다.
- primitive/semantic 분리는 단순하지만 실용적인 mental model이다.
- 이 모델은 작은 UI decision을 체계적으로 만들도록 돕는다.
- 예를 들어 `FF4300` 같은 색상을 누가 직접 접근해야 하는지 판단할 수 있다.
- `bananaYellow` 같은 색상이 dark mode를 지원해야 하는지도 판단할 수 있다.
- icon naming에서 `arrow`와 `navigationBarBackButton` 중 무엇이 적절한지도 판단할 수 있다.
- 이런 micro-decision이 쌓여 codebase의 품질을 결정한다.
- UI library가 지저분하면 앱의 모든 UI도 지저분해질 가능성이 높다.
- 그래서 UI library는 특별히 신경 써서 관리해야 한다.
- 이 장에서 만든 foundation은 더 큰 design system으로 확장될 수 있다.
- 다음 단계는 UI library를 broader design system에 통합하는 것이다.
- design system은 단순한 코드 모음보다 넓은 개념이다.
- 하지만 잘 정리된 UI library가 그 출발점이 된다.

# 4. Migrating from Legacy UI to Semantic UI 요약

## 핵심 개요
- 이 장은 기존 `legacy UI`를 새 `semantic UI` 체계로 migrate하는 방법을 다룬다.
- 지난 장들에서 만든 colors, spacing, fonts, shadows, icons 기반 UI library를 실제 codebase에 적용하는 단계다.
- 새 UI system은 더 consistent하고 maintainable하며 design system으로 확장하기 좋다.
- 하지만 migration은 단순한 code replacement가 아니라 team adoption 문제다.
- Feature engineers에게 migration은 좋은 변화이면서도 추가 작업이다.
- 따라서 migration은 technical strategy와 psychological strategy를 함께 설계해야 한다.
- 핵심 목표는 new code가 new UI를 쓰게 만들고, legacy UI는 점진적으로 줄이며, 팀이 자연스럽게 adopt하도록 돕는 것이다.

## 4.1 Why other engineers aren’t always excited about new libraries
- 새 UI library를 만든 사람에게는 semantic UI가 명백히 더 좋아 보인다.
- 하지만 다른 engineers에게는 추가 migration work로 느껴질 수 있다.
- Feature engineers는 이미 deadline, release pressure, feature work를 감당하고 있다.
- 새 UI system이 생기면 곧 ship하려는 brand-new feature도 갑자기 `legacy`처럼 보일 수 있다.
- Migration owner는 사람들이 변화에 반대하는 것이 아니라 바쁘고 기존 방식에 익숙하다는 점을 이해해야 한다.
- Migration은 “좋은 system을 만들었으니 따라오라”가 아니라 “부담 없이 옮겨갈 수 있게 돕는 과정”이어야 한다.

## 4.2 In this chapter
- 이 장은 entire app을 새 standardized UI system으로 migrate하는 방법을 다룬다.
- New UI system 도입에는 code level과 psychological level 모두에서 friction이 생긴다.
- 먼저 new code와 legacy code를 다르게 다루는 방식을 설명한다.
- 이후 existing UI를 점진적으로 migrate하는 방법을 다룬다.
- 그 다음 팀이 migration에 동참하도록 만드는 adoption 전략을 살펴본다.
- 마지막으로 per-view가 아니라 app-wide migration이 필요한 경우를 다룬다.

## 4.3 What the migration looks like
- Migration은 inline UI values를 semantic values로 바꾸는 작업이다.
- 예를 들어 raw color, RGB literal, image asset string을 semantic API로 교체한다.
```plain text
view.backgroundColor = UIColor.white
label.textColor = UIColor(red: 0.2, green: 0.2, blue: 0.2, alpha: 1.0)
imageView.image = UIImage(named: "icon_exclamation_mark")
```
```plain text
view.backgroundColor = Color.primaryBackground
label.textColor = Color.textPrimary
imageView.image = Icon.warning
```
- 모든 migration이 single-line replacement는 아니다.
- Shadows처럼 여러 줄의 layer 설정을 `view.applyShadow(Shadow.medium)` 같은 하나의 semantic call로 바꿀 수 있다.
- Text style도 여러 줄의 font/color 설정을 `.companyFont(.body)` 같은 configuration으로 줄일 수 있다.
- Semantic UI는 code를 줄이고, intent를 명확히 하고, consistency를 높인다.
- 실제 project에는 이런 코드가 수천 줄 있을 수 있으므로 strategy가 필요하다.

## 4.4 Preparing the team
- Migration에서는 current code와 new code를 다르게 다뤄야 한다.
- Existing legacy code는 당장 모두 고치기 어렵다.
- 하지만 new code가 계속 legacy UI를 사용하면 migration은 절대 끝나지 않는다.
- 먼저 new code가 legacy UI를 만들지 않도록 막아야 한다.
- 갑작스러운 강제는 resistance를 만들기 때문에, context 공유와 적응 시간이 필요하다.

### 4.4.1 Educating the team
- 팀에게 어떤 변화가 올지 설명하고, 언제부터 새 semantic UI를 사용해야 하는지 알려야 한다.
- 처음부터 required로 강제하기보다 strongly preferred라고 안내하는 것이 좋다.
- 작은 semantic color나 shadow부터 시도하게 하며 curiosity와 momentum을 만든다.
- 나중에는 legacy UI가 포함된 PR이 block될 수 있다는 점을 가볍게 암시할 수 있다.
- Old system을 사용했다고 punished된다고 느끼지 않게 해야 한다.

### 4.4.2 Be approachable and helpful
- Migration owner는 approachable해야 한다.
- Documentation, migration guide, before/after examples를 제공해야 한다.
- Presentations, code samples, shared examples도 도움이 된다.
- Slack channel이나 질문 공간을 만드는 것도 좋다.
- Missing UI elements, bugs, unclear naming은 migration 중 반드시 나오므로 designer와 다시 align할 준비가 필요하다.

## 4.5 Ensuring new code uses new UI
- 팀이 새 system을 이해했다면 new code가 new UI를 사용하도록 만들어야 한다.
- 사람이 PR마다 지적하면 nagging처럼 느껴질 수 있으므로 tooling을 활용하는 것이 좋다.
- IDE나 linter가 알려 주면 개인적인 지적으로 느껴지지 않는다.
- Linter는 raw colors, raw spacing, old UI API 사용을 감지하는 데 쓸 수 있다.

### 4.5.1 Using a linter
- Linter는 custom rules를 만들 수 있다.
- 예를 들어 default `Color.blue` 사용을 금지하고 semantic color 사용을 안내할 수 있다.
- 좋은 linter message는 “틀렸다”가 아니라 “무엇을 대신 쓰면 되는지” 알려 줘야 한다.
- 대체 API와 documentation link를 제공하면 adoption이 쉬워진다.
- 이전에 legacy UI를 사용한 것은 개발자의 잘못이 아니므로 wording이 중요하다.

### 4.5.2 Soft rollout
- Linter를 처음부터 error로 설정하지 않는 것이 좋다.
- 처음에는 warning으로 시작해 기대치를 알리되 build나 merge를 막지 않는다.
- 팀이 새 system에 익숙해지면 warning을 error로 바꿀 수 있다.
- CI에서 warnings as errors 설정을 사용할 수도 있다.
- 갑작스러운 강제보다 staged rollout이 resistance를 줄인다.

### 4.5.3 Lint UI rules for new code only
- Linter를 전체 codebase에 바로 적용하면 수백 개의 legacy violation이 쏟아질 수 있다.
- Existing legacy code가 아직 migrate되지 않은 것은 받아들여야 한다.
- 대신 new code와 changed code에만 linter를 적용하는 것이 좋다.
- PR diff에 대해서만 lint를 실행하면 새로 작성한 코드만 기준을 지키면 된다.
- Existing debt는 천천히 갚되, new debt는 추가하지 않는 전략이다.

### 4.5.4 Preventing regressions
- 한 번 migrate된 view가 다시 legacy UI를 쓰게 되는 regression을 막아야 한다.
- Folder별 linting rule을 다르게 두는 방법이 있다.
- `LEGACY_UI` 폴더는 warning만 띄우고, `SEMANTIC_UI` 폴더는 legacy UI 사용을 error로 막는다.
- View가 fully migrated되면 `SEMANTIC_UI` 폴더로 이동시킨다.
- Migration은 forward-only로 진행되어야 한다.

## 4.6 Migrating preexisting features
- New code가 new UI를 쓰도록 막았다면 기존 legacy UI를 migrate해야 한다.
- Preexisting features는 이미 ship되었고 동작 중인 screens와 flows다.
- 이런 화면은 product work를 멈추지 않고 조심스럽게 migrate해야 한다.
- 기존 화면에는 hard-coded styling, one-off layout hacks, inline values가 있을 수 있다.
- 가능하면 migration owner가 많은 grunt work를 직접 처리해 feature teams의 부담을 줄여야 한다.

### 4.6.1 Start by doing some work yourself
- 팀을 설득하는 가장 빠른 방법은 favor를 요구하지 않고 migration을 어느 정도 진행하는 것이다.
- UI system effort를 이끄는 사람이라면 직접 migrate해 보는 것이 좋다.
- 수십, 수백 개의 view가 있을 수 있으므로 tooling과 scripting이 중요하다.

### 4.6.2 Locally lint the legacy files
- Legacy UI를 빠르게 찾기 위해 linter를 local로 실행할 수 있다.
- 전체 codebase나 migrate하려는 feature를 대상으로 실행한다.
- Team 전체의 IDE에 warning을 띄우지 않고 문제 지점을 파악할 수 있다.
- Local linting 결과를 기반으로 batch migration 계획을 세울 수 있다.

### 4.6.3 Migrating to primitives via scripting
- Script를 활용하면 반복적인 replacement를 빠르게 처리할 수 있다.
- Entire UI migration을 완전히 automate할 수는 없지만, inline values를 primitives로 바꾸는 것은 비교적 안전하다.
- 예를 들어 `view.layer.cornerRadius = 8`을 `Radius.r8`로 바꿀 수 있다.
- `UIColor.darkGray`를 `Palette.greyDarkest`로 바꾸는 것도 비교적 안전하다.
- 이 단계는 semantic UI가 아니라 UI primitives로 migrate하는 것이다.
- 그래도 raw inline values를 제거하고 predefined values를 쓰게 만드는 큰 진전이다.

### 4.6.4 When to migrate to primitives vs semantic values
- Primitives로 migrate하는 것은 좋은 intermediate step이다.
- 빠르게 적용할 수 있고 predefined/pre-approved values를 사용하게 만든다.
- Colors를 `Palette`로 centralize하면 palette value 변경이 전체에 반영된다.
- 하지만 primitives migration은 최종 목표가 아니다.
- Primitive에는 semantic meaning이 없고 dark mode도 자동으로 지원되지 않을 수 있다.
- 이상적인 목표는 raw inline values → primitives → semantic values 순서로 이동하는 것이다.

### 4.6.5 Migrating to semantic UI
- Semantic UI로의 migration은 단순 script로 완전히 처리하기 어렵다.
- Script는 designer의 intent를 알 수 없다.
- 같은 `0xFFFFFF` color가 background인지, surface인지, highlight인지 알기 어렵다.
- `UIColor.white`를 `Palette.white`으로 바꾸는 것은 비교적 안전하다.
- 하지만 이를 `Color.primaryBackgroundColor`로 바꾸는 것은 context 판단이 필요하다.
- 잘못된 semantic replacement는 dark mode에서 bug를 만들 수 있다.
- Semantic migration은 abstraction level을 올리는 작업이므로 human review가 필요하다.

### 4.6.6 Script and undo workflow
- 완전 자동화는 어렵지만 semi-automated workflow는 매우 유용하다.
- 예를 들어 대부분의 white color가 background일 가능성이 높다면 script로 `Color.primaryBackgroundColor`로 바꿀 수 있다.
- 그 결과 80%는 맞고 20%는 틀릴 수 있다.
- 사람이 100%를 모두 고치는 대신 틀린 20%만 되돌리면 된다.
- Script를 실행한 뒤 commit 전에 git diff를 보고 잘못된 replacement를 undo한다.
- 이런 방식으로 80~90%를 scripting으로 처리하고, 10~20%만 manual로 처리한다.

### 4.6.7 Migrating via an intermediate step
- 어떤 primitive가 여러 의미로 사용되고 있지만 아직 최종 semantic name이 없을 수 있다.
- 이때 `MigrationAdapter` 같은 intermediate adapter를 만들 수 있다.
- 같은 light blue가 background, highlight, selected state로 쓰일 수 있다.
- Adapter를 사용하면 temporary semantic state를 만들 수 있다.
- 내부적으로는 둘 다 `Palette.lightBlue`를 반환해도, call site에서 의도가 분리된다.
- 나중에 meaning이 명확해지면 adapter reference를 proper semantic color로 바꿀 수 있다.

### 4.6.8 Preparing for manual labor
- Semi-automation 이후에는 manual work가 남는다.
- Multi-line configuration을 one-liner로 바꾸는 작업은 사람이 해야 할 수 있다.
- New headers 때문에 layout을 다시 잡거나 entire view를 new component로 교체해야 할 수도 있다.
- Migration owner는 자동화 가능한 부분을 먼저 줄이고, 남은 manual work를 팀과 함께 처리해야 한다.

## 4.7 Turning Migration into Adoption
- 모든 migration을 한 사람이 끝낼 수는 없다.
- Entire views나 components를 교체해야 하는 작업은 feature teams의 참여가 필요하다.
- 사람들이 new system을 adopt하게 만드는 것은 심리적 문제이기도 하다.
- 저항이 어디서 오는지 이해하고, migration을 실제 workflow 안에 녹여야 한다.
- 목표는 top-down 강제가 아니라 safe하고 obvious한 next step처럼 느끼게 만드는 것이다.

### 4.7.1 Start with migrating components
- Shared components를 migrate하는 것은 가장 leverage가 큰 작업 중 하나다.
- `PrimaryButton`, `CardView`, `FormField` 같은 component를 먼저 바꾼다.
- Component 내부가 semantic UI를 사용하면 call site는 그대로여도 benefit을 얻는다.
- Legacy screens도 shared components를 쓰고 있다면 자연스럽게 새 design system에 가까워진다.
- Component는 개발자에게 유지보수 부담을 줄이고 가치가 명확하기 때문에 adoption 설득이 쉽다.

### 4.7.2 Status quo bias
- 사람들은 익숙한 것을 선호한다.
- Legacy UI에 문제가 있어도 이미 알고 있고 편하기 때문에 계속 쓰고 싶어 한다.
- New system은 익숙한 workflow를 disrupt한다.
- Migration owner는 new system이 덜 intimidating하게 느껴지도록 해야 한다.
- Old code와 new code의 side-by-side comparison이 도움이 된다.

### 4.7.3 New rule: Touch a view? Migrate it.
- Product work를 멈추고 full-scale migration sprint를 잡는 것은 현실적이지 않을 수 있다.
- 대신 누군가 UI component나 view를 touch하면 그때 semantic UI로 migrate하게 한다.
- 변경된 file은 PR diff에 나타나고, linter가 warning을 띄울 수 있다.
- 이 방식은 migration을 별도 sprint가 아니라 기존 workflow에 통합한다.

### 4.7.4 Loss aversion
- 사람들은 이미 동작하는 것을 포기하고 싶어 하지 않는다.
- New system이 더 좋아도 기존 UI는 이미 tested되었고 launch된 상태다.
- 이때 migration의 장점만 말하는 것으로는 부족하다.
- Not migrating의 cost를 보여 줘야 한다.
- Dark mode, larger text sizes, design updates, onboarding이 더 어려워질 수 있다.

### 4.7.5 Make them take a small step
- 시작이 가장 어렵다.
- 첫 migration step은 아주 작게 만들어야 한다.
- 예를 들어 old `LoginButton`을 new `PrimaryButton`으로 바꾸는 정도다.
- Magic numbers를 `Spacing.medium`, `Spacing.small`로 바꾸는 것도 좋은 시작점이다.
- 작은 성공은 momentum을 만든다.

### 4.7.6 Social proof
- 사람들은 새 library를 처음 쓰는 것을 꺼릴 수 있다.
- 반대로 마지막으로 migrate하는 팀이 되는 것도 싫어한다.
- 이미 다른 팀이 성공적으로 사용하고 있으면 system이 더 safe하게 느껴진다.
- 다른 팀의 migration success story를 공유하면 adoption이 쉬워진다.

### 4.7.7 Using Self-Determination Theory to motivate migration
- Motivation은 autonomy, competence, relatedness와 관련된다.
- Autonomy는 사람들이 스스로 선택했다고 느끼게 하는 것이다.
- Competence는 사람들이 자신이 할 수 있다고 느끼게 하는 것이다.
- Relatedness는 다른 사람들과 함께하고 있다고 느끼게 하는 것이다.
- Teams가 점진적으로 opt in할 수 있게 하고, examples와 upgrade guide를 제공해야 한다.

### 4.7.8 The IKEA effect
- 사람들은 자신이 일부라도 만든 것에 더 애착을 가진다.
- Early adopters가 new UI system에 기여하게 하면 ownership이 생긴다.
- Component naming, spacing issue fix, new variant 제안에 참여할 수 있다.
- Small contribution만으로도 trust와 adoption이 생긴다.

### 4.7.9 Reward & recognize teams that migrate
- 먼저 migrate한 팀을 celebrate해야 한다.
- Slack, internal newsletter, team meeting에서 shout-out할 수 있다.
- 작은 thank-you, badge, leadership note도 효과가 있다.
- Recognition은 migration을 extra work가 아니라 progress처럼 느끼게 한다.

### 4.7.10 Helping people want the new system
- New system migration은 technical challenge이면서 psychological challenge다.
- 사람들은 lazy하거나 resistant한 것이 아니라 careful하고 busy하다.
- 기존 system은 flawed하더라도 delivery할 수 있는 safety net이었다.
- New system이 safe하고 obvious한 next step처럼 느껴지게 만드는 것이 중요하다.

## 4.8 App-wide migrations
- View-by-view migration은 많은 경우 효과적이다.
- 하지만 어떤 변경은 app-wide하게 적용되어야 한다.
- 예를 들어 header style을 크게 바꿨다면 일부 screen만 바뀌면 inconsistency가 생긴다.
- Visual consistency가 중요한 component는 app-wide migration 전략이 필요하다.
- App-wide migration에는 component adapters, API matching, feature flags 같은 방법이 있다.

### 4.8.1 Component adapters
- Component level adapter를 사용하면 app-wide migration을 쉽게 만들 수 있다.
- `OldPrimaryButton`을 `PrimaryButton`으로 migrate해야 한다면 call site를 모두 바꾸지 않아도 된다.
- `OldPrimaryButton`의 public API는 그대로 두고 내부 구현만 `PrimaryButton`으로 교체한다.
- 그러면 screen code를 수정하지 않고 수백 개 button을 한 번에 migrate할 수 있다.

### 4.8.2 Match the API
- New component가 old component와 비슷한 API를 제공하면 migration이 쉬워진다.
- API가 같으면 rename만으로 교체할 수 있다.
- Temporary compatibility API를 제공하면 migration을 돕는다.
- API matching은 call site churn을 줄인다.

### 4.8.3 Feature flags
- Fonts처럼 app 전체에 영향을 주는 변경은 feature flag가 적합할 수 있다.
- Font change는 text wrapping, view height, alignment에 영향을 준다.
- Runtime flag 뒤에 새 font system을 숨기는 방식이 안전하다.
- 충분한 views가 `companyFont(style:)`을 사용하게 된 뒤 flag를 켜면 전체 app header를 한 번에 바꿀 수 있다.
- Feature flag를 제거하기 전에는 old/new variant가 모두 안정적인지 확인해야 한다.

