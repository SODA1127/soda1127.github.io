---
layout: post
title: "Jetpack Compose Internals - 12장. Advanced Compose Runtime Use Cases (Compose 런타임 심층 확장 및 고급 활용)"
description: Compose UI를 넘어선 범용 트리 빌더로서의 Compose 런타임 내부 구조, VectorPainter의 독립 Composition 및 VNode 트리 메커니즘, Kotlin/JS 기반 DOM 제어와 독립형(Standalone) 브라우저 Composition 파이프라인(Recomposer, MonotonicFrameClock, GlobalSnapshotManager)을 심층 분석합니다.
image: 'https://imgur.com/o8Jk4Is.png'
category: 'programming'
date: 2026-09-10 09:00:00
tags:
- kotlin
- android
- jetpack compose
- compose internals
- compose runtime
- applier
- vectorpainter
- kotlin multiplatform
- kotlin js
- recomposer
introduction: Jetpack Compose는 흔히 안드로이드의 차세대 UI 툴킷으로 알려져 있지만, 그 내부의 Compose Compiler와 Compose Runtime은 특정 UI 프레임워크나 안드로이드 플랫폼에 전혀 종속되지 않는 범용 반응형 트리 관리 엔진입니다. UI 밖에서 Compose 런타임을 활용하는 아키텍처와 내부 동작 메커니즘을 상세히 분석합니다.
twitter_text: Compose UI 밖에서도 동작하는 Compose Runtime: VectorPainter의 커스텀 Composition, VNode 트리, Kotlin/JS 브라우저 DOM 엔진 및 독립형 Recomposer 파이프라인 심층 분석.
---

# 12장. Advanced Compose Runtime Use Cases (Compose 런타임 심층 확장 및 고급 활용)

지금까지 우리는 컴포저블 함수의 본질, Compose 컴파일러의 IR 트랜스폼 및 정적 분석, 런타임 슬롯 테이블(Slot Table)과 체인지 리스트(Changes), Compose UI의 `LayoutNode` 계층, 스냅샷 상태 관리 시스템(Snapshot System), 그리고 부수효과 핸들러(Effect Handlers)에 이르기까지 Compose의 핵심 아키텍처 전반을 심도 있게 탐구했습니다.

이 모든 논의는 대부분 안드로이드 UI 개발 맥락에서 이루어졌습니다. 대다수의 개발자가 안드로이드 앱 개발을 통해 Compose를 접하기 때문입니다.

그러나 Jetpack Compose의 설계 철학을 깊이 들여다보면 매우 놀라운 사실을 발견하게 됩니다. **Compose Compiler와 Compose Runtime은 안드로이드나 UI라는 개념 자체를 전혀 알지 못합니다.**

Compose 런타임은 임의의 객체로 구성된 **트리(Tree) 계층 구조를 선언적으로 생성하고, 상태 변화에 따라 점진적으로 업데이트(Recomposition & Apply Changes)하는 범용적인 반응형 노드 트리 관리 인프라**입니다.

이번 장에서는 Compose UI의 경계를 넘어, Compose 런타임이 제공하는 기본 메커니즘을 사용해 독자적인 커스텀 트리를 구축하고 다른 플랫폼과 환경으로 확장하는 고급 활용 사례와 그 내부 구현 원리를 소스 코드 레벨에서 살펴봅니다.

---

## Compose Runtime vs Compose UI: 근본적인 아키텍처 분리

Compose를 제대로 이해하려면 먼저 **Compose UI**와 **Compose Runtime**의 명확한 역할 분리를 인식해야 합니다.

![초기 Compose 프로토타입 (Kotlin 내 XML 삽입 구문)](/assets/img/compose-internals/chapter12/img-000.png)

위 이미지는 Compose 팀이 초기 프로토타입 단계에서 트위터(@AndroidDev)를 통해 공개했던 역사적인 스크린샷입니다. 당시에는 코틀린 코드 안에 XML 태그를 직접 작성하는 형태의 구문(`@html`, `<div>`, `<h1>`)을 테스트하기도 했습니다. 이는 과거 웹 생태계에서 React가 JS 안에 HTML을 인라인하는 JSX 방식을 도입했던 것과 매우 유사한 궤를 그립니다.

React가 초기에는 브라우저 DOM을 조작하기 위해 탄생했지만, 본질적인 조정(Reconciliation) 엔진이 분리되어 React Native(모바일), Ink(터미널 콘솔), React-three-fiber(3D 렌더러) 등으로 폭발적인 확장을 이룬 것처럼, Compose 역시 컴파일러/런타임과 구체적인 렌더러가 완전히 분리되어 있습니다.

| 계층 (Layer) | 주요 책임과 역할 | 플랫폼 의존성 |
| :--- | :--- | :--- |
| **Compose Compiler** | `@Composable` 함수 분석, `Composer` 및 스코프 주입, 메모이제이션, 비교 전파(Comparison Propagation), IR Lowering | 코틀린 컴파일러 플러그인 (모든 코틀린 플랫폼 지원) |
| **Compose Runtime** | 슬롯 테이블(`SlotTable`), 변경 목록(`Changes`), 상태 스냅샷 추적(`Snapshot`), 변경 사항 커밋(`Applier`) | 순수 코틀린 공통 런타임 (플랫폼 무관) |
| **Compose UI** | `LayoutNode` 트리 구축, 측정(Measure) 및 레이아웃(Layout) 정책, `Modifier` 체인 처리, 캔버스 드로잉(Draw), 터치 이벤트 및 접근성(Semantics) | 안드로이드 플랫폼 (뷰/윈도우 기반) |

Compose 런타임 소스 코드 어디에도 `View`, `Canvas`, `MotionLayout`, `MotionEvent` 같은 UI 관련 키워드는 등장하지 않습니다. 런타임의 주된 관심사는 **"주어진 노드 팩토리로 노드를 만들고, 변경된 속성을 반영하며, 자식 노드를 부모 노드에 올바른 순서로 삽입·이동·삭제하는 것"**뿐입니다.

### Kotlin Multiplatform으로의 확장

Compose 컴파일러가 코틀린의 모든 타깃 플랫폼(JVM, JS, Native, WASM)을 지원하게 되면서, 런타임을 이용해 트리 구조를 다루는 영역이 전 방위로 확장되었습니다.

![멀티플랫폼 환경에서의 Compose 모듈 구조 아키텍처](/assets/img/compose-internals/chapter12/img-001.png)

1. **Compose for Desktop (JVM)**:
   - Skia 그래픽 엔진 래퍼(Skiko)를 JVM으로 포팅하여 안드로이드 Compose UI의 렌더링 계층 전체(`LayoutNode`, 측정/배치, 드로잉)를 거의 100% 동일하게 재사용합니다.
   - 마우스 휠, 키보드 단축키, 창 크기 조절 등 데스크톱 전용 이벤트 시스템이 확장되었습니다.
2. **Compose for iOS (Kotlin/Native)**:
   - 데스크톱과 마찬가지로 Skiko를 통한 Skia 렌더링을 활용합니다.
   - JVM에서 코틀린/네이티브로 이식 가능한 공통 UI 로직을 최대한 공유하여 iOS 환경에서 픽셀 단위로 동일한 UI를 그립니다.
3. **Compose for Web (Kotlin/JS)**:
   - 데스크톱/iOS와 달리 안드로이드 Compose UI의 렌더링 계층을 사용하지 않고, **오직 Compiler와 Runtime만을 재사용**합니다.
   - 캔버스에 직접 그리는 대신, 브라우저가 기본 제공하는 HTML/CSS DOM 트리를 직접 조작하도록 설계되었습니다.
   - 최근에는 Kotlin WASM과 Skia 캔버스를 결합하여 모바일/데스크톱과 동일한 Compose UI를 웹 브라우저에서 구동하는 Canvas 기반 웹 렌더러도 공존하고 있습니다.

---

## (Re-) Introducing Composition: 트리와 런타임의 연결 고리

컴포저블 함수의 실행 컨텍스트이자 핵심 엔진은 바로 **`Composition`**입니다.

![Composition과 Recomposer, SlotTable, Applier 간의 구조적 관계](/assets/img/compose-internals/chapter12/img-003.png)

`Composition`은 다음과 같은 핵심 구성 요소들의 접점입니다:
- **`SlotTable`**: 컴포지션의 상태, 기억된 값(`remember`), 리컴포지션 스코프, 그룹 메타데이터를 저장하는 갭 버퍼 기반 캐시.
- **`Applier<N>`**: 런타임이 연산해 낸 변경 목록(Changes)을 실제 대상 트리 노드(`N`)에 반영하는 구체적 조작 인터페이스.
- **`Recomposer`**: 상태 스냅샷의 변경을 감지하고, 연관된 컴포지션의 리컴포지션을 스케줄링하여 변경 사항을 적용하는 오케스트레이터.

일반적인 안드로이드 앱 개발 시에는 `ComponentActivity.setContent` 또는 `ComposeView`가 백그라운드에서 알아서 `Composition`을 조립해주므로 개발자가 이를 직접 생성할 일이 없습니다. 하지만 Compose UI 외부에서 독자적인 트리를 구성하거나 특수한 서브 트리를 관리할 때는 개발자가 직접 컴포지션을 인스턴스화해야 합니다.

### Composition 생성 팩토리

런타임 라이브러리(`androidx.compose.runtime.Composition.kt`)는 다음과 같은 팩토리 함수를 제공합니다:

```kotlin
fun Composition(
    applier: Applier<*>,
    parent: CompositionContext
): Composition = ControlledComposition(
    applier = applier,
    parent = parent
)
```

이 함수는 두 개의 핵심 매개변수를 요구합니다:
1. **`parent: CompositionContext`**: 부모 컴포지션의 컨텍스트입니다. 컴포저블 함수 내부에서는 `rememberCompositionContext()`를 호출하여 손쉽게 현재 컴포지션의 컨텍스트를 획득할 수 있습니다. 독립적인 루트 환경에서는 `Recomposer` 자체가 `CompositionContext`를 구현하므로 `recomposer` 인스턴스를 부모로 넘깁니다. 이를 통해 `CompositionLocal` 전파와 스케줄러 동기화가 이루어집니다.
2. **`applier: Applier<*>`**: 컴포지션이 생성한 트리 변경 명령들을 실제 노드에 어떻게 반영할지 정의하는 객체입니다.

> **💡 Fun Fact: 노드가 없는 컴포지션 (`Applier<Nothing>`)**  
> 트리 노드를 아예 생성하지 않고 오직 컴포저블 함수의 리액티브 상태 구독, 파생 상태 연산, 비동기 이펙트(`LaunchedEffect`) 트리거링 능력만을 활용하고 싶다면 어떻게 해야 할까요?  
> 아무 동작도 하지 않는 빈 `Applier<Nothing>`을 넘기고 본문에서 `ComposeNode`를 호출하지 않으면 됩니다! Square의 Cash App 팀이 만든 **Molecule** 라이브러리가 바로 이 원리를 사용합니다. UI 트리를 생성하지 않고 컴포저블 함수의 반응형 로직만으로 `StateFlow`를 방출하는 아키텍처 패턴입니다.

---

## Compose UI 내부의 커스텀 Composition: VectorPainter 심층 분석

Compose UI 라이브러리 자체에서도 Compose 런타임의 트리 독립성을 활용하여 자체적인 커스텀 서브 트리를 운용하는 훌륭한 사례가 있습니다. 바로 벡터 그래픽을 렌더링하는 **`VectorPainter`**입니다.

기존 안드로이드에서 `VectorDrawable` XML을 파싱해 그리던 벡터 그래픽은 패스(Path), 그룹(Group), 클리핑, 회전 및 크기 조절 등의 계층 구조로 이루어져 있습니다. Compose UI는 이를 표현하기 위해 거대한 `LayoutNode`를 사용하지 않고, 훨씬 가볍고 벡터에 특화된 독자적인 `VNode` 트리를 구성합니다.

```kotlin
Image(
    painter = rememberVectorPainter { width, height ->
        Group(
            scaleX = 0.75f,
            scaleY = 0.75f
        ) {
            val pathData = PathData { /* SVG 패스 명령어 */ }
            Path(pathData = pathData)
        }
    },
    contentDescription = "Vector Image"
)
```

`rememberVectorPainter` 블록 내부에서 호출되는 `Group`과 `Path`는 평범한 컴포저블 함수처럼 보이지만, 일반적인 Compose UI 컴포저블(`Box`, `Text`, `Row`)과는 완전히 다른 성격의 컴포저블입니다.

![Compose UI와 VectorPainter의 서브컴포지션(독립 Composition) 구조](/assets/img/compose-internals/chapter12/img-006.png)

UI 컴포지션 내부에는 UI 전용 `UiApplier`가 연결되어 있고, 그 하위 트리는 `LayoutNode`로만 채워집니다. 반면 `VectorPainter`는 UI 컴포지션 내부에 자신만의 **독립된 서브 컴포지션(Vector Composition)**을 캡슐화하여 운용하며, 여기에는 `VectorApplier`가 바인딩됩니다.

따라서 `rememberVectorPainter` 내부에서는 오직 벡터용 요소(`Group`, `Path`)만 배치되어야 하며, 만약 여기에 `Image`나 `Box` 같은 일반 UI 노드를 호출하면 런타임에 클래스 캐스팅 예외가 발생하게 됩니다.

### 1. VNode 트리 계층 구조 설계

벡터 이미지는 레이아웃 측정이나 터치 이벤트 분배가 필요 없으므로 `LayoutNode`에 비해 구조가 매우 단순합니다:

```kotlin
sealed class VNode {
    abstract fun DrawScope.draw()
}

// 루트 노드: 전체 뷰포트 크기를 설정하고 렌더링을 캐싱
internal class VectorComponent : VNode() {
    val root = GroupComponent()

    override fun DrawScope.draw() {
        // 뷰포트 크기 설정, 렌더 타깃 캐싱 후 루트 그룹 드로잉
        root.draw()
    }
}

// 리프(Leaf) 노드: 실제 SVG PathData 렌더링
internal class PathComponent : VNode() {
    var pathData: List<PathNode> = emptyList()
    var fill: Brush? = null
    var stroke: Brush? = null
    // 추가 속성들...

    override fun DrawScope.draw() {
        // PathData 경로 렌더링
    }
}

// 컨테이너 노드: 자식 노드들을 보유하고 매트릭스 변환(회전, 스케일, 이동) 적용
internal class GroupComponent : VNode() {
    private val children = mutableListOf<VNode>()
    var scaleX: Float = 1f
    var scaleY: Float = 1f
    var translationX: Float = 0f
    var translationY: Float = 0f
    // 추가 속성들...

    fun insertAt(index: Int, instance: VNode) {
        children.add(index, instance)
    }

    fun remove(index: Int, count: Int) {
        children.subList(index, index + count).clear()
    }

    fun move(from: Int, to: Int, count: Int) {
        // 리스트 내 원소 이동
    }

    override fun DrawScope.draw() {
        // 현재 그룹의 변환 매트릭스를 적용한 뒤 모든 자식 노드를 순회하며 draw() 호출
        withTransform({
            scale(scaleX, scaleY)
            translate(translationX, translationY)
        }) {
            children.forEach { child ->
                with(child) { draw() }
            }
        }
    }
}
```

이 노드들은 `internal`로 감추어져 있으며, 외부 개발자는 오직 `@Composable` 함수 인터페이스를 통해서만 노드를 조작할 수 있습니다.

### 2. ComposeNode를 통한 트리 노드 방출 및 증분 업데이트

런타임에 노드를 생성하고 슬롯 테이블의 체인지 리스트에 방출하는 핵심 프리미티브는 **`ComposeNode`**입니다.

```kotlin
@Composable
fun Group(
    scaleX: Float = DefaultScaleX,
    scaleY: Float = DefaultScaleY,
    translationX: Float = DefaultTranslationX,
    translationY: Float = DefaultTranslationY,
    content: @Composable () -> Unit
) {
    ComposeNode<GroupComponent, VectorApplier>(
        factory = { GroupComponent() },
        update = {
            set(scaleX) { this.scaleX = it }
            set(scaleY) { this.scaleY = it }
            set(translationX) { this.translationX = it }
            set(translationY) { this.translationY = it }
        },
        content = content
    )
}

@Composable
fun Path(
    pathData: List<PathNode>,
    fill: Brush? = null,
    stroke: Brush? = null
) {
    ComposeNode<PathComponent, VectorApplier>(
        factory = { PathComponent() },
        update = {
            set(pathData) { this.pathData = it }
            set(fill) { this.fill = it }
            set(stroke) { this.stroke = it }
        }
    )
}
```

`ComposeNode`의 3가지 핵심 파라미터는 다음과 같이 동작합니다:
1. **`factory: () -> T`**: 컴포지션에 해당 노드가 처음 나타났을 때 인스턴스를 생성하는 팩토리 람다입니다.
2. **`update: Updater<T>.() -> Unit`**: 이미 생성된 노드의 속성을 증분 업데이트하는 블록입니다. `Updater.set(value)` 헬퍼는 전달된 값이 이전 값과 달라졌을 때만(`equals` 검사) 노드의 프로퍼티 변경 명령을 변경 목록(Changes)에 추가하므로 불필요한 재드로잉이나 무효화를 방지합니다.
3. **`content: @Composable () -> Unit`**: 자식 노드들을 정의하는 컴포저블 람다입니다. `Path`와 같은 리프 노드는 자식을 가질 수 없으므로 `content` 파라미터가 생략된 오버로드를 사용합니다.

### 3. VectorApplier 구현

`VNode` 트리를 조작하기 위해 `AbstractApplier`를 상속한 `VectorApplier`를 구현합니다:

```kotlin
class VectorApplier(root: VNode) : AbstractApplier<VNode>(root) {
    override fun insertTopDown(index: Int, instance: VNode) {
        current.asGroup().insertAt(index, instance)
    }

    override fun insertBottomUp(index: Int, instance: VNode) {
        // 벡터 트리는 Top-down으로 조립되므로 무시됩니다.
    }

    override fun remove(index: Int, count: Int) {
        current.asGroup().remove(index, count)
    }

    override fun move(from: Int, to: Int, count: Int) {
        current.asGroup().move(from, to, count)
    }

    override fun onClear() {
        root.asGroup().remove(0, root.asGroup().children.size)
    }

    private fun VNode.asGroup(): GroupComponent {
        return when (this) {
            is GroupComponent -> this
            else -> error("VNode는 GroupComponent에만 자식으로 삽입될 수 있습니다.")
        }
    }
}
```

#### Top-down vs Bottom-up 삽입 방식의 성능 차이
`Applier` 인터페이스는 `insertTopDown`과 `insertBottomUp` 두 가지 삽입 메서드를 제공합니다:
- **`insertTopDown`**: 부모 노드를 트리에 먼저 추가한 후, 그 아래에 자식 노드들을 하나씩 순차적으로 삽입합니다.
- **`insertBottomUp`**: 자식 노드들을 먼저 모두 조립하여 완전한 서브 트리를 완성한 뒤, 완성된 서브 트리를 부모에 한 번에 부착합니다.

이 구분의 이유는 **환경별 레이아웃/부착 비용의 차이** 때문입니다. 안드로이드의 고전적인 `ViewGroup`이나 브라우저의 DOM의 경우, 노드가 트리에 추가될 때마다 즉시 리레이아웃(Re-layout)이나 스타일 재계산이 트리거될 수 있어 `insertBottomUp`이 성능상 훨씬 유리합니다.  
반면 메모리 상의 순수 객체 리스트만을 조작하는 `VNode`나 `LayoutNode`의 경우 부착 시점의 레이아웃 부하가 없으므로 `insertTopDown`을 채택하여 단순하고 직관적으로 트리를 구축합니다.

### 4. VectorPainter와 UI 컴포지션의 수명 주기 결합

이제 `VectorApplier`와 `VNode`를 UI 컴포지션 생명주기에 결합하는 `VectorPainter`의 내부 구현을 완성합니다:

```kotlin
class VectorPainter internal constructor() : Painter() {
    private val vector = VectorComponent()
    private var composition: Composition? = null

    @Composable
    internal fun RenderVector(
        content: @Composable (viewportWidth: Float, viewportHeight: Float) -> Unit
    ) {
        // 1. UI 컴포지션으로부터 부모 Context를 상속받아 독립 Composition 생성
        val composition = composeVector(
            rememberCompositionContext(),
            content
        )

        // 2. UI 컴포지션을 벗어나면 벡터 컴포지션을 안전하게 dispose
        DisposableEffect(composition) {
            onDispose {
                composition.dispose()
            }
        }
    }

    private fun composeVector(
        parent: CompositionContext,
        composable: @Composable (Float, Float) -> Unit
    ): Composition {
        val existingComposition = composition
        val nextComposition = if (existingComposition == null || existingComposition.isDisposed) {
            Composition(
                VectorApplier(vector.root),
                parent
            )
        } else {
            existingComposition
        }
        this.composition = nextComposition

        // 3. 컴포지션에 벡터 컨텐트 설정
        nextComposition.setContent {
            composable(vector.viewportWidth, vector.viewportHeight)
        }

        return nextComposition
    }

    // Painter 인터페이스 구현: 화면에 렌더링될 때 호출됨
    override fun DrawScope.onDraw() {
        with(vector) {
            draw()
        }
    }
}
```

이 구현의 핵심적인 세부 동작을 분석해 보면:
1. **컨텍스트 상속 (`rememberCompositionContext()`)**: 부모 UI 컴포지션의 컨텍스트를 넘겨줌으로써 UI의 `LocalDensity`나 테마 등의 `CompositionLocal` 값이 벡터 내부 컴포저블로 자연스럽게 상속되며, 동일한 Recomposer 스케줄러를 공유하게 됩니다.
2. **생명주기 정리 (`DisposableEffect`)**: `VectorPainter`가 화면에서 제거되거나 컴포지션 범위를 벗어날 때 `composition.dispose()`를 호출하여 슬롯 테이블 메모리와 리컴포지션 옵저버를 완전히 해제합니다.
3. **독립된 Applier 유지**: `VectorApplier`를 사용하므로 UI의 `LayoutNode`와 충돌하지 않고 완전히 독립된 순수 벡터 그래픽 트리를 효율적으로 유지보수합니다.

---

## Kotlin/JS 환경에서 브라우저 DOM 관리하기: 독립형(Standalone) Composition 구축

Compose의 확장성을 가장 극명하게 보여주는 사례는 **브라우저 환경(Kotlin/JS)에서 HTML DOM 트리를 Compose 런타임으로 직접 제어하는 것**입니다.

브라우저는 이미 HTML/CSS 기반의 훌륭한 뷰 트리를 내장하고 있으며, 자바스크립트 및 Kotlin/JS 표준 라이브러리는 이를 조작할 수 있는 W3C DOM API(`org.w3c.dom.*`)를 제공합니다.

### 브라우저 DOM과 Kotlin/JS 객체 모델

다음과 같은 간단한 HTML 목록 구조를 살펴보겠습니다:

```html
<div>
  <ul>
    <li>Item 1</li>
    <li>Item 2</li>
    <li>Item 3</li>
  </ul>
</div>
```

이 문서는 브라우저와 자바스크립트 엔진 내부에서 각각 다음과 같은 트리 구조로 해석됩니다:

![브라우저 관점의 HTML DOM 트리 계층 구조](/assets/img/compose-internals/chapter12/img-008.png)

![JavaScript 및 Kotlin/JS 관점의 DOM 객체 트리 구조](/assets/img/compose-internals/chapter12/img-010.png)

- **`HTMLElement`**: `<div>`, `<ul>`, `<li>` 같은 태그 요소를 나타내며, `document.createElement(tagName)`으로 생성됩니다.
- **`Text`**: 태그 사이의 문자열 컨텐츠를 나타내는 텍스트 노드로, `document.createTextNode(text)`로 생성됩니다.

### 1. HTML 태그 및 텍스트 컴포저블 작성

DOM 노드를 방출하는 컴포저블을 `ComposeNode`와 `ReusableComposeNode`를 사용해 설계합니다:

```kotlin
@Composable
fun Tag(
    tag: String,
    content: @Composable () -> Unit
) {
    ComposeNode<HTMLElement, DomApplier>(
        factory = { document.createElement(tag) as HTMLElement },
        update = {},
        content = content
    )
}

@Composable
fun Text(value: String) {
    ReusableComposeNode<Text, DomApplier>(
        factory = { document.createTextNode("") },
        update = {
            set(value) { this.data = it }
        }
    )
}
```

여기서 두 컴포저블의 구현 차이에 주목해야 합니다:
- **`Tag`**: 태그 이름(예: `<div>` vs `<span>`)은 브라우저 내부에서 근본적으로 다른 DOM 타입 객체이므로 인플레이스(in-place) 교체가 불가능합니다. 만약 태그 이름이 동적으로 변경된다면 이전 노드를 삭제하고 새 노드를 만들어야 합니다. 따라서 일반적인 웹 컴포즈 구현에서는 `Tag("div")` 대신 `Div { ... }`, `Span { ... }`, `Button { ... }`처럼 전용 컴포저블로 감싸서 컴파일 타임 그룹 키를 다르게 부여함으로써 노드가 안전하게 재생성되도록 유도합니다.
- **`Text`**: 텍스트 노드는 내용이 변경되더라도 DOM 구조 자체가 바뀌지 않고 단순히 `data` 프로퍼티만 업데이트하면 됩니다. 따라서 **`ReusableComposeNode`**를 사용하여 슬롯 테이블에서 노드 인스턴스를 적극적으로 재사용합니다.

### 2. DomApplier 구현

W3C `Node` 인터페이스를 조작하는 `DomApplier`를 구현합니다:

```kotlin
class DomApplier(root: HTMLElement) : AbstractApplier<Node>(root) {
    override fun insertTopDown(index: Int, instance: Node) {
        // 브라우저 DOM은 자식 삽입 시 리플로우를 유발하므로 BottomUp을 주로 사용하거나
        // insertBefore API를 통해 특정 인덱스에 노드를 삽입합니다.
    }

    override fun insertBottomUp(index: Int, instance: Node) {
        val currentElement = current as HTMLElement
        val children = currentElement.childNodes
        if (index >= children.length) {
            currentElement.appendChild(instance)
        } else {
            currentElement.insertBefore(instance, children.item(index))
        }
    }

    override fun remove(index: Int, count: Int) {
        val currentElement = current as HTMLElement
        repeat(count) {
            currentElement.childNodes.item(index)?.let { child ->
                currentElement.removeChild(child)
            }
        }
    }

    override fun move(from: Int, to: Int, count: Int) {
        // DOM API를 사용해 특정 위치의 자식 노드들을 이동
    }

    override fun onClear() {
        while (root.hasChildNodes()) {
            root.removeChild(root.firstChild!!)
        }
    }
}
```

### 3. 독립형 런타임 구동 파이프라인: renderComposable

안드로이드의 `ComposeView` 도움 없이, 브라우저에서 최초의 루트 컴포지션을 띄우고 리컴포지션 루프를 가동하는 `renderComposable` 함수를 구축합니다:

```kotlin
fun renderComposable(root: HTMLElement, content: @Composable () -> Unit) {
    // 1. 전역 스냅샷 매니저 가동 (상태 변경 감지 알림 활성화)
    GlobalSnapshotManager.ensureStarted()

    // 2. 프레임 클록 및 메인 디스패처로 Recomposer 코루틴 컨텍스트 구성
    val recomposerContext = DefaultMonotonicFrameClock + Dispatchers.Main
    val recomposer = Recomposer(recomposerContext)

    // 3. 최상위 ControlledComposition 생성
    val composition = ControlledComposition(
        applier = DomApplier(root),
        parent = recomposer
    )

    // 4. 루트 컴포저블 컨텐트 바인딩
    composition.setContent(content)

    // 5. 비동기 리컴포지션 및 변경 사항 적용 루프 시작
    CoroutineScope(recomposerContext).launch(start = CoroutineStart.UNDISPATCHED) {
        recomposer.runRecomposeAndApplyChanges()
    }
}
```

이 부트스트랩 파이프라인의 5단계는 모던 Compose 아키텍처의 심장부입니다:

1. **`GlobalSnapshotManager.ensureStarted()`**:
   - Compose의 상태(`mutableStateOf`)가 변경되면 백그라운드에서 `Snapshot.sendApplyNotifications()`가 호출되어야 대기 중인 리컴포지션 스코프들이 무효화(Invalidation)됩니다.
   - 플랫폼 런타임은 이 전역 스냅샷 리스너를 실행해 주어야 하며, JS 환경에서는 채널 기반의 스냅샷 큐를 백그라운드에 구동합니다.
2. **`DefaultMonotonicFrameClock + Dispatchers.Main`**:
   - `Recomposer`가 언제 리컴포지션을 실행할지 결정하는 시계(`MonotonicFrameClock`)가 필요합니다.
   - 브라우저 환경에서는 브라우저 렌더링 프레임 동기화 함수인 `window.requestAnimationFrame`을 래핑한 프레임 클록을 사용합니다.
   - 자바스크립트는 싱글 스레드 이벤트 루프 환경이므로 `Dispatchers.Main`이 기본 디스패처가 됩니다.
3. **`Recomposer.runRecomposeAndApplyChanges()`**:
   - 리컴포지션 요청을 대기하고 있다가, 프레임 틱(Frame Tick)이 발생하면 무효화된 스코프들을 다시 계산하고 변경 사항을 `DomApplier`를 통해 브라우저 화면에 즉시 플러시(Flush)합니다.
   - 안드로이드에서는 Activity 생명주기에 맞춰 `recomposer.cancel()`을 호출하지만, 웹 브라우저에서는 페이지가 닫힐 때까지 프로세스가 영구 지속됩니다.

### 4. 인터랙션 및 상태 연동

DOM 이벤트 리스너를 지원하도록 `Tag` 컴포저블에 클릭 핸들러를 추가하고, 반응형 카운터 애플리케이션을 작성해 봅니다:

```kotlin
@Composable
fun Tag(
    tag: String,
    onClick: () -> Unit = {},
    content: @Composable () -> Unit
) {
    ComposeNode<HTMLElement, DomApplier>(
        factory = { document.createElement(tag) as HTMLElement },
        update = {
            // 이벤트 리스너 람다가 변경되면 DOM 노드의 onclick 속성을 재할당
            set(onClick) {
                this.onclick = { _ -> onClick() }
            }
        },
        content = content
    )
}
```

이제 브라우저의 `main()` 진입점에서 다음과 같이 카운터 컴포저블을 렌더링할 수 있습니다:

```kotlin
fun main() {
    renderComposable(document.body!!) {
        var counterState by remember { mutableStateOf(0) }

        Tag("h1") {
            Text("현재 카운트: $counterState")
        }

        Tag("button", onClick = { counterState++ }) {
            Text("숫자 올리기!")
        }
    }
}
```

버튼을 클릭하면 `counterState`의 쓰기 옵저버가 발화되고, 전역 스냅샷 매니저가 변경 사항을 감지하여 `Recomposer`를 깨웁니다. `Recomposer`는 다음 애니메이션 프레임에 `Tag("h1")` 내부의 `Text` 컴포저블만을 정밀하게 리컴포지션하여, 브라우저 DOM의 텍스트 노드 값을 `0`에서 `1`로 최소한의 비용으로 업데이트합니다!

이 모든 과정이 안드로이드 라이브러리 참조 없이, 순수 코틀린 멀티플랫폼 Compose 런타임 위에서 완벽하게 동작합니다.

---

## Compose 생태계의 다양한 확장 사례

이러한 독립형 Compose 런타임의 강력함 덕분에, 모바일 앱 외에도 전 세계 개발자들에 의해 놀라운 오픈소스 프로젝트들이 탄생했습니다:

```
┌─────────────────────────────────────────────────────────┐
│                    Compose Compiler                     │
│         (@Composable 함수 변환, IR Lowering)             │
└───────────────────────────┬─────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────┐
│                    Compose Runtime                      │
│        (SlotTable, Snapshot State, Recomposer)          │
└───────────────┬───────────────────────────┬─────────────┘
                │                           │
    ┌───────────▼───────────┐   ┌───────────▼───────────┐
    │     Compose UI        │   │    Custom Runtimes    │
    ├───────────────────────┤   ├───────────────────────┤
    │ • Android UI          │   │ • VectorPainter (VNode)
    │ • Compose Desktop     │   │ • Compose for Web (DOM)
    │ • Compose iOS (Skia)  │   │ • Mosaic (Terminal)   │
    │ • Compose Wasm (Skia) │   │ • Molecule (Flow)     │
    └───────────────────────┘   └───────────────────────┘
```

1. **Mosaic (Jake Wharton)**:
   - 터미널 콘솔 화면에서 ANSI 이스케이프 시퀀스와 텍스트 노드 트리를 다루는 CLI 전용 Compose 런타임입니다.
   - 터미널에서 프로그레스 바, 실시간 다운로드 상태, 대화형 CLI 도구를 선언형 컴포저블 코드로 우아하게 구현합니다.
2. **Molecule (Cash App)**:
   - UI 트리를 아예 만들지 않고, 컴포저블 함수의 단방향 데이터 흐름과 상태 구독 기능을 활용해 비즈니스 로직 Presenter를 작성하고 `StateFlow`를 생성합니다.
3. **Jetpack Glance**:
   - 안드로이드 홈 화면 위젯(`AppWidget`)을 제작하기 위해 `RemoteViews` 계층을 Compose 런타임 노드로 구축하여 위젯 개발 생산성을 혁신했습니다.

---

## 핵심 요약

| 구성 요소 | 핵심 역할 | 주요 구현 및 확장 포인트 |
| :--- | :--- | :--- |
| **`Composition`** | 트리 조작과 상태 메모이제이션을 연결하는 핵심 단위 | `Composition(applier, parent)` 팩토리를 통해 독자적 컴포지션 인스턴스화 |
| **`Applier<N>`** | 컴포지션의 노드 변경 명령을 실제 대상 트리에 반영하는 브릿지 | `insertTopDown` / `insertBottomUp`, `remove`, `move`, `onClear` 구현 |
| **`ComposeNode`** | 컴포저블 함수 내에서 실제 타깃 노드를 슬롯 테이블에 방출 | `factory`(생성), `update`(속성 증분 갱신), `content`(자식 트리 바인딩) |
| **`VectorPainter`** | UI 컴포지션 내에 캡슐화된 독립 서브 컴포지션 활용 사례 | `VNode` 트리 + `VectorApplier`, 부모 컨텍스트 상속 및 `DisposableEffect` 정리 |
| **`renderComposable`** | 독립 플랫폼(Web/CLI)에서 Compose를 부트스트랩하는 진입점 | `GlobalSnapshotManager` 가동 + `Recomposer` 런칭 + 프레임 클록 연결 |

Jetpack Compose는 단순한 안드로이드의 UI 툴킷이 아닙니다.  
그것은 **"코틀린이라는 언어 위에 구축된, 모든 종류의 트리와 계층 구조를 상태 기반으로 가장 우아하게 다룰 수 있는 차세대 반응형 프로그래밍 아키텍처"**입니다.

이로써 *Jetpack Compose Internals* 전 챕터에 걸친 대여정이 마무리되었습니다. 컴파일러가 만들어내는 바이트코드의 마법부터, 런타임의 치밀한 슬롯 테이블 메모리 관리, 프레임워크 전반을 관통하는 상태 스냅샷 시스템과 멀티플랫폼 확장성에 이르기까지, 내부 동작 원리에 대한 깊은 통찰이 여러분의 모던 앱 엔지니어링 여정에 강력한 날개가 되기를 바랍니다.

---

## 이전 글

- [11장. Effects and Effect Handlers (이펙트와 이펙트 핸들러 심층 분석)](/jetpack-compose-internals-11-effects-and-effect-handlers/)
- [10장. State Snapshot System (스냅샷 상태 시스템 심층 분석)](/jetpack-compose-internals-10-state-snapshot-system/)
- [9장. Modifier 체인, 노드 드로잉 및 시맨틱스(Semantics)](/jetpack-compose-internals-09-modifiers-drawing-and-semantics/)
- [8장. LookaheadLayout과 애니메이션 레이아웃](/jetpack-compose-internals-08-lookahead-layout/)
- [7장. Advanced Compose Runtime Use Cases (개요)](/jetpack-compose-internals-07-advanced-compose-runtime-use-cases/)
- [7장. Layouts and Measurement](/jetpack-compose-internals-07-layouts-and-measurement/)
- [6장. Effects and Effect Handlers (기초 요약)](/jetpack-compose-internals-06-effects-and-effect-handlers/)
- [5장. State Snapshot System (기초 요약)](/jetpack-compose-internals-05-state-snapshot-system/)
- [4장. Compose UI](/jetpack-compose-internals-04-compose-ui/)
- [3장. Compose Runtime (3)](/jetpack-compose-internals-03-compose-runtime-03/)
- [2장. Compose Compiler](/jetpack-compose-internals-02-compose-compiler/)
- [1장. Composable 함수](/jetpack-compose-internals-01-composable-functions/)
