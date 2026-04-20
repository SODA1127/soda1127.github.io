---
layout: post
title: "Jetpack Compose Internals - 7장. Layouts and Measurement"
description: Compose의 레이아웃 단계와 측정(Measurement) 프로세스, 제약 조건(Constraints) 전달 방식을 정리합니다.
image: 'https://imgur.com/o8Jk4Is.png'
category: 'programming'
date: 2026-04-20 13:00:00
tags:
- kotlin
- android
- jetpack compose
- compose internals
- layout
- measurement
- constraints
introduction: Layouts and Measurement는 UI 트리의 노드들이 크기를 결정하고 화면에 배치되는 Compose의 핵심 렌더링 단계입니다.
twitter_text: Compose의 Layout 단계와 단일 통과(Single-pass) 측정 시스템의 내부 동작 원리를 정리합니다.
---

# 7장. Layouts and Measurement

Jetpack Compose에서 UI를 화면에 그리기 위해서는 각 요소의 **크기를 측정(Measure)**하고 **위치를 결정(Placement)**하는 과정이 필요합니다. 이번 장에서는 Compose 레이아웃 시스템의 핵심인 **단일 통과 측정(Single-pass Measurement)**과 **제약 조건(Constraints)**의 흐름을 정리합니다.

---

## Layout 단계의 3단계 흐름

Compose 레이아웃은 크게 세 단계로 진행됩니다:

1. **자식 측정 (Measure children):** 각 요소가 자신의 자식 노드들을 측정합니다.
2. **자신의 크기 결정 (Decide own size):** 자식들의 측정 결과를 바탕으로 자신의 크기를 결정합니다.
3. **자식 배치 (Place children):** 결정된 크기 내에서 자식들의 위치를 지정합니다.

![Layout Flow](/assets/img/compose-internals/chapter07/img-000.png)

이 과정은 트리 구조를 따라 **단 한 번만(Single-pass)** 발생하므로 성능상 매우 효율적입니다.

---

## Constraints (제약 조건)

부모는 자식에게 `Constraints`를 전달하여 자식이 가질 수 있는 **최대/최소 너비와 높이**를 제한합니다.

- `minWidth`, `maxWidth`
- `minHeight`, `maxHeight`

자식은 이 제약 조건 범위 내에서 자신의 크기를 결정해야 하며, 부모의 허락 없이 범위를 벗어날 수 없습니다.

---

## 단일 통과 측정의 중요성

기존 View 시스템에서는 복잡한 레이아웃(예: `RelativeLayout`)이 중첩될 때 다중 측정이 발생하여 성능 저하의 원인이 되곤 했습니다. Compose는 **Single-pass**를 강제하여 깊은 트리 구조에서도 예측 가능한 성능을 보장합니다.

![Single Pass](/assets/img/compose-internals/chapter07/img-002.png)

---

## Layout Composable

모든 레이아웃의 기본은 `Layout` 컴포저블입니다. `Box`, `Column`, `Row` 등도 내부적으로는 이 `Layout`을 호출하여 측정 및 배치 로직을 구현합니다.

```kotlin
Layout(
    content = content,
    modifier = modifier
) { measurables, constraints ->
    // 1. 자식 측정
    val placeables = measurables.map { measurable ->
        measurable.measure(constraints)
    }

    // 2. 부모 크기 결정 및 배치
    layout(width, height) {
        placeables.forEach { placeable ->
            placeable.placeRelative(0, 0)
        }
    }
}
```

---

## 요약

7장에서는 Compose가 화면을 구성하는 물리적인 방식을 다루었습니다.

1. ✅ **Single-pass:** 모든 노드는 단 한 번만 측정됩니다.
2. ✅ **Constraints:** 부모에서 자식으로 제약 조건이 흐릅니다.
3. ✅ **Measure & Place:** 크기 측정 후 위치 배치가 일어납니다.

정확한 측정과 배치의 원리를 이해하면 커스텀 레이아웃을 작성하거나 성능 문제를 진단할 때 큰 도움이 됩니다.

---

### 다음 장 예고

다음 장에서는 **8장. Custom Layouts**를 다룹니다:
- ✨ 커스텀 레이아웃 직접 만들기
- ✨ 고정 관념을 깨는 UI 배치 전략
- ✨ 고성능 UI를 위한 최적화 팁
