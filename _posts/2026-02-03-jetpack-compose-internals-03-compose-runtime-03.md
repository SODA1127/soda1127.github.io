---
layout: post
title: "Jetpack Compose Internals - 3장. Compose 런타임 (3) - Composition과 Recomposer"
description: Composition과 Recomposer가 어떻게 전체 Compose 프로세스를 조율하는지 살펴봅니다.
image: 'https://imgur.com/o8Jk4Is.png'
category: 'programming'
date: 2026-02-03 23:45:00
tags:
- kotlin
- android
- jetpack compose
- compose internals
- runtime
- composition
- recomposer
introduction: Composition 생성부터 Recomposer를 통한 리컴포지션 프로세스까지, Compose 런타임의 전체 흐름을 완성합니다.
twitter_text: Composition과 Recomposer가 어떻게 전체 Compose 프로세스를 조율하는지 살펴봅니다.
---

# 3장. Compose 런타임 (3) - Composition과 Recomposer

지난 글에서 **Composer**가 어떻게 변경 사항을 기록하고 슬롯 테이블과 상호작용하는지 배웠습니다. 하지만 아직 한 가지 중요한 질문이 남아 있습니다: **누가 Composition을 생성하고, 언제 시작되며, 어떻게 리컴포지션이 트리거될까요?**

이번 글에서는 **Composition**과 **Recomposer**가 어떻게 전체 프로세스를 조율하는지 알아보겠습니다.

---

## Composition의 역할

지금까지 Composer가 Composition에 대한 참조를 가지고 있다고 배웠습니다. 하지만 실제로는 그 반대입니다. **Composition이 생성될 때 Composer를 내부적으로 생성합니다**. Composer는 `currentComposer` 메커니즘을 통해 접근 가능해지며, Composition이 관리하는 트리를 생성하고 업데이트하는 데 사용됩니다.

![Composition and Composer Relationship](/assets/img/compose-internals/chapter3/part3/diag-01.png)

### Compose 런타임의 진입점

클라이언트 라이브러리가 Compose 런타임에 진입하는 방법은 두 가지로 나뉩니다:

1. **Composable 함수 작성**: 함수들이 필요한 정보를 emit하여 런타임과 연결
2. **setContent 호출**: 플랫폼과의 통합 레이어로, 여기서 Composition이 생성되고 시작됨

Composable 함수는 아무리 잘 작성해도 Composition 프로세스 없이는 실행되지 않습니다. 따라서 `setContent`가 필수적인 진입점이 됩니다.

---

## Composition 생성하기

Android의 경우, `ViewGroup.setContent`를 호출하면 새로운 Composition이 생성됩니다:

```kotlin
internal fun ViewGroup.setContent(
    parent: CompositionContext,
    content: @Composable () -> Unit
): Composition {
    val composeView = ...
    return doSetContent(composeView, parent, content)
}

private fun doSetContent(
    owner: AndroidComposeView,
    parent: CompositionContext,
    content: @Composable () -> Unit
): Composition {
    val original = Composition(UiApplier(owner.root), parent)
    val wrapped = WrappedComposition(owner, original)
    wrapped.setContent(content)
    return wrapped
}
```

### WrappedComposition의 역할

`WrappedComposition`은 Composition을 `AndroidComposeView`에 연결하여 Android View 시스템과 직접 통합하는 데코레이터입니다. 다음과 같은 작업을 수행합니다:

- 키보드 가시성 변화나 접근성을 추적하는 controlled effects 시작
- Android Context 정보를 `CompositionLocal`로 노출
- `LifecycleOwner`, `SavedStateRegistryOwner`, View 등을 Composable 함수에서 암시적으로 사용 가능하게 만듦

### Applier 선택

`UiApplier(owner.root)`가 Composition에 전달되는 것을 주목하세요. 트리의 루트 `LayoutNode`를 가리키는 Applier 인스턴스입니다. **이것이 클라이언트 라이브러리가 Applier 구현을 선택하는 첫 번째 명시적인 지점**입니다.

### 다른 Composition 예시: VectorPainter

Compose UI의 또 다른 좋은 예는 `VectorPainter`입니다. 벡터를 화면에 그리는 데 사용되며, 자체 Composition을 생성하고 유지합니다:

```kotlin
@Composable
internal fun RenderVector(
    name: String,
    viewportWidth: Float,
    viewportHeight: Float,
    content: @Composable (viewportWidth: Float, viewportHeight: Float) -> Unit
) {
    val composition = composeVector(rememberCompositionContext(), content)
    
    DisposableEffect(composition) {
        onDispose {
            composition.dispose() // 끝날 때 dispose 필수!
        }
    }
}
```

---

## 초기 Composition 프로세스

새 Composition이 생성되면 항상 `composition.setContent(content)` 호출이 뒤따릅니다. 이것이 실제로 Composition이 초기에 채워지는(슬롯 테이블이 관련 데이터로 채워지는) 시점입니다.

![Initial Composition Process](/assets/img/compose-internals/chapter3/part3/diag-02.png)

이 호출은 부모 Composition에 위임되어 초기 Composition 프로세스를 트리거합니다:

```kotlin
override fun setContent(content: @Composable () -> Unit) {
    this.composable = content
    parent.composeInitial(this, composable)
}
```

Subcomposition의 경우 부모는 다른 Composition이 되고, 루트 Composition의 경우 부모는 **Recomposer**가 됩니다. Subcomposition의 경우 `composeInitial` 호출이 루트 Composition에 도달할 때까지 부모에게 계속 위임됩니다.

---

## Recomposer의 역할

지금까지 초기 Composition이 어떻게 발생하는지, 그리고 `RecomposeScope`와 무효화에 대해 배웠습니다. 하지만 **Recomposer가 실제로 어떻게 작동하는지**에 대해서는 아직 거의 알지 못합니다.

Recomposer는 `ControlledComposition`을 제어하며, 필요할 때 리컴포지션을 트리거하여 궁극적으로 업데이트를 적용합니다. 또한 **어떤 스레드에서 compose하거나 recompose할지, 그리고 변경 사항을 적용할 때 어떤 스레드를 사용할지**를 결정합니다.

![Recomposer Context and Lifecycle](/assets/img/compose-internals/chapter3/part3/diag-03.png)

### Recomposer 생성하기

클라이언트 라이브러리가 Compose에 진입하는 지점은 Composition을 생성하고 `setContent`를 호출하는 것입니다. Composition을 생성할 때 부모를 제공해야 하며, 루트 Composition의 부모는 Recomposer이므로 이때 Recomposer도 생성됩니다.

Android의 경우, Compose UI가 이 진입점을 제공합니다. Composition(내부적으로 자체 Composer 생성)과 부모로 사용할 Recomposer를 생성합니다.

### 리컴포지션 프로세스

Recomposer는 다음과 같은 방식으로 리컴포지션을 관리합니다:

1. **무효화 수집**: State가 변경되면 해당 State를 읽는 `RecomposeScope`가 무효화됩니다.
2. **리컴포지션 스케줄링**: 무효화된 scope를 수집하고 리컴포지션을 스케줄링합니다.
3. **스냅샷 생성**: 새 State 스냅샷을 생성하여 리컴포지션 중에 격리된 State 값을 사용합니다.
4. **Composition.recompose() 호출**: 각 무효화된 Composition에 대해 `recompose()`를 호출합니다.
5. **변경 사항 적용**: 리컴포지션이 완료되면 `applyChanges()`를 호출하여 변경 사항을 적용합니다.

### Recomposer 상태

Recomposer는 다양한 상태를 가질 수 있습니다:

![Recomposer State Machine](/assets/img/compose-internals/chapter3/part3/diag-04.png)

- **ShutDown**: Recomposer가 취소되고 정리 작업이 완료된 상태. 더 이상 사용할 수 없습니다.
- **Inactive**: Recomposer가 무효화를 무시하고 리컴포지션을 트리거하지 않는 상태.
- **Idle**: 무효화를 추적하고 있지만 현재 수행할 작업이 없는 상태.
- **PendingWork**: 보류 중인 작업이 있어 수행 중이거나 수행할 기회를 기다리는 상태.

---

## 요약

**Composition**과 **Recomposer**는 Compose 런타임의 핵심 조율자입니다. Composition은 Composer를 통해 데이터의 구조를 관리하고, Recomposer는 스냅샷 시스템과 연동하여 언제 어떤 스레드에서 UI를 업데이트할지 조율합니다.

이제 Compose 런타임의 핵심 메커니즘을 모두 이해했습니다. 다음 글에서는 **Compose UI**가 이러한 런타임을 어떻게 활용하여 실제 Android UI를 렌더링하는지 살펴보겠습니다.

> 이 시리즈는 "Jetpack Compose Internals" 책의 내용을 바탕으로 정리한 것입니다.
