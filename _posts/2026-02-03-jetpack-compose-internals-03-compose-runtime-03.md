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

private fun composeVector(
    parent: CompositionContext,
    composable: @Composable (viewportWidth: Float, viewportHeight: Float) -> Unit
): Composition {
    val existing = composition
    val next = if (existing == null || existing.isDisposed) {
        Composition(VectorApplier(vector.root), parent) // 벡터용 Applier
    } else {
        existing
    }
    composition = next
    next.setContent {
        composable(vector.viewportWidth, vector.viewportHeight)
    }
    return next
}
```

여기서는 벡터 트리의 루트 노드(`VNode`)를 가리키는 `VectorApplier`라는 다른 Applier 전략이 선택됩니다.

### CompositionContext를 통한 연결

Composition을 생성할 때 부모 `CompositionContext`를 전달할 수 있습니다(nullable). 부모 컨텍스트가 있으면 새 Composition을 기존 Composition에 논리적으로 연결하여, **무효화와 CompositionLocal이 마치 하나의 Composition인 것처럼 해결될 수 있습니다**.

### Composition의 생명주기

Composition이 생성되는 것처럼, 더 이상 필요 없을 때는 반드시 **dispose**해야 합니다(`composition.dispose()`). 이는 UI(또는 다른 사용 사례)가 폐기될 때 발생합니다. Composition은 **owner에 스코핑**되어 있다고 말할 수 있습니다.

---

## 초기 Composition 프로세스

새 Composition이 생성되면 항상 `composition.setContent(content)` 호출이 뒤따릅니다. 이것이 실제로 Composition이 초기에 채워지는(슬롯 테이블이 관련 데이터로 채워지는) 시점입니다.

이 호출은 부모 Composition에 위임되어 초기 Composition 프로세스를 트리거합니다:

```kotlin
override fun setContent(content: @Composable () -> Unit) {
    this.composable = content
    parent.composeInitial(this, composable)
}
```

Subcomposition의 경우 부모는 다른 Composition이 되고, 루트 Composition의 경우 부모는 **Recomposer**가 됩니다. Subcomposition의 경우 `composeInitial` 호출이 루트 Composition에 도달할 때까지 부모에게 계속 위임됩니다.

따라서 `parent.composeInitial(composition, content)`는 결국 `recomposer.composeInitial(composition, content)`로 번역됩니다.

### Recomposer가 수행하는 작업

초기 Composition을 채우기 위해 Recomposer는 다음과 같은 중요한 작업을 수행합니다:

1. **State 스냅샷 생성**: 모든 State 객체의 현재 값에 대한 스냅샷을 생성합니다. 이 값들은 다른 스냅샷의 잠재적 변경으로부터 격리됩니다. 스냅샷은 변경 가능하지만 동시성 안전합니다.

2. **State 수정 제한**: 이 스냅샷의 State 값은 `snapshot.enter(block: () -> T)` 호출 시 전달된 블록 내에서만 수정할 수 있습니다.

3. **읽기/쓰기 옵저버 등록**: 스냅샷을 생성할 때 State 객체에 대한 읽기나 쓰기를 위한 옵저버도 전달하여, Composition이 그러한 작업이 발생할 때 알림을 받을 수 있도록 합니다. 이를 통해 Composition은 영향을 받는 recomposition scope를 "used"로 표시할 수 있으며, 시간이 되면 리컴포지션됩니다.

4. **Composition 실행**: `composition.composeContent(content)`를 블록으로 전달하여 스냅샷에 진입합니다. 여기서 실제 Composition이 발생합니다. 진입하는 행위가 Recomposer에게 Composition 중에 읽거나 쓴 State 객체가 추적될 것임을 알립니다.

5. **Composer에 위임**: Composition 프로세스는 Composer에 위임됩니다.

6. **State 변경 전파**: Composition이 완료되면 State 객체에 대한 변경 사항은 현재 State 스냅샷에만 적용되므로, `snapshot.apply()`를 통해 이러한 변경 사항을 전역 상태로 전파할 때입니다.

### Composer가 수행하는 실제 Composition 프로세스

Composer에 위임된 Composition 프로세스는 대략 다음과 같이 진행됩니다:

1. **재진입 방지**: Composition이 이미 실행 중이면 예외를 던지고 새 Composition을 폐기합니다. 재진입 Composition은 지원되지 않습니다.

2. **무효화 복사**: 대기 중인 무효화가 있으면 Composer가 유지하는 무효화 리스트로 복사합니다.

3. **플래그 설정**: `isComposing` 플래그를 true로 설정합니다.

4. **루트 시작**: `startRoot()`를 호출하여 슬롯 테이블에서 Composition의 루트 그룹을 시작하고 다른 필요한 필드와 구조를 초기화합니다.

5. **콘텐츠 그룹 시작**: 슬롯 테이블에서 콘텐츠를 위한 그룹을 시작합니다.

6. **콘텐츠 실행**: 콘텐츠 람다를 호출하여 모든 변경 사항을 emit합니다.

7. **그룹 종료**: 슬롯 테이블에서 그룹을 종료합니다.

8. **루트 종료**: `endRoot()`를 호출하여 Composition을 종료합니다.

9. **플래그 해제 및 정리**: `isComposing`을 false로 설정하고 임시 데이터를 유지하는 다른 구조를 정리합니다.

---

## 초기 Composition 후 변경 사항 적용

초기 Composition 후, Applier는 프로세스 중에 기록된 모든 변경 사항을 적용하라는 알림을 받습니다: `composition.applyChanges()`.

이는 Composition을 통해 수행되며, 다음 순서로 진행됩니다:

1. `applier.onBeginChanges()` 호출
2. 변경 사항 리스트를 순회하며 각 변경 사항 실행 (Applier와 SlotWriter 인스턴스 전달)
3. 모든 변경 사항 적용 후 `applier.onEndChanges()` 호출

그 후:
- 등록된 모든 `RememberObserver`를 디스패치하여 `RememberObserver` 계약을 구현하는 클래스(예: `LaunchedEffect`, `DisposableEffect`)가 Composition에 진입하거나 떠날 때 알림을 받을 수 있도록 합니다.
- 모든 `SideEffect`가 기록된 순서대로 트리거됩니다.

---

## Composition에 대한 추가 정보

Composition은 리컴포지션을 위한 대기 중인 무효화를 인식하고 있으며, 현재 composing 중인지도 알고 있습니다. 이 지식은 무효화를 즉시 적용할지(composing 중일 때) 아니면 연기할지(그렇지 않을 때) 결정하는 데 사용될 수 있습니다.

런타임은 `ControlledComposition`이라는 Composition의 변형에 의존합니다. 이는 외부에서 제어할 수 있도록 몇 가지 추가 함수를 추가합니다. 이를 통해 **Recomposer가 무효화와 추가 리컴포지션을 조율**할 수 있습니다. `composeContent`나 `recompose` 같은 함수가 좋은 예입니다.

---

## Recomposer의 역할

지금까지 초기 Composition이 어떻게 발생하는지, 그리고 `RecomposeScope`와 무효화에 대해 배웠습니다. 하지만 **Recomposer가 실제로 어떻게 작동하는지**에 대해서는 아직 거의 알지 못합니다.

- Recomposer는 언제 어떻게 생성되나요?
- 무효화를 어떻게 감지하여 자동으로 리컴포지션을 트리거하나요?

Recomposer는 `ControlledComposition`을 제어하며, 필요할 때 리컴포지션을 트리거하여 궁극적으로 업데이트를 적용합니다. 또한 **어떤 스레드에서 compose하거나 recompose할지, 그리고 변경 사항을 적용할 때 어떤 스레드를 사용할지**를 결정합니다.

### Recomposer 생성하기

클라이언트 라이브러리가 Compose에 진입하는 지점은 Composition을 생성하고 `setContent`를 호출하는 것입니다. Composition을 생성할 때 부모를 제공해야 하며, 루트 Composition의 부모는 Recomposer이므로 이때 Recomposer도 생성됩니다.

Android의 경우, Compose UI가 이 진입점을 제공합니다. Composition(내부적으로 자체 Composer 생성)과 부모로 사용할 Recomposer를 생성합니다.

각 플랫폼의 각 잠재적 사용 사례는 자체 Composition을 생성할 가능성이 높으며, 동일한 방식으로 자체 Recomposer도 생성할 것입니다.

### Recomposer의 시작

Android ViewGroup에서 Compose를 사용하려면 `ViewGroup.setContent`를 호출하며, 이는 결국 Recomposer 팩토리에 부모 컨텍스트 생성을 위임합니다.

Recomposer는 **코루틴 스코프**에서 실행되며, `recomposer.runRecomposeAndApplyChanges()`를 호출하여 무효화를 감지하고 리컴포지션을 처리하는 무한 루프를 시작합니다.

### 리컴포지션 프로세스

Recomposer는 다음과 같은 방식으로 리컴포지션을 관리합니다:

1. **무효화 수집**: State가 변경되면 해당 State를 읽는 `RecomposeScope`가 무효화됩니다.

2. **리컴포지션 스케줄링**: 무효화된 scope를 수집하고 리컴포지션을 스케줄링합니다.

3. **스냅샷 생성**: 새 State 스냅샷을 생성하여 리컴포지션 중에 격리된 State 값을 사용합니다.

4. **Composition.recompose() 호출**: 각 무효화된 Composition에 대해 `recompose()`를 호출합니다.

5. **변경 사항 적용**: 리컴포지션이 완료되면 `applyChanges()`를 호출하여 변경 사항을 적용합니다.

### 동시 리컴포지션

Compose는 **동시 리컴포지션**을 지원합니다. 여러 Composition이 병렬로 리컴포지션될 수 있으며, 각각 독립적인 State 스냅샷을 사용합니다. 이는 멀티코어 프로세서를 활용하여 성능을 향상시킵니다.

하지만 변경 사항 적용은 **메인 스레드에서 순차적**으로 수행되어 UI 일관성을 보장합니다.

### Recomposer 상태

Recomposer는 다양한 상태를 가질 수 있습니다:

- **Inactive**: 아직 시작되지 않음
- **Running**: 정상적으로 실행 중
- **Idle**: 실행 중이지만 대기 중인 작업 없음
- **ShutDown**: 종료됨

이러한 상태는 `Recomposer.State` Flow를 통해 관찰할 수 있으며, Composition의 생명주기를 추적하는 데 유용합니다.

---

## 전체 흐름 정리

이제 Compose 런타임의 전체 흐름을 이해할 수 있습니다:

1. **앱 시작**: `setContent` 호출
2. **Recomposer 생성**: 부모 컨텍스트로 사용
3. **Composition 생성**: Applier와 함께 생성
4. **초기 Composition**: Recomposer가 State 스냅샷을 생성하고 Composer가 슬롯 테이블을 채움
5. **변경 사항 적용**: Applier가 변경 사항을 실행하여 노드 트리 구체화
6. **State 변경 감지**: State가 변경되면 해당 RecomposeScope 무효화
7. **리컴포지션**: Recomposer가 무효화된 scope를 리컴포지션
8. **변경 사항 재적용**: 새로운 변경 사항을 다시 적용

이 사이클이 앱이 실행되는 동안 반복되어 UI를 항상 최신 상태로 유지합니다.

---

## 요약

**Composition**과 **Recomposer**는 Compose 런타임의 핵심 조율자입니다:

### Composition
- Composer를 생성하고 소유함
- 슬롯 테이블을 관리하고 변경 사항을 적용함
- 무효화를 추적하고 리컴포지션을 트리거함
- `setContent`를 통해 초기 콘텐츠 설정
- Dispose될 때까지 생명주기 관리

### Recomposer
- 루트 Composition의 부모로 동작
- State 스냅샷을 생성하고 관리
- 무효화된 scope를 수집하고 리컴포지션 스케줄링
- 동시 리컴포지션 조율
- 코루틴 컨텍스트에서 실행되어 스레드 관리

### 핵심 개념
- **State 스냅샷**: 격리된, 동시성 안전한 State 값의 스냅샷
- **무효화**: State 변경 시 RecomposeScope 무효화
- **동시 리컴포지션**: 병렬 리컴포지션으로 성능 향상
- **변경 사항 적용**: 메인 스레드에서 순차적으로 적용

이제 Compose 런타임의 핵심 메커니즘을 모두 이해했습니다. 다음 글에서는 **Compose UI**가 이러한 런타임을 어떻게 활용하여 실제 Android UI를 렌더링하는지 살펴보겠습니다.

> 이 시리즈는 "Jetpack Compose Internals" 책의 내용을 바탕으로 정리한 것입니다.
