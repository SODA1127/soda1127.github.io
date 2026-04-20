---
layout: post
title: "Jetpack Compose Internals - 7장. Layouts and Measurement"
description: "Jetpack Compose의 단일 패스 측정(Single Pass Measurement) 원칙과 커스텀 레이아웃 구현 메커니즘을 정리합니다."
image: 'https://imgur.com/o8Jk4Is.png'
category: 'programming'
date: 2026-04-20 12:00:00
tags:
- kotlin
- android
- jetpack compose
- compose internals
- layout
- measurement
introduction: "Layouts and Measurement 단계는 Composable 트리가 픽셀 데이터로 변환되기 전, 각 요소의 크기와 위치를 결정하는 핵심 과정입니다."
twitter_text: "Compose의 Single Pass Measurement와 Layout 프로세스를 한 번에 정리합니다."
---

# 7장. Layouts and Measurement

Jetpack Compose는 기존 안드로이드 뷰 시스템의 성능 병목 중 하나였던 '다중 측정(Multi-pass measurement)' 문제를 해결하기 위해 **단일 패스 측정(Single Pass Measurement)** 원칙을 도입했습니다. 이번 장에서는 레이아웃이 결정되는 3단계 과정과 제약 조건(Constraints) 시스템을 살펴봅니다.

---

## Single Pass Measurement 원칙

Compose의 모든 노드는 **단 한 번만 측정**됩니다. 이는 성능 최적화를 위한 핵심 설계로, 자식 노드가 자신의 크기를 결정하기 위해 부모를 다시 호출하거나 부모가 여러 번 자식을 측정하는 '지수적 비용'을 방지합니다.

![Layout Principles](/assets/img/compose-internals/chapter07/img-001.png)

만약 자식을 여러 번 측정해야 하는 특수한 상황(예: 자식 간 크기 맞춤)이 필요하다면 **Intrinsic Measurement**라는 별도의 메커니즘을 활용합니다.

---

## Layout 단계의 3단계 프로세스

Compose 노드가 배치되는 과정은 다음 세 단계를 순차적으로 거칩니다:

1. **Measuring Children (자식 측정):** 부모는 자식에게 `Constraints`를 전달하며 "이 범위 내에서 크기를 결정해라"고 요청합니다.
2. **Deciding Own Size (자신의 크기 결정):** 자식들이 보고한 크기를 바탕으로 부모 자신의 너비와 높이를 결정합니다.
3. **Placing Children (자식 배치):** 부모의 좌표계 안에서 각 자식이 위치할 (x, y) 좌표를 지정합니다.

![Layout Process](/assets/img/compose-internals/chapter07/img-002.png)

---

## Constraints (제약 조건) 시스템

`Constraints`는 부모가 자식에게 전달하는 최소/최대 크기 정보입니다.

- `minWidth`, `maxWidth`
- `minHeight`, `maxHeight`

자식 노드는 반드시 이 범위 안에서 크기를 결정해야 합니다. 예를 들어 부모가 `minWidth = 100`을 주었다면, 자식은 아무리 작아도 100 이상의 너비를 가져야 합니다.

---

## Custom Layout 구현

`Layout` 컴포저블을 사용하면 직접 배치를 제어할 수 있습니다. `MeasurePolicy` 인터페이스를 통해 자식들을 측정하고 배치하는 로직을 작성합니다.

```kotlin
Layout(
    content = content,
    modifier = modifier
) { measurables, constraints ->
    // 1. 자식들을 주어진 제약조건으로 측정
    val placeables = measurables.map { it.measure(constraints) }

    // 2. 측정된 자식들의 정보를 바탕으로 부모 크기 결정
    layout(width, height) {
        // 3. 자식들을 좌표에 배치
        placeables.forEach { it.placeRelative(x, y) }
    }
}
```

![Custom Layout Example](/assets/img/compose-internals/chapter07/img-005.png)

---

## 요약

7장 Layouts and Measurement의 핵심 포인트는 다음과 같습니다:

1. ✅ **Single Pass:** 노드는 단 한 번만 측정되어 성능을 보장합니다.
2. ✅ **3-Step Process:** 측정(Measure) -> 크기 결정(Size) -> 배치(Place).
3. ✅ **Constraints:** 부모가 자식의 가용 범위를 결정합니다.
4. ✅ **Layout Composable:** 모든 고수준 레이아웃(Box, Column 등)의 근간이 되는 기본 요소입니다.

---

### 다음 장 예고

다음 장에서는 **Advanced Layouts**를 다룹니다:
- ✨ Intrinsic Measurement 활용법
- ✨ 서브컴포지션 레이아웃 (SubcompositionLayout)
- ✨ 복잡한 UI 최적화 전략
