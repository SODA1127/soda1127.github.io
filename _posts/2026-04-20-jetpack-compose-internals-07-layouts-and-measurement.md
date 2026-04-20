---
layout: post
title: "Jetpack Compose Internals 07: Layouts and Measurement"
description: "Jetpack Compose의 레이아웃 시스템과 측정(Measurement) 프로세스에 대해 깊이 있게 알아봅니다."
image: /assets/img/compose-internals/chapter27/img-000.png
date: 2026-04-20
tags: [Android, Jetpack Compose, Layout, Measurement, Deep Dive]
---

Jetpack Compose Internals의 7장, **Layouts and Measurement**에 대한 분석 내용을 정리합니다. 이 장에서는 Compose가 화면의 UI 요소들을 어떻게 배치하고, 크기를 결정하는지 그 내부 메커니즘을 다룹니다.

## 1. Compose 레이아웃의 원칙: Single Pass Measurement

Compose 레이아웃 시스템의 가장 큰 특징 중 하나는 **단일 패스 측정(Single Pass Measurement)**입니다. 기존 View 시스템에서 발생할 수 있었던 복잡한 중첩 레이아웃에서의 성능 저하(Exponential measurement overhead)를 방지하기 위해, 모든 노드는 오직 한 번만 측정됩니다.

![](/assets/img/compose-internals/chapter27/img-001.png)

## 2. Layout 단계의 세 가지 프로세스

Compose가 UI를 그릴 때 레이아웃 단계는 다음 세 가지 과정을 거칩니다.

1.  **아이들 측정 (Measuring Children):** 각 자식 노드들에게 가용 공간(Constraints)을 전달하고 각자의 크기를 결정하게 합니다.
2.  **자신 크기 결정 (Deciding own size):** 자식들의 크기 정보를 바탕으로 부모 노드 자신의 크기를 설정합니다.
3.  **자식 배치 (Placing Children):** 부모의 영역 내에서 자식들의 위치(x, y 좌표)를 지정합니다.

![](/assets/img/compose-internals/chapter27/img-002.png)

## 3. Constraints (제약 조건)

부모는 자식에게 `Constraints` 객체를 전달합니다. 이는 자식이 가질 수 있는 최소/최대 너비와 높이를 정의합니다.

-   `minWidth`, `maxWidth`
-   `minHeight`, `maxHeight`

자식 노드는 이 제약 조건 범위 내에서 자신의 크기를 결정해야 하며, 이를 위반할 수 없습니다.

## 4. Custom Layout 만들기

`Layout` 컴포저블을 사용하면 커스텀 레이아웃을 직접 구현할 수 있습니다. `MeasurePolicy`를 통해 자식들의 `Measurable` 리스트를 받고, 이를 `Placeable`로 변환한 뒤 배치하는 로직을 작성하게 됩니다.

```kotlin
Layout(
    content = content,
    modifier = modifier
) { measurables, constraints ->
    // 1. 자식 측정
    val placeables = measurables.map { it.measure(constraints) }
    
    // 2. 부모 크기 결정
    layout(width, height) {
        // 3. 자식 배치
        placeables.forEach { it.placeRelative(x, y) }
    }
}
```

![](/assets/img/compose-internals/chapter27/img-005.png)

---

이번 장을 통해 Compose가 어떻게 선언적인 코드를 효율적인 픽셀 데이터로 변환하는지 이해할 수 있었습니다. 다음 8장에서는 더욱 심화된 배치 전략에 대해 다뤄보겠습니다.
