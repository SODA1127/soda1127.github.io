---
layout: post
title: "Jetpack Compose Internals - 4장. Compose UI"
description: Compose UI가 어떻게 런타임과 통합되어 화면의 UI 트리를 구축하는지 살펴봅니다.
image: 'https://imgur.com/o8Jk4Is.png'
category: 'programming'
date: 2026-02-05 23:00:00
tags:
- kotlin
- android
- jetpack compose
- compose internals
- compose ui
- applier
- layout node
introduction: Compose UI는 런타임의 클라이언트 라이브러리로서, Composable 함수의 변경 사항을 실제 UI 트리에 반영하는 역할을 합니다. 이 장에서는 Compose UI가 어떻게 런타임과 통합되는지 알아봅니다.
twitter_text: Compose UI가 어떻게 런타임과 통합되어 화면의 UI 트리를 구축하는지 살펴봅니다.
---

# 4장. Compose UI

지금까지 우리는 Compose 컴파일러와 런타임을 깊이 있게 살펴봤습니다. 하지만 이들만으로는 화면에 UI를 표시할 수 없습니다. **Compose UI는 런타임의 클라이언트 라이브러리로서, Composable 함수의 실행 결과를 실제 UI 트리로 변환하는 역할**을 합니다.

---

## Compose UI란 무엇인가?

Jetpack Compose라고 말할 때 보통 **컴파일러 + 런타임 + Compose UI**를 모두 포함합니다. 하지만 Compose UI는 선택사항이 아닙니다. 런타임만으로는 UI를 렌더링할 수 없기 때문입니다.

### Compose UI는 멀티플랫폼 라이브러리

Compose UI는 Kotlin 멀티플랫폼 프로젝트로 구성되어 있습니다:

- **Common sourceset**: 모든 플랫폼이 공유하는 코드
- **Android sourceset**: Google이 관리하며 Android 통합 레이어 제공
- **Desktop sourceset**: JetBrains가 관리하며 Desktop 통합 레이어 제공

흥미롭게도, 다른 클라이언트 라이브러리도 존재합니다:
- **Compose for Web**: JetBrains에서 관리하며 DOM 기반
- **Mosaic**: Jake Wharton의 커맨드라인 UI 라이브러리

이들은 모두 Compose 런타임을 기반으로 하지만, **각기 다른 노드 타입과 Applier**를 사용합니다.

---

## Compose UI와 런타임의 통합

Compose UI의 핵심 역할은 **Composable 함수의 변경 사항을 실제 UI 트리에 반영**하는 것입니다.

### Composition과 Layout Tree

Composable 함수가 실행될 때 생성되는 변경 사항들은 **Composition**이라는 데이터 구조에 기록됩니다. 이 Composition은 다음을 추적합니다:

1. **Scheduled Changes**: Composable 함수가 요청한 변경 사항들
2. **Node Tree Mapping**: 이 변경 사항들을 실제 UI 트리에 어떻게 적용할지 매핑

초기 composition 단계에서는 모든 노드가 트리에 추가되고, 이후 recomposition에서는 트리가 업데이트됩니다.

### 여러 개의 Composition?

놀랍게도, **하나의 앱에 여러 Composition이 존재**할 수 있습니다. 예를 들어:

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {  // Composition 1
            MaterialTheme {
                Text("Hello Compose!")
            }
        }
    }
}
```

`setContent`를 호출할 때마다 새로운 Composition이 생성됩니다. 각 Composition은 독립적인 Layout Tree를 관리합니다.

또한 Dialog, PopupWindow, 또는 다른 UI 계층이 필요할 때도 새로운 Composition을 생성할 수 있습니다. 이를 **Subcomposition**이라고 부릅니다.

---

## 변경 사항을 UI 트리에 반영하기 (Applier)

런타임이 Composable 함수를 실행하고 변경 사항을 기록하면, **Applier**가 이 변경 사항들을 실제 UI 노드에 적용합니다.

### Applier의 역할

Applier는 런타임에게 **다음 작업들을 수행**합니다:

1. **노드 삽입 (Insert)**: 새로운 UI 컴포넌트를 트리에 추가
2. **노드 제거 (Remove)**: 더 이상 필요 없는 UI 컴포넌트 제거
3. **노드 이동 (Move)**: UI 컴포넌트의 위치 변경
4. **노드 교체 (Replace)**: 기존 노드를 새로운 노드로 교체

### Compose UI의 Applier: UiApplier

Android의 경우, Compose UI는 **UiApplier**를 제공합니다:

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
    val original = Composition(
        UiApplier(owner.root),  // 여기서 UiApplier 생성
        parent
    )
    val wrapped = WrappedComposition(owner, original)
    wrapped.setContent(content)
    return wrapped
}
```

`UiApplier`는 `owner.root` (LayoutNode)를 가리키며, **Compose UI의 노드 타입을 관리**합니다. 런타임은 Applier를 통해서만 노드 타입을 알 수 있고, 직접 접근하지 않습니다.

### 다양한 Applier 구현

Compose UI는 UI 요소에 따라 다양한 Applier를 지원합니다:

1. **UiApplier**: LayoutNode 기반 (일반적인 UI)
2. **VectorPainter의 Applier**: 벡터 그래픽 렌더링
3. **Desktop Applier**: Desktop 플랫폼용

각 Applier는 특정 노드 타입에 맞게 구현되므로, 런타임은 어떤 플랫폼이든 지원할 수 있습니다.

---

## LayoutNode: UI의 기본 단위

Compose UI의 모든 UI 요소는 **LayoutNode**로 표현됩니다. LayoutNode는 다음을 담당합니다:

1. **레이아웃 정보**: 크기, 위치, 패딩 등
2. **그리기 (Drawing)**: 화면에 렌더링될 내용
3. **이벤트 처리**: 사용자 입력 (터치, 클릭 등)
4. **자식 노드 관리**: 계층 구조 유지

### LayoutNode 생성 과정

Composable 함수가 실행되면 다음과 같이 진행됩니다:

```
Composable 함수 실행
    ↓
변경 사항 기록 (Slot Table)
    ↓
Applier가 변경 사항 순회
    ↓
새로운 LayoutNode 생성 및 트리에 추가
    ↓
레이아웃 계산 및 그리기
```

예를 들어, `Layout` Composable은 다음과 같이 동작합니다:

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
            // ... 더 많은 속성 설정
        }
    )
}
```

---

## 레이아웃 측정 (Measuring)

UI 요소를 화면에 표시하려면 먼저 **크기를 결정**해야 합니다. Compose UI는 매우 효율적인 측정 시스템을 제공합니다.

### Measuring Policy

각 LayoutNode는 `MeasurePolicy`를 가지고 있으며, 이것이 **레이아웃 크기를 결정**합니다:

1. **자식 요소들의 크기 측정**
2. **자신의 크기 결정**
3. **자식 요소들의 위치 배치**

### Intrinsic Measurements

Compose UI는 `intrinsic measurements`도 지원합니다. 이는 다음을 계산합니다:

- **intrinsicWidth**: 콘텐츠가 필요로 하는 최소/최대 너비
- **intrinsicHeight**: 콘텐츠가 필요로 하는 최소/최대 높이

예를 들어, `Text` 컴포넌트는 텍스트의 최소 너비를 계산하여 다른 레이아웃이 올바른 크기를 결정하도록 도웁니다.

### Layout Constraints

레이아웃 제약 조건(Layout Constraints)은 다음을 정의합니다:

```
minWidth, maxWidth
minHeight, maxHeight
```

자식 요소들은 이 제약 조건 내에서 크기를 결정해야 합니다. 예를 들어, Column이 화면 너비 전체를 차지하도록 요청하면, 자식들은 그 너비를 존중해야 합니다.

---

## Recomposition과 UI 업데이트

State가 변경되어 recomposition이 발생하면, 다음과 같이 진행됩니다:

1. **Runtime**: Recomposer가 영향받은 Composition을 찾아 recompose
2. **변경 사항 기록**: 새로운 변경 사항들이 Slot Table에 기록됨
3. **Applier**: 변경 사항을 순회하며 LayoutNode 트리 업데이트
4. **레이아웃 재계산**: 영향받은 노드들만 다시 측정
5. **그리기 (Redraw)**: 화면에 새로운 UI 반영

이 과정은 **스마트 리컴포지션**으로 인해 최적화됩니다. 변경되지 않은 부분은 다시 계산되지 않습니다.

---

## 요약

**Compose UI**는 다음을 담당합니다:

1. ✅ Composable 함수와 실제 UI 요소(LayoutNode) 연결
2. ✅ 변경 사항을 UI 트리에 적용 (Applier)
3. ✅ 레이아웃 측정 및 배치
4. ✅ 화면에 그리기
5. ✅ State 변경에 따른 UI 업데이트

Compose 컴파일러가 코드를 최적화하고, Compose 런타임이 Composable 함수를 실행하며 변경 사항을 추적한다면, **Compose UI는 이 모든 것을 실제 화면의 UI로 변환**합니다. 이것이 Jetpack Compose가 선언적 UI 프레임워크로서의 가치를 발휘하는 지점입니다.

---

### 다음 장 예고

다음 장에서는 **Compose UI의 심화 주제들**을 다룹니다:
- ✨ 다양한 타입의 Applier 구현
- ✨ 커스텀 Layout 만들기
- ✨ 성능 최적화
- ✨ 다른 플랫폼을 위한 클라이언트 라이브러리 작성