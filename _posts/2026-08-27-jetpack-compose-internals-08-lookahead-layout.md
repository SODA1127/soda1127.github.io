---
layout: post
title: "Jetpack Compose Internals - 8장. LookaheadLayout과 애니메이션 레이아웃"
description: Compose의 LookaheadLayout 내부 동작 원리, 사전 계산(Lookahead Pass) 메커니즘, 그리고 자연스러운 레이아웃 애니메이션 구현 방식을 정리합니다.
image: 'https://imgur.com/o8Jk4Is.png'
category: 'programming'
date: 2026-08-27 09:00:00
tags:
- kotlin
- android
- jetpack compose
- compose internals
- lookahead layout
- layout
- animation
introduction: LookaheadLayout은 측정 및 배치의 최종 목표 상태를 미리 계산하여 부드럽고 정교한 레이아웃 애니메이션 및 공유 요소 전환(Shared Element Transition)을 가능하게 하는 Compose의 핵심 메커니즘입니다.
twitter_text: Compose의 LookaheadLayout 내부 동작 원리와 Lookahead Pass(사전 측정/배치) 시스템을 살펴봅니다.
---

# 8장. LookaheadLayout과 애니메이션 레이아웃

Compose의 레이아웃 시스템은 **단일 통과 측정(Single-pass Measurement)**을 기반으로 설계되어 있습니다. 하지만 화면 전환이나 레이아웃 크기 변경 시, 요소들이 갑자기 점프하듯 변하지 않고 부드럽게 애니메이션되려면 **"최종적으로 도착할 목적지의 크기와 위치"**를 미리 알아야 합니다.

과거에는 이러한 목적지 좌표를 알기 위해 복잡한 매직 넘버를 계산하거나 부자연스러운 트릭을 써야 했습니다. Jetpack Compose는 이를 해결하기 위해 **`LookaheadLayout`**이라는 강력한 사전 계산 메커니즘을 도입했습니다.

이번 장에서는 LookaheadLayout의 등장 배경, 다른 사전 계산 방식과의 차이점, 핵심 API 사용법, 그리고 런타임 내부의 **Lookahead Measure/Layout Pass** 동작 구조를 깊이 있게 살펴봅니다.

---

## LookaheadLayout이란?

LookaheadLayout의 핵심 아이디어는 간단하면서도 혁신적입니다:

> **"트리가 변경되거나 상태가 바뀌었을 때, 애니메이션이 이미 완료되었다고 가정한 미래의 크기와 위치(Lookahead 상태)를 먼저 계산한다."**

이렇게 미리 계산된 최종 크기(`lookaheadSize`)와 최종 좌표(`lookaheadPosition`)를 자식 노드들에게 전달하면, 각 노드는 현재 상태에서 목표 상태까지 값을 점진적으로 보간(Interpolation)하며 스스로를 애니메이션할 수 있습니다.

![Lookahead Shared Element Transition](/assets/img/compose-internals/chapter08/img-000.png)
*LookaheadLayout을 활용한 화면 상태 전환 및 공유 요소(Shared Element) 애니메이션 목표 상태*

위 다이어그램의 검은 사각형들은 전환될 목적지 화면에서 각 공유 요소가 최종적으로 갖게 될 크기와 위치 타깃을 의미합니다. LookaheadLayout이 이 타깃을 사전에 정확히 계산해 주기 때문에 매끄러운 공유 요소 전환이 가능해집니다.

---

## 레이아웃 전환 문제: SmartBox 예제

상태 변경에 따라 `Row`와 `Column` 사이를 전환하는 간단한 컴포저블을 생각해 보겠습니다.

![SmartBox Transition](/assets/img/compose-internals/chapter08/img-002.png)
*레이아웃 구조 전환 시의 목표 영역*

```kotlin
@Composable
fun SmartBox() {
    var vertical by remember { mutableStateOf(false) }

    Box(Modifier.clickable { vertical = !vertical }) {
        if (vertical) {
            Column {
                Text("Text 1")
                Text("Text 2")
            }
        } else {
            Row {
                Text("Text 1")
                Text("Text 2")
            }
        }
    }
}
```

기본적인 조건문 분기에서는 `vertical` 상태가 바뀌는 순간 `Text 1`과 `Text 2`의 위치가 즉각적으로 바뀝니다. 두 텍스트는 논리적으로 동일한 요소이므로, 화면 배치가 바뀔 때 새로운 위치로 부드럽게 이동하는 것이 이상적입니다.

더 나아가 목록 화면(`CharacterList`)에서 상세 화면(`Detail`)으로 화면을 전환할 때도, LookaheadLayout을 사용하면 대상 화면에서의 최종 크기와 위치를 미리 파악하여 애니메이션 타깃으로 삼을 수 있습니다.

---

## Compose의 레이아웃 사전 계산 방식 비교

Compose에는 레이아웃을 사전에 계산하거나 가늠하는 방식이 크게 3가지 존재합니다. 각각의 목적과 동작 방식이 확연히 다릅니다.

| 구분 | SubcomposeLayout | Intrinsics (고유 측정) | LookaheadLayout |
| :--- | :--- | :--- | :--- |
| **주요 목적** | 공간에 따른 조건부 컴포지션 | 부모/자식 간 크기 관계 잠정 파악 | 정밀한 애니메이션 타깃 사전 계산 |
| **실행 시점** | Measure 단계에서 Composition 지연 실행 | 본 측정 직전 잠정 measure 람다 호출 | 일반 Measure/Layout 직전에 Lookahead Pass 선행 |
| **비용/성능** | 컴포지션을 동반하므로 비교적 무거움 | 비교적 가벼우나 매번 계산 | **LookaheadDelegate를 통한 적극적 캐싱** |
| **결과 보장** | 컴포지션 결과 트리가 동적으로 결정됨 | 실제 측정 시 제약 조건이 달라질 수 있음 | **최종 상태로 도달한다는 암묵적 보장** |

- **`SubcomposeLayout`**: `BoxWithConstraints`나 `LazyColumn`처럼 가용 공간에 따라 어떤 서브트리를 빌드할지 결정할 때 사용합니다.
- **`Intrinsics`**: 가장 큰 자식의 높이에 맞추는 등, 1회성 잠정 크기 계산을 위해 사용됩니다.
- **`LookaheadLayout`**: 크기뿐 아니라 **위치(Placement)**까지 사전 계산하며, 트리가 변경되지 않으면 캐시된 데이터를 활용하여 불필요한 재계산을 철저히 방지합니다.

---

## LookaheadLayout의 핵심 API

LookaheadLayout은 `LookaheadLayoutScope` 내에서 콘텐츠를 실행하며, 자식 노드가 사용할 수 있는 두 가지 핵심 Modifier를 제공합니다.

### 1. `Modifier.intermediateLayout`
노드가 다시 측정될 때 호출됩니다. 사전 계산된 최종 크기인 `lookaheadSize`를 인자로 받아, 현재 프레임에서 표시할 **중간 레이아웃(Intermediate Layout)** 크기를 결정하고 자식을 측정합니다.

> ⚠️ `intermediateLayout`의 람다는 Lookahead Pass 중에는 자동으로 건너뛰어집니다. 이 Modifier는 오직 최종 상태로 가는 "중간 상태"를 생성하기 위한 것이기 때문입니다.

### 2. `Modifier.onPlaced`
노드가 다시 배치될 때 호출됩니다. `LookaheadLayout`의 좌표계(`lookaheadScopeCoordinates`)와 자식의 좌표계(`layoutCoordinates`)를 제공받아 로컬 기준의 목표 위치(`targetOffset`)와 현재 위치(`placementOffset`)를 계산할 수 있습니다.

---

## 크기 및 위치 애니메이션 구현 패턴

### 크기 애니메이션: `animateConstraints`

사전 계산된 `lookaheadSize`를 향해 크기를 점진적으로 변화시키는 커스텀 Modifier 예시입니다:

```kotlin
fun Modifier.animateConstraints(lookaheadScope: LookaheadLayoutScope) = composed {
    var sizeAnimation: Animatable<IntSize, AnimationVector2D>? by remember {
        mutableStateOf(null)
    }
    var targetSize: IntSize? by remember { mutableStateOf(null) }

    LaunchedEffect(Unit) {
        snapshotFlow { targetSize }.collect { target ->
            if (target != null && target != sizeAnimation?.targetValue) {
                sizeAnimation?.run {
                    launch { animateTo(target) }
                } ?: Animatable(target, IntSize.VectorConverter).let {
                    sizeAnimation = it
                }
            }
        }
    }

    with(lookaheadScope) {
        this@composed.intermediateLayout { measurable, _, lookaheadSize ->
            targetSize = lookaheadSize
            val (width, height) = sizeAnimation?.value ?: lookaheadSize
            val animatedConstraints = Constraints.fixed(width, height)

            val placeable = measurable.measure(animatedConstraints)
            layout(placeable.width, placeable.height) {
                placeable.place(0, 0)
            }
        }
    }
}
```

1. **`snapshotFlow` & `LaunchedEffect`**: 측정 단계에서 직접 부수효과를 일으키지 않고, 상태 흐름을 통해 안전하게 애니메이션 코루틴을 트리거합니다.
2. **`lookaheadSize` 관찰**: 레이아웃 변화가 감지되면 목표 크기(`targetSize`)가 갱신되어 애니메이션이 시작됩니다.
3. **`Constraints.fixed` 적용**: 매 프레임 애니메이션 진행 값으로 자식을 측정하여 부드러운 크기 변화를 연출합니다.

### 위치 애니메이션: `animatePlacementInScope`

```kotlin
fun Modifier.animatePlacementInScope(lookaheadScope: LookaheadLayoutScope) = composed {
    var offsetAnimation: Animatable<IntOffset, AnimationVector2D>? by remember {
        mutableStateOf(null)
    }
    var placementOffset: IntOffset by remember { mutableStateOf(IntOffset.Zero) }
    var targetOffset: IntOffset? by remember { mutableStateOf(null) }

    LaunchedEffect(Unit) {
        snapshotFlow { targetOffset }.collect { target ->
            if (target != null && target != offsetAnimation?.targetValue) {
                offsetAnimation?.run {
                    launch { animateTo(target) }
                } ?: Animatable(target, IntOffset.VectorConverter).let {
                    offsetAnimation = it
                }
            }
        }
    }

    with(lookaheadScope) {
        this@composed.onPlaced { lookaheadScopeCoordinates, layoutCoordinates ->
            // 로컬 좌표계 기준 목표 위치
            targetOffset = lookaheadScopeCoordinates.localLookaheadPositionOf(layoutCoordinates).round()
            // 로컬 좌표계 기준 현재 위치
            placementOffset = lookaheadScopeCoordinates.localPositionOf(layoutCoordinates, Offset.Zero).round()
        }
        .intermediateLayout { measurable, constraints, _ ->
            val placeable = measurable.measure(constraints)
            layout(placeable.width, placeable.height) {
                val (x, y) = offsetAnimation?.run { value - placementOffset }
                    ?: (targetOffset!! - placementOffset)
                placeable.place(x, y)
            }
        }
    }
}
```

---

## LookaheadLayout의 내부 동작 구조

LookaheadLayout이 내부적으로 2단계(Lookahead Pass → Normal Pass) 측정을 수행하는 구체적인 과정을 다이어그램과 함께 살펴보겠습니다.

### 1. Lookahead Measure Pass

![Lookahead Measure Pass](/assets/img/compose-internals/chapter08/img-004.png)
*Lookahead Measure Pass 흐름도*

노드가 처음 부착되거나 레이아웃 변경이 요청되면:

1. **`LookaheadPassDelegate.measure()` 호출**: 노드가 `LookaheadLayout` 서브트리에 속해 있다면, 일반 측정에 앞서 주황색으로 표시된 **Lookahead Pass**가 시작됩니다.
2. **`LookaheadDelegate` 체인 순회**: 바깥쪽 `LayoutNodeWrapper`부터 시작하여 연결된 Modifier 체인의 `LookaheadDelegate`들을 거쳐 자식 노드의 lookahead 측정을 완료합니다.
3. **일반 Measure Pass 실행**: Lookahead 측정이 끝나면 `MeasurePassDelegate`를 통해 실제 프레임별 측정이 진행됩니다.

### 2. Lookahead Layout Pass

![Lookahead Layout Pass](/assets/img/compose-internals/chapter08/img-006.png)
*Lookahead Layout Pass 흐름도*

측정(Measure)과 마찬가지로 배치(Placement) 단계 역시 동일한 위임(Delegate) 패턴을 따릅니다:

1. **`LookaheadPassDelegate.placeAt()`**: 최종 목표 위치를 계산하여 캐시합니다.
2. **`MeasurePassDelegate.placeAt()`**: 보간된 중간 좌표(`x, y`)를 바탕으로 실제 화면상의 노드 배치를 확정합니다.

### 무효화 범위 최적화

트리 내 일부 노드의 내용만 변경된 경우, LookaheadLayout 전체가 다시 계산되는 것이 아니라 **영향을 받는 서브트리 노드들만 선별적으로 무효화(Invalidation)**됩니다. 또한 부모의 `LookaheadScope`는 자식 노드들에게 자동으로 상속되어 중첩된 LookaheadLayout 구조에서도 단 한 번의 통합 패스로 효율적인 처리가 보장됩니다.

> 💡 **Tip:** 화면 간 공유 요소 전환 시 `movableContentOf`와 함께 사용하면, 컴포저블이 트리 내 다른 위치로 이동하더라도 내부 상태(State)를 유실하지 않고 완벽하게 유지할 수 있습니다.

---

## 요약

| 핵심 개념 | 설명 |
| :--- | :--- |
| **Lookahead Pass** | 애니메이션 완료 후의 미래 크기/위치를 사전 계산하는 패스 |
| **LookaheadDelegate** | 사전 계산된 측정 및 배치 결과를 캐싱하여 성능 낭비 방지 |
| **intermediateLayout** | Lookahead Pass에서는 생략되고, 실제 측정 시 중간 프레임 크기를 생성 |
| **onPlaced** | Scope 기준 상대 좌표를 파악하여 위치 보간 애니메이션 계산 |
| **단일 통과 원칙 준수** | Lookahead Pass 역시 트리를 단 한 번만 순회하므로 고성능 보장 |

LookaheadLayout은 Compose의 선언적 UI 철학을 해치지 않으면서도, 복잡한 레이아웃 애니메이션과 공유 요소 전환을 고성능으로 구현할 수 있도록 만든 핵심 엔진입니다.

---

## 이전 글

- [7장. Advanced Compose Runtime Use Cases](/jetpack-compose-internals-07-advanced-compose-runtime-use-cases/)
- [7장. Layouts and Measurement](/jetpack-compose-internals-07-layouts-and-measurement/)
- [6장. Effects and Effect Handlers](/jetpack-compose-internals-06-effects-and-effect-handlers/)
- [5장. State Snapshot System](/jetpack-compose-internals-05-state-snapshot-system/)
- [4장. Compose UI](/jetpack-compose-internals-04-compose-ui/)
- [3장. Compose Runtime (3)](/jetpack-compose-internals-03-compose-runtime-03/)
- [2장. Compose Compiler](/jetpack-compose-internals-02-compose-compiler/)
- [1장. Composable 함수](/jetpack-compose-internals-01-composable-functions/)
