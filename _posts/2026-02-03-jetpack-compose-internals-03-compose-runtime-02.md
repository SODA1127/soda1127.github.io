---
layout: post
title: "Jetpack Compose Internals - 3장. Compose 런타임 (2) - Composer"
description: Composable 함수와 런타임을 연결하는 Composer의 역할과 동작 원리를 파헤쳐 봅니다.
image: 'https://imgur.com/o8Jk4Is.png'
category: 'programming'
date: 2026-02-03 23:00:00
tags:
- kotlin
- android
- jetpack compose
- compose internals
- runtime
- composer
introduction: Composer가 어떻게 Composable 함수의 호출을 런타임과 연결하고 변경 사항을 기록하는지 알아봅니다.
twitter_text: Composable 함수와 런타임을 연결하는 Composer의 역할과 동작 원리를 파헤쳐 봅니다.
---

# 3장. Compose 런타임 (2) - Composer

지난 글에서 **슬롯 테이블**과 **변경 리스트**의 차이를 배웠습니다. 이번 글에서는 Composable 함수와 런타임을 실제로 연결하는 핵심 컴포넌트인 **Composer**를 깊이 있게 살펴보겠습니다.

---

## Composer의 역할

2장에서 컴파일러가 모든 Composable 함수에 `$composer: Composer` 파라미터를 주입한다는 것을 배웠습니다. 이 주입된 `$composer` 인스턴스가 바로 **우리가 작성한 Composable 함수와 Compose 런타임을 연결하는 다리** 역할을 합니다.

`currentComposer`를 통해 Composition을 호출하는 것은 모두 Jetpack Compose 런타임의 일부입니다.

---

## Composer에 데이터 공급하기

노드가 어떻게 메모리 내 트리 표현에 추가되는지 알아보겠습니다. 모든 UI 컴포넌트의 기반이 되는 `Layout` Composable을 예시로 들어보겠습니다.

```kotlin
@Composable inline fun Layout(
    content: @Composable () -> Unit,
    modifier: Modifier = Modifier,
    measurePolicy: MeasurePolicy
) {
    val density = LocalDensity.current
    val layoutDirection = LocalLayoutDirection.current
    
    ReusableComposeNode<ComposeUiNode, Applier<Any>>(
        factory = ComposeUiNode.Constructor,
        update = {
            set(measurePolicy, ComposeUiNode.SetMeasurePolicy)
            set(density, ComposeUiNode.SetDensity)
            set(layoutDirection, ComposeUiNode.SetLayoutDirection)
        },
        skippableUpdate = materializerOf(modifier),
        content = content
    )
}
```

`Layout`은 `ReusableComposeNode`를 사용하여 `LayoutNode`를 Composition에 방출(emit)합니다. 하지만 여기서 중요한 점은, 이것이 **즉시 노드를 생성하고 추가하는 것이 아니라**, **적절한 시점에 노드를 생성, 초기화, 삽입하는 방법을 런타임에게 가르쳐주는 것**입니다.

```kotlin
@Composable
inline fun <T, reified E : Applier<*>> ReusableComposeNode(
    noinline factory: () -> T,
    update: @DisallowComposableCalls Updater<T>.() -> Unit,
    noinline skippableUpdate: @Composable SkippableUpdater<T>.() -> Unit,
    content: @Composable () -> Unit
) {
    currentComposer.startReusableNode()
    currentComposer.createNode(factory)
    
    Updater<T>(currentComposer).update() // 초기화
    
    currentComposer.startReplaceableGroup(0x7ab4aae9)
    content()
    currentComposer.endReplaceableGroup()
    currentComposer.endNode()
}
```

모든 작업이 `currentComposer` 인스턴스에 위임됩니다. 또한 `content` 람다를 감싸기 위해 replaceable 그룹을 시작하는 것을 볼 수 있습니다. `content` 람다 내에서 방출된 모든 자식은 Composition에서 이 그룹(그리고 Composable)의 자식으로 효과적으로 저장됩니다.

동일한 방출 작업은 다른 모든 Composable 함수에서도 이루어집니다. `remember`를 예로 들어보겠습니다:

```kotlin
@Composable
inline fun <T> remember(calculation: @DisallowComposableCalls () -> T): T =
    currentComposer.cache(invalid = false, calculation)
```

`remember`는 `currentComposer`를 사용하여 제공된 람다가 반환한 값을 Composition에 캐시(기억)합니다.

```kotlin
@ComposeCompilerApi
inline fun <T> Composer.cache(invalid: Boolean, block: () -> T): T {
    return rememberedValue().let {
        if (invalid || it === Composer.Empty) {
            val value = block()
            updateRememberedValue(value)
            value
        } else it
    } as T
}
```

먼저 Composition(슬롯 테이블)에서 값을 검색합니다. 찾지 못하면 값을 업데이트하도록 변경 사항을 스케줄링(기록)합니다. 그렇지 않으면 값을 그대로 반환합니다.

---

## 변경 사항 모델링

이전 섹션에서 설명했듯이, `currentComposer`에 위임된 모든 방출 작업은 내부적으로 **Change**로 모델링되어 리스트에 추가됩니다. `Change`는 현재 Applier와 SlotWriter에 접근할 수 있는 지연된 함수입니다:

```kotlin
internal typealias Change = (
    applier: Applier<*>,
    slots: SlotWriter,
    rememberManager: RememberManager
) -> Unit
```

이러한 변경 사항들은 리스트에 추가(기록)됩니다. "방출(emitting)"이란 본질적으로 이러한 `Change`들을 생성하는 것을 의미하며, 이들은 슬롯 테이블에서 노드를 추가, 제거, 교체 또는 이동하고 결과적으로 Applier에 알리기 위한 지연된 람다입니다.

이러한 이유로, "변경 사항 방출(emitting changes)"에 대해 이야기할 때 "변경 사항 기록(recording changes)" 또는 "변경 사항 스케줄링(scheduling changes)"이라는 단어를 사용할 수도 있습니다. 모두 같은 것을 가리킵니다.

Composition 후, 모든 Composable 함수 호출이 완료되고 모든 변경 사항이 기록되면, 이들은 **Applier에 의해 일괄적으로 적용**됩니다.

---

## 쓰기 시점 최적화

위에서 배웠듯이, 새 노드 삽입은 Composer에 위임됩니다. 즉, Composer는 이미 Composition에 새 노드를 삽입하는 프로세스에 있을 때를 항상 알고 있습니다.

그런 경우, Composer는 프로세스를 단축하고 변경 사항을 기록하는 대신(나중에 해석될 리스트에 추가) **변경 사항이 방출될 때 즉시 슬롯 테이블에 쓰기를 시작**할 수 있습니다. 다른 경우에는 아직 변경할 시점이 아니므로 변경 사항이 기록되고 지연됩니다.

---

## 그룹 쓰기와 읽기

Composition이 완료되면 `composition.applyChanges()`가 최종적으로 호출되어 트리를 구체화하고 변경 사항이 슬롯 테이블에 기록됩니다. Composer는 다양한 유형의 정보를 쓸 수 있습니다: 데이터, 노드 또는 그룹. 하지만 궁극적으로는 모두 단순화를 위해 **그룹 형태로 저장**됩니다. 단지 구별을 위해 다른 그룹 필드를 가질 뿐입니다.

Composer는 모든 그룹을 "시작(start)"하고 "종료(end)"할 수 있습니다. 이는 수행 중인 작업에 따라 다른 의미를 갖습니다:
- **쓰기 중**: 슬롯 테이블에서 "그룹 생성됨"과 "그룹 제거됨"을 의미
- **읽기 중**: SlotReader가 그룹에서 읽기를 시작하거나 종료하도록 읽기 포인터를 이동

### 그룹 시작 시 발생하는 일들

Composer가 그룹을 시작하려고 할 때 다음과 같은 일들이 발생합니다:

1. **삽입 중인 경우**: 이미 쓰기 작업을 진행 중이므로 기다릴 이유 없이 슬롯 테이블에 바로 씁니다.

2. **대기 중인 작업이 있는 경우**: 변경 사항을 적용할 때 적용될 수 있도록 기록합니다. 여기서 Composer는 그룹이 이미 존재하는 경우(테이블에) 재사용을 시도합니다.

3. **그룹이 다른 위치에 있는 경우**(이동됨): 그룹의 모든 슬롯을 이동하는 작업을 기록합니다.

4. **그룹이 새로운 경우**(테이블에서 찾을 수 없음): 삽입 모드로 전환하여 그룹이 완료될 때까지 그룹과 모든 자식을 중간 `insertTable`(다른 SlotTable)에 씁니다. 이는 최종 테이블에 삽입될 그룹을 스케줄링합니다.

5. **삽입 중이지 않고 대기 중인 쓰기 작업이 없는 경우**: 그룹을 읽기 시작하려고 시도합니다.

그룹 재사용은 흔합니다. 때로는 새 노드를 만들 필요 없이 이미 존재하는 경우 재사용할 수 있습니다(`ReusableComposeNode` 참조). 이는 Applier가 노드로 이동하는 작업을 방출(기록)하지만 생성 및 초기화 작업은 건너뜁니다.

노드의 프로퍼티를 업데이트해야 하는 경우, 그 작업도 `Change`로 기록됩니다.

---

## 값 기억하기 (Remembering Values)

Composer는 값을 Composition에 기억(슬롯 테이블에 쓰기)하고 나중에 업데이트할 수 있는 능력이 있습니다. 마지막 Composition에서 변경되었는지 확인하는 비교는 `remember`가 호출될 때 바로 수행되지만, **업데이트 작업은 Composer가 이미 삽입 중이 아닌 한 `Change`로 기록됩니다**.

업데이트할 값이 `RememberObserver`인 경우, Composer는 Composition에서 기억 작업을 추적하기 위한 암시적 `Change`도 기록합니다. 이는 나중에 기억된 모든 값을 잊어야 할 때 필요합니다.

---

## Recompose Scopes

Composer를 통해 일어나는 또 다른 중요한 작업은 **recompose scopes**입니다. 이는 스마트 리컴포지션을 가능하게 합니다. Recompose scopes는 restart 그룹과 직접 연결되어 있습니다.

Restart 그룹이 생성될 때마다 Composer는 그것을 위한 `RecomposeScope`를 만들고, 이를 Composition의 `currentRecomposeScope`로 설정합니다.

```kotlin
// 컴파일러가 보일러플레이트를 삽입한 후
@Composable
fun A(x: Int, $composer: Composer<*>, $changed: Int) {
    $composer.startRestartGroup()
    // ...
    f(x)
    $composer.endRestartGroup()?.updateScope { next ->
        A(x, next, $changed or 0b1)
    }
}
```

`RecomposeScope`는 **나머지 Composition과 독립적으로 리컴포지션될 수 있는 Composition의 영역**을 모델링합니다. Composable을 수동으로 무효화하고 리컴포지션을 트리거하는 데 사용할 수 있습니다:

```kotlin
composer.currentRecomposeScope().invalidate()
```

리컴포지션을 위해 Composer는 슬롯 테이블을 이 그룹의 시작 위치로 이동한 다음, 람다에 전달된 recompose 블록을 호출합니다. 이는 Composable 함수를 다시 효과적으로 호출하여 한 번 더 방출하게 하고, 따라서 Composer에게 테이블의 기존 데이터를 재정의하도록 요청합니다.

Composer는 무효화된(리컴포지션 대기 중인) 모든 recompose scopes의 **Stack**을 유지합니다. `currentRecomposeScope`는 실제로 이 Stack을 peek하여 얻습니다.

하지만 `RecomposeScope`는 항상 활성화되는 것은 아닙니다. Compose가 Composable 내에서 State 스냅샷의 읽기 작업을 발견할 때만 발생합니다. 이 경우 Composer는 `RecomposeScope`를 **used**로 표시하여, Composable 끝에 삽입된 "end" 호출이 더 이상 null을 반환하지 않게 하고, 따라서 뒤따르는 리컴포지션 람다를 활성화합니다(`?` 문자 다음 참조).

Composer는 리컴포지션이 필요할 때 현재 부모 그룹의 모든 무효화된 자식 그룹을 리컴포지션하거나, 필요하지 않을 때는 단순히 리더가 그룹을 끝까지 건너뛰도록 만들 수 있습니다(2장의 비교 전파 섹션 참조).

---

## Composer의 Side Effects

Composer는 **SideEffect**도 기록할 수 있습니다. `SideEffect`는 항상 **Composition 후에 실행**됩니다. 변경 사항이 Composition에 적용될 때 호출할 함수로 기록됩니다.

---

## CompositionLocals 저장

Composer는 `CompositionLocal`을 등록하고 키가 주어지면 그 값을 얻는 수단도 제공합니다. `CompositionLocal.current` 호출은 이에 의존합니다. Provider와 그 값들도 슬롯 테이블에 그룹으로 함께 저장됩니다.

---

## 소스 정보 저장

Composer는 Composition 중에 수집된 `CompositionData` 형태의 소스 정보도 저장하여 Compose 도구가 활용할 수 있도록 합니다.

---

## CompositionContext를 통한 Composition 연결

단일 Composition이 아니라 **Composition과 Subcomposition의 트리**가 존재합니다. Subcomposition은 별도의 무효화를 지원하기 위해 현재 Composition의 컨텍스트에서 별도의 Composition을 구성할 목적으로 인라인으로 생성된 Composition입니다.

![Composition과 Subcomposition 구조](/assets/img/compose-internals/chapter3/composition-subcomposition.png)
*Composition과 Subcomposition의 트리 구조. LayoutNode와 VNode가 각각의 Composition에서 관리됩니다.*

Subcomposition은 부모 `CompositionContext` 참조를 통해 부모 Composition에 연결됩니다. 이 컨텍스트는 Composition과 Subcomposition을 트리로 함께 연결하기 위해 존재합니다. 이를 통해 `CompositionLocal`과 무효화가 단일 Composition에 속한 것처럼 트리 아래로 투명하게 해결/전파됩니다.

![Fragment와 Composition의 관계](/assets/img/compose-internals/chapter3/fragment-composition.png)
*여러 Fragment에서 각각 독립적인 Composition을 가질 수 있습니다.*

Subcomposition 생성은 보통 `rememberCompositionContext`를 통해 수행됩니다:

```kotlin
@Composable fun rememberCompositionContext(): CompositionContext {
    return currentComposer.buildContext()
}
```

이 함수는 슬롯 테이블의 현재 위치에 새 Composition을 기억하거나, 이미 기억되어 있는 경우 반환합니다. `VectorPainter`, `Dialog`, `SubcomposeLayout`, `Popup`, Android View를 Composable 트리에 통합하는 래퍼인 `AndroidView` 같은 별도의 Composition이 필요한 곳에서 Subcomposition을 만드는 데 사용됩니다.

---

## Applier와 변경 적용

이 챕터에서 여러 번 언급했듯이, **Applier**가 이를 담당합니다. 현재 Composer는 Composition 후 기록된 모든 변경 사항을 적용하기 위해 이 추상화에 위임합니다. 이것이 우리가 "구체화(materializing)"라고 알고 있는 프로세스입니다. 이 프로세스는 `Change` 리스트를 실행하고 그 결과로 슬롯 테이블을 업데이트하며, 테이블에 저장된 Composition 데이터를 해석하여 효과적으로 결과를 산출합니다.

**런타임은 Applier가 어떻게 구현되는지에 대해 불가지론적**입니다. 클라이언트 라이브러리가 구현해야 하는 공개 계약에 의존합니다. Applier는 플랫폼과의 통합 지점이므로 사용 사례에 따라 달라지기 때문입니다.

![Applier 계층 구조](/assets/img/compose-internals/chapter3/applier-hierarchy.png)
*Applier 인터페이스와 구현체들. UiApplier는 LayoutNode를, VectorApplier는 VNode를 처리합니다.*

```kotlin
interface Applier<N> {
    val current: N
    fun onBeginChanges() {}
    fun onEndChanges() {}
    fun down(node: N)
    fun up()
    fun insertTopDown(index: Int, instance: N)
    fun insertBottomUp(index: Int, instance: N)
    fun remove(index: Int, count: Int)
    fun move(from: Int, to: Int, count: Int)
    fun clear()
}
```

첫 번째로 보이는 것은 계약 선언의 `N` 타입 파라미터입니다. 이것은 적용하는 노드의 타입입니다. **이것이 Compose가 일반적인 호출 그래프나 노드 트리와 함께 작동할 수 있는 이유**입니다. 항상 사용되는 노드의 타입에 대해 불가지론적입니다.

Applier는 트리를 순회하고, 노드를 삽입, 제거 또는 이동하는 작업을 제공하지만, 그러한 노드의 타입이나 궁극적으로 삽입되는 방법에 대해서는 신경 쓰지 않습니다. 스포일러: 그것은 **노드 자체에 위임**됩니다.

계약은 또한 현재 노드에서 주어진 범위의 모든 자식을 제거하거나, 현재 노드에서 자식을 이동하여 위치를 변경하는 방법을 정의합니다. `clear` 작업은 루트를 가리키고 트리에서 모든 노드를 제거하여 Applier와 루트를 미래의 새 Composition의 대상으로 사용할 수 있도록 준비하는 방법을 정의합니다.

Applier는 모든 노드를 방문하고 적용하면서 전체 트리를 순회합니다. 트리는 **위에서 아래로(top-down)** 또는 **아래에서 위로(bottom-up)** 순회할 수 있습니다. 항상 방문하고 변경 사항을 적용 중인 현재 노드의 참조를 유지합니다.

---

## 트리 구축 성능

트리를 위에서 아래로 구축하는 것과 아래에서 위로 구축하는 것 사이에는 중요한 성능 차이가 있습니다.

### Top-Down 삽입

다음 트리를 고려해봅시다:

```
    R
    |
    B
   / \
  A   C
```

이 트리를 top-down으로 구축하려면:
1. B를 R에 삽입
2. A를 B에 삽입
3. C를 B에 삽입

### Bottom-Up 삽입

Bottom-up 트리 구축은:
1. A와 C를 B에 삽입
2. B 트리를 R에 삽입

Top-down vs bottom-up으로 트리를 구축하는 성능은 상당히 다를 수 있습니다. 이 결정은 사용되는 Applier 구현에 달려 있으며, 일반적으로 새 자식이 삽입될 때마다 알림을 받아야 하는 노드의 수에 의존합니다.

예를 들어, Compose로 표현하려는 그래프가 노드가 삽입될 때마다 모든 조상에게 알려야 한다면:
- **Top-down**: 각 삽입이 여러 노드(부모, 부모의 부모... 등)에 알릴 수 있으며, 새 레벨이 삽입될 때마다 카운트가 기하급수적으로 증가
- **Bottom-up**: 부모가 아직 트리에 연결되지 않았으므로 항상 직접 부모에게만 알림

하지만 전략이 모든 자식에게 알리는 것이라면 반대가 될 수 있습니다. 따라서 항상 표현하는 트리와 변경 사항을 트리 위 또는 아래로 알려야 하는 방법에 따라 달라집니다.

핵심은 삽입을 위해 **한 가지 전략을 선택하되, 둘 다 사용하지 않는 것**입니다.

---

## 변경 적용 방법 - UiApplier 예시

클라이언트 라이브러리는 `Applier` 인터페이스의 구현을 제공하며, 그 예로 Android UI를 위한 `UiApplier`가 있습니다. "노드 적용"이 무엇을 의미하는지, 그리고 그것이 화면에서 볼 수 있는 컴포넌트를 어떻게 산출하는지에 대한 완벽한 예입니다.

```kotlin
internal class UiApplier(
    root: LayoutNode
) : AbstractApplier<LayoutNode>(root) {

    override fun insertTopDown(index: Int, instance: LayoutNode) {
        // 무시됨 - Bottom-up 전략 사용
    }

    override fun insertBottomUp(index: Int, instance: LayoutNode) {
        current.insertAt(index, instance)
    }

    override fun remove(index: Int, count: Int) {
        current.removeAt(index, count)
    }

    override fun move(from: Int, to: Int, count: Int) {
        current.move(from, to, count)
    }

    override fun onClear() {
        root.removeAll()
    }

    override fun onEndChanges() {
        super.onEndChanges()
        (root.owner as? AndroidComposeView)?.clearInvalidObservations()
    }
}
```

`UiApplier`는 **bottom-up 삽입 전략**을 사용합니다(`insertTopDown`은 무시됨). 실제 삽입, 제거, 이동 작업은 `LayoutNode` 자체에 위임됩니다. 이것이 Compose가 플랫폼 불가지론적일 수 있는 이유입니다 - **노드가 스스로를 관리하는 방법을 알고 있습니다**.

---

## 요약

**Composer**는 Compose 런타임의 핵심 중재자입니다:

- Composable 함수 호출을 런타임과 연결
- 변경 사항을 `Change`로 모델링하여 지연 실행
- 슬롯 테이블에 그룹과 데이터 쓰기/읽기
- `remember`, `RecomposeScope`, `SideEffect` 관리
- `CompositionLocal` 및 소스 정보 저장
- **Applier**를 통해 변경 사항을 실제 노드 트리에 적용

다음 글에서는 **Composition**과 **Recomposer**가 어떻게 전체 프로세스를 조율하는지 살펴보겠습니다.

> 이 시리즈는 "Jetpack Compose Internals" 책의 내용을 바탕으로 정리한 것입니다.
