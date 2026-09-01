---
layout: post
title: "Jetpack Compose Internals - 9장. Modifier 체인, 노드 드로잉 및 시맨틱스(Semantics)"
description: Compose의 Modifier 체인 구조, materialize 및 ComposedModifier 동작 원리, LayoutNodeWrapper 체인 재구축, 하드웨어 가속 렌더링(RenderNodeLayer vs ViewLayer), 그리고 시맨틱스(Semantics) 트리의 내부 메커니즘을 상세히 분석합니다.
image: 'https://imgur.com/o8Jk4Is.png'
category: 'programming'
date: 2026-09-01 09:00:00
tags:
- kotlin
- android
- jetpack compose
- compose internals
- modifiers
- layout node
- drawing
- semantics
- accessibility
introduction: Compose의 Modifier 체인은 불변 연결 리스트 구조로 관리되며, 런타임에서 LayoutNodeWrapper 체인으로 재구성되어 측정, 레이아웃, 드로잉, 그리고 접근성 시맨틱스 트리를 구동합니다.
twitter_text: Compose의 Modifier 체인 모델링, 노드 드로잉 파이프라인, 그리고 접근성을 지원하는 시맨틱스 트리의 내부 동작 원리를 정리합니다.
---

# 9장. Modifier 체인, 노드 드로잉 및 시맨틱스(Semantics)

Jetpack Compose에서 컴포저블의 모양과 동작, 인터랙션을 정의할 때 가장 빈번하게 사용하는 것이 바로 **`Modifier`**입니다. 단순한 데코레이터처럼 보이지만, 내부적으로 Modifier는 레이아웃 측정, 터치 이벤트 처리, 하드웨어 드로잉 계층 구성, 그리고 스크린 리더와 테스트 프레임워크를 위한 **시맨틱스(Semantics) 트리** 구축까지 담당하는 핵심 인프라입니다.

이번 장에서는 Modifier 체인이 내부적으로 어떻게 모델링되고 `LayoutNode`에 주입되는지, 컴포즈가 이를 바탕으로 화면을 렌더링하고 시맨틱스 정보를 운영체제에 전달하는 전체 파이프라인을 깊이 있게 살펴봅니다.

---

## 1. Modifier 체인의 내부 모델링

`Modifier` 인터페이스는 UI 컴포저블을 장식(decorate)하거나 부가 동작을 추가하는 **불변(Immutable) 요소들의 컬렉션**입니다.

Modifier는 다음과 같은 기본 기능들을 제공합니다:
- **`then(other)`**: 두 개의 Modifier를 연결하여 새로운 체인을 생성합니다.
- **`foldIn(initial, operation)`**: 체인의 머리(Head)부터 꼬리(Tail) 방향으로 누적 연산을 수행합니다.
- **`foldOut(initial, operation)`**: 체인의 꼬리(Tail)부터 머리(Head) 방향(역순)으로 누적 연산을 수행합니다.
- **`all(predicate)` / `any(predicate)`**: 체인 내의 요소들이 특정 조건을 만족하는지 검사합니다.

### 불변 연결 리스트: `CombinedModifier`

코드에서 여러 Modifier를 연속해서 호출하면, 내부적으로 `CombinedModifier`라는 연결 리스트 형태의 이진 트리 구조가 생성됩니다.

```kotlin
// Modifier 체이닝 예시
Box(
    modifier
        .then(indentMod)
        .fillMaxWidth()
        .height(targetThickness)
        .background(color = color)
)
```

위의 체이닝은 내부적으로 `CombinedModifier`가 중첩된 형태로 표현됩니다:

```kotlin
class CombinedModifier(
    private val outer: Modifier,
    private val inner: Modifier
) : Modifier
```

결과적으로 `CombinedModifier(a, CombinedModifier(b, CombinedModifier(c, d)))`와 같이 바깥쪽(`outer`)이 안쪽(`inner`)을 감싸는 중첩 연결 구조가 형성됩니다.

---

## 2. LayoutNode에 Modifier 주입하기

컴포저블 `Layout`이 실행되면 런타임의 `update` 람다를 통해 생성된 `LayoutNode`에 각종 속성들이 주입됩니다.

```kotlin
@Composable inline fun Layout(
    modifier: Modifier = Modifier,
    measurePolicy: MeasurePolicy
) {
    val density = LocalDensity.current
    val layoutDirection = LocalLayoutDirection.current
    val viewConfiguration = LocalViewConfiguration.current

    // 1. Modifier 체인을 머티리얼라이즈(Materialize)
    val materialized = currentComposer.materialize(modifier)

    ReusableComposeNode<ComposeUiNode, Applier<Any>>(
        factory = { LayoutNode() },
        update = {
            set(measurePolicy, { this.measurePolicy = it })
            set(density, { this.density = it })
            set(layoutDirection, { this.layoutDirection = it })
            set(viewConfiguration, { this.viewConfiguration = it })
            set(materialized, { this.modifier = it }) // 2. LayoutNode에 주입
        }
    )
}
```

### Modifier 구체화 (Materialization)와 `ComposedModifier`

Compose UI에는 두 가지 유형의 Modifier가 있습니다:
1. **표준 Modifier (Standard Modifier)**: 상태가 없는 순수 객체 (예: `padding`, `size`, `background` 등).
2. **상태를 가지는 Modifier (Composed Modifier)**: `remember`로 상태를 보존하거나 `CompositionLocal`을 읽어야 하는 Modifier (예: `clickable`, `pointerInput`, `scrollable`, `focusable` 등).

```kotlin
private open class ComposedModifier(
    inspectorInfo: InspectorInfo.() -> Unit,
    val factory: @Composable Modifier.() -> Modifier
) : Modifier.Element, InspectorValueInfo(inspectorInfo)
```

`LayoutNode`는 컴포지션 컨텍스트를 직접 알지 못하므로 `ComposedModifier`를 바로 처리할 수 없습니다. 따라서 노드에 설정하기 전 **`currentComposer.materialize(modifier)`**를 호출하여 `factory` 컴포저블 람다를 실행하고, 내부의 `remember`나 `CompositionLocal` 조회를 완료한 표준 Modifier 체인으로 변환(Materialize)합니다.

---

## 3. LayoutNode의 Modifier 수용 및 체인 재구축

새로운 Modifier 체인이 `LayoutNode.modifier`에 할당되면, 노드는 성능 최적화를 위해 기존 Wrapper들을 최대한 재사용하면서 **`LayoutNodeWrapper` 체인**을 갱신합니다.

### 1단계: Head → Tail 순회 (`foldIn`) 및 캐시 매칭
1. 기존에 설정되어 있던 Modifier 인스턴스들을 임시 캐시 목록에 보관합니다.
2. `foldIn`을 통해 새 Modifier 체인을 머리(Head)부터 순회합니다.
3. 새 Modifier와 동일한 인스턴스가 캐시에 존재하면 해당 Modifier 및 상위 조상들을 `reusable`로 마킹합니다.

### 2단계: Tail → Head 역순 순회 (`foldOut`) 및 Wrapper 체인 조립
새로운 `LayoutNodeWrapper` 체인을 구축하기 위해 꼬리(Tail)부터 역방향으로 접어 올립니다(`foldOut`).

![Modifier Resolution for Measuring](/assets/img/compose-internals/chapter09/img-000.png)
*Modifier 체인 순회 및 LayoutNodeWrapper 체인 재구축 구조*

- **시작점**: 가장 안쪽의 코어 노드를 감싸는 `innerLayoutNodeWrapper`에서 시작합니다.
- **Wrapper 생성/재사용**:
  - 캐시에서 재사용 가능한 Wrapper가 발견되면 그대로 링크하고 캐시에서 제거합니다.
  - 재사용할 수 없는 새로운 Modifier라면 해당 역할에 맞는 `LayoutNodeWrapper` 하위 타입(`ModifiedLayoutNode`, `ModifiedDrawNode` 등)으로 래핑하여 연결합니다.
- **완료**: 가장 바깥쪽인 `outerLayoutNodeWrapper`까지 도달하면 부모 노드의 내부 Wrapper와 연결하고, 사용되지 않고 남은 이전 Wrapper들의 `detach()`를 호출한 뒤 재측정(Remeasure)을 요청합니다.

---

## 4. 노드 트리 드로잉 파이프라인

측정(Measure)과 배치(Layout)가 완료되면 최종적으로 화면에 픽셀을 그리는 드로잉(Draw) 패스가 진행됩니다.

Android 플랫폼에서 드로잉은 `Owner` 역할을 하는 `AndroidComposeView`의 `dispatchDraw()`에서 시작되어 트리의 루트 `LayoutNode.draw(canvas)`로 전달됩니다.

### Wrapper 기반 드로잉 디스패치 순서

`LayoutNodeWrapper` 체인은 바깥쪽부터 안쪽 순서로 드로잉을 디스패치합니다:

1. **Drawing Layer 존재 시**: Wrapper에 연결된 하드웨어 드로잉 레이어(`RenderNodeLayer` 또는 `ViewLayer`)에 그리기를 위임합니다.
2. **Draw Modifier 존재 시**: 별도 레이어가 없지만 `DrawModifier`가 연결되어 있다면 해당 Modifier 체인을 순차적으로 실행합니다.
3. **다음 Wrapper 전파**: 위 작업이 끝나면 체인의 다음 Wrapper `draw()`를 재귀 호출하여 최종 자식 노드까지 드로잉을 마칩니다.

### 하드웨어 가속 레이어: `RenderNodeLayer` vs `ViewLayer`

| 레이어 유형 | 설명 및 특징 | 사용 조건 |
| :--- | :--- | :--- |
| **`RenderNodeLayer`** | 플랫폼 `RenderNode`를 직접 활용하는 고성능 하드웨어 가속 레이어. 한 번 디스플레이 리스트를 기록하면 렌더 스레드에서 GPU를 통해 초고속 재드로잉 수행. | Android API 23+ (기본 렌더링 계층) |
| **`ViewLayer`** | `View` 객체를 생성하여 `ViewGroup.drawChild` 메커니즘을 우회 활용하는 폴백 레이어. 그림자 및 `elevation mode`를 지원. | RenderNode 직접 생성이 불가능한 구형 플랫폼 환경 |

### 드로잉 단계에서의 스냅샷 상태 관찰

드로잉 람다 내부에서 `SnapshotState`를 읽는 경우, 해당 상태가 변경되면 컴포지션이나 레이아웃 측정 단계를 건너뛰고 **해당 드로잉 레이어만 단독으로 무효화(Invalidation)**되어 불필요한 연산 없이 화면이 빠르게 다시 그려집니다.

---

## 5. 시맨틱스(Semantics) 트리와 접근성 시스템

Jetpack Compose는 렌더링 트리와 병렬로 **접근성(TalkBack) 서비스 및 UI 테스트 프레임워크**가 해석할 수 있는 **시맨틱스(Semantics) 트리**를 유지 관리합니다.

### 시맨틱스 루트 노드 구성

`AndroidComposeView`가 생성될 때, 루트 `LayoutNode`에는 기본적으로 3가지 핵심 Modifier가 설정됩니다:

```kotlin
override val root = LayoutNode().also {
    it.measurePolicy = RootMeasurePolicy
    it.modifier = Modifier
        .then(semanticsModifier)       // 시맨틱스 트리 초기화
        .then(_focusManager.modifier)   // 접근성 포커스 이동 관리
        .then(keyInputModifier)         // 키보드 입력 및 내비게이션 파이프라인
    it.density = density
}
```

### `Modifier.semantics`와 고유 ID

개발자가 특정 컴포저블에 의미론적 정보를 추가할 때는 `Modifier.semantics`를 사용합니다:

```kotlin
fun Modifier.semantics(
    mergeDescendants: Boolean = false,
    properties: (SemanticsPropertyReceiver.() -> Unit)
): Modifier = composed(
    inspectorInfo = debugInspectorInfo {
        name = "semantics"
        this.properties["mergeDescendants"] = mergeDescendants
        this.properties["properties"] = properties
    }
) {
    val id = remember { SemanticsModifierCore.generateSemanticsId() }
    SemanticsModifierCore(
        id = id,
        mergeDescendants = mergeDescendants,
        clearAndSetSemantics = false,
        properties = properties
    )
}
```

각 시맨틱스 노드는 정적으로 순차 증가하는 고유 정수 `id`를 부여받아 시스템 접근성 노드와 매핑됩니다.

### 시맨틱스 변경 통지 파이프라인

시맨틱스 정보가 변경되면 `AccessibilityDelegateCompat`을 통해 안드로이드 프레임워크로 전달됩니다:

```
[ 시맨틱스 변경 감지 ]
        │
        ▼
[ 메인 루퍼 Handler Action 등록 ]
        │
        ├─► 1. 구조적 변경 (노드 추가/삭제) ──► ConflatedChannel ──► 100ms 배치 코루틴 처리
        │
        └─► 2. 속성값 변경 (텍스트/상태) ──► ViewParent.requestSendAccessibilityEvent()
```

1. **구조적 변경 (Structural Changes)**: 자식 노드가 추가되거나 제거된 경우, `ConflatedChannel`을 통해 100ms 단위로 이벤트를 배치(Batch) 수집하여 시스템에 통지합니다.
2. **속성 변경 (Property Changes)**: 텍스트나 상태값이 변경된 경우 네이티브 접근성 이벤트를 즉시 발송합니다.

### 통합 트리(Merged) vs 비통합 트리(Unmerged)

Compose는 두 종류의 시맨틱스 트리를 제공합니다:
- **비통합 트리 (Unmerged Tree)**: 개별 컴포저블 노드의 시맨틱스 정보를 그대로 유지합니다 (주로 상세 테스트에서 활용).
- **통합 트리 (Merged Tree)**: `mergeDescendants = true` 속성을 통해 복잡한 자식 컴포넌트들을 하나의 논리적 단위로 병합합니다.

```kotlin
// 예: ContentDescription 병합 정책
val ContentDescription = SemanticsPropertyKey<List<String>>(
    name = "ContentDescription",
    mergePolicy = { parentValue, childValue ->
        parentValue?.toMutableList()?.also { it.addAll(childValue) } ?: childValue
    }
)
```

목록 아이템 내부의 수많은 텍스트와 아이콘을 TalkBack이 하나씩 따로 읽는 대신, 부모 컨테이너가 `mergeDescendants = true`를 선언하면 자식들의 `ContentDescription`이 리스트로 결합되어 한 번에 자연스럽게 음성 출력됩니다.

---

## 요약

| 핵심 구성요소 | 주요 역할 및 메커니즘 |
| :--- | :--- |
| **CombinedModifier** | 불변 이진 트리 구조로 선언적 Modifier 체이닝을 표현 |
| **currentComposer.materialize** | `ComposedModifier`의 팩토리 람다를 실행하여 표준 Modifier 체인으로 구체화 |
| **LayoutNodeWrapper 체인** | `foldOut`을 통해 꼬리부터 래퍼를 조립하고 캐시를 활용해 재사용 극대화 |
| **RenderNodeLayer** | 디스플레이 리스트 캐싱 및 GPU 하드웨어 가속 렌더링 지원 |
| **Semantics Tree** | 병합 정책(`mergePolicy`)을 기반으로 접근성 및 테스트용 메타데이터 제공 |

Modifier 체인은 단순한 속성 집합을 넘어, 컴포즈 UI 트리가 화면에 그려지고(Render), 사용자와 상호작용하며(Input), 접근성 서비스와 소통하는 모든 경로의 핵심 교량 역할을 수행합니다.

---

## 이전 글

- [8장. LookaheadLayout과 애니메이션 레이아웃](/jetpack-compose-internals-08-lookahead-layout/)
- [7장. Advanced Compose Runtime Use Cases](/jetpack-compose-internals-07-advanced-compose-runtime-use-cases/)
- [7장. Layouts and Measurement](/jetpack-compose-internals-07-layouts-and-measurement/)
- [6장. Effects and Effect Handlers](/jetpack-compose-internals-06-effects-and-effect-handlers/)
- [5장. State Snapshot System](/jetpack-compose-internals-05-state-snapshot-system/)
- [4장. Compose UI](/jetpack-compose-internals-04-compose-ui/)
- [3장. Compose Runtime (3)](/jetpack-compose-internals-03-compose-runtime-03/)
- [2장. Compose Compiler](/jetpack-compose-internals-02-compose-compiler/)
- [1장. Composable 함수](/jetpack-compose-internals-01-composable-functions/)
