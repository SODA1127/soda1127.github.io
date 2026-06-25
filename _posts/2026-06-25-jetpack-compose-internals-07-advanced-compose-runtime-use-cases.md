---
layout: post
title: "Jetpack Compose Internals - 7장. Advanced Compose Runtime Use Cases"
description: Compose UI 밖에서 Compose 런타임을 활용하는 고급 사용 사례를 다룹니다. 커스텀 Composition, Kotlin/JS DOM 조작, Mosaic 등.
image: 'https://imgur.com/o8Jk4Is.png'
category: 'programming'
date: 2026-06-25 09:00:00
tags:
- kotlin
- android
- jetpack compose
- compose internals
- kotlin multiplatform
- kotlin js
introduction: Compose UI가 아닌 환경에서도 Compose 런타임과 컴파일러를 활용할 수 있습니다. 벡터 이미지, 웹 DOM, CLI 등 다양한 커스텀 트리에 Compose를 적용하는 방법을 다룹니다.
twitter_text: Compose UI 밖에서도 Compose를 사용할 수 있습니다. Kotlin/JS, Mosaic, 커스텀 Composition의 내부 원리를 살펴봅니다.
---

# 7장. Advanced Compose Runtime Use Cases

지금까지 Composable 함수, 컴파일러, 런타임, UI 통합, 상태 스냅샷, 효과 핸들러를 다뤘습니다. 이 장에서는 Compose UI **밖**에서 Compose 런타임을 활용할 수 있는 고급 사용 사례를 살펴봅니다.

핵심 아이디어는 간단합니다. Compose 컴파일러와 런타임은 **특정 UI 프레임워크에 종속되지 않습니다**. 대신 "Composition"이라는 추상화된 트리를 만들고, Applier라는 인터페이스를 통해 실제 노드 조작을 위임합니다. UI 프레임워크가 무엇이든, 이 Applier만 구현하면 됩니다.

---

## VectorPainter를 통한 커스텀 Composition

Compose UI에서 벡터 그래픽을 그릴 때 `VectorPainter`가 내부적으로 어떻게 동작하는지 살펴봅니다. VectorPainter는 **자신만의 독립적인 Composition**을 유지합니다.

### 왜 별도 Composition이 필요한가?

`ComposeNode`는 전달된 Applier와 일치하는 노드 타입을 요구합니다. Android UI 컨텍스트에서 사용하는 Applier는 벡터 노드와는 호환되지 않습니다. 따라서 벡터 그래픽은 별도의 Composition 트리를 가지게 됩니다.

```kotlin
class RenderVector(private val vector: ImageVector) : Painter() {
    private var composition: Composition? = null

    override fun DrawScope.onDraw() {
        with(vector) {
            draw()
        }
    }

    private fun createComposition() {
        composition = createComposition {
            applier = VectorApplier(this@onDraw)
        }
        composition!!.setContent {
            composable(vector.viewportWidth, vector.viewportHeight) {
                // 벡터 노드 생성
            }
        }
    }
}
```

세 가지 핵심 사항:

1. **Painter는 자신만의 Composition을 유지**합니다. `ComposeNode`가 요구하는 Applier는 UI 컨텍스트의 Applier와 호환되지 않기 때문입니다.

2. **Composition은 초기화되지 않았거나 스코프를 벗어날 때 갱신**됩니다.

3. **`setContent`을 통해 채웁니다.** `RenderVector`가 다른 콘텐츠로 호출될 때마다 `setContent`이 다시 실행되어 벡터 구조가 새로고침됩니다.

이렇게 하면 VectorPainter는 `@Composable` 콘텐츠를 화면에 그릴 수 있게 됩니다. VectorPainter 내부의 Composables도 UI Composition의 상태와 CompositionLocal에 접근할 수 있습니다.

> 핵심: 커스텀 트리를 기존 Composition에 임베딩하는 방법을 알면, Compose의 유연성을 완전히 활용할 수 있습니다.

---

## DOM을 Compose로 관리하기 (Kotlin/JS)

Compose의 멀티플랫폼 지원은 아직 초기 단계입니다. 런타임과 컴파일러만 JVM 외부에서 사용 가능합니다. 하지만 이 두 모듈만으로도 Composition을 만들고 실행할 수 있습니다.

### HTML 트리 → DOM 표현

브라우저는 이미 HTML/CSS 기반의 "뷰" 시스템을 가지고 있습니다. 이 요소들은 JS를 통해 DOM API로 조작할 수 있으며, Kotlin/JS 표준 라이브러리에서도 제공됩니다.

![HTML DOM 트리 표현](/assets/img/compose-internals/chapter07/img-001.png)
*브라우저에서 HTML의 트리 표현*

```html
<div>
  <ul>
    <li>Item 1</li>
    <li>Item 2</li>
    <li>Item 3</li>
  </ul>
</div>
```

Kotlin/JS에서 DOM은 `org.w3c.dom.Node`로 표현됩니다. 관련 타입은 두 가지:

- **HTML 요소** (`org.w3c.dom.HTMLElement`의 하위 클래스): `document.createElement(<tagName>)`으로 생성
- **텍스트 노드** (`org.w3c.dom.Text`): `document.createTextElement(<value>)`으로 생성

### Kotlin/JS에서의 DOM 트리 표현

![Kotlin/JS HTML 트리 표현](/assets/img/compose-internals/chapter07/img-005.png)
*Kotlin/JS에서 HTML 트리의 표현*

이제 이 DOM 요소를 기반으로 Compose 관리 트리, 즉 VectorApplier가 벡터 이미지 구성에 사용하는 것과 유사한 방식을 적용할 수 있습니다.

```kotlin
@Composable
fun Tag(tag: String, content: @Composable () -> Unit) {
    ComposeNode<HTMLElement, DomApplier>(
        factory = { document.createElement(tag) as HTMLElement },
        update = {},
        content = content
    )
}

@Composable
fun Text(value: String) {
    ReusableComposeNode<Text, DomApplier>(
        factory = { document.createTextElement("") },
        update = {
            set(value) { this.data = it }
        }
    )
}
```

여기서 중요한 두 가지 패턴이 있습니다:

- **Tag**는 `ComposeNode`를 사용합니다. `<audio>`와 `<div>`는 완전히 다른 브라우저 표현을 가지므로, 태그명이 변경되면 노드를 재생성해야 합니다. 서로 다른 컴파일타임 그룹을 만들면 Compose가 속성만 업데이트하는 것이 아니라 완전히 교체하도록 힌트를 줍니다.

- **Text**는 `ReusableComposeNode`를 사용합니다. 구조적으로 동일한 노드이므로, Compose가 다른 그룹 내에서도 인스턴스를 재사용합니다. 텍스트 노드는 콘텐츠 없이 생성되고, `update` 파라미터로 값을 설정합니다.

DomApplier는 DOM 노드의 add/remove 자식 메서드를 사용하는 점을 제외하면 VectorApplier와 매우 유사합니다. 대부분의 코드는 기계적이며 (노드를 올바른 인덱스로 이동), Compose for Web의 Applier를 참고하세요.

---

## 브라우저에서 독립적인 Composition

Compose UI에서는 `ComposeView`가 모든 초기화를 처리하지만, 브라우저 환경에서는 처음부터 만들어야 합니다.

```kotlin
fun renderComposable(root: HTMLElement, content: @Composable () -> Unit) {
    GlobalSnapshotManager.ensureStarted()

    val recomposerContext = DefaultMonotonicFrameClock + Dispatchers.Main
    val recomposer = Recomposer(recomposerContext)

    val composition = ControlledComposition(
        applier = DomApplier(root),
        parent = recomposer
    )

    composition.setContent(content)

    CoroutineScope(recomposerContext).launch(start = UNDISPATCHED) {
        recomposer.runRecomposeAndApplyChanges()
    }
}
```

`renderComposable`은 Composition 시작의 모든 구현 세부사항을 숨기고, DOM 요소에 Composable 요소를 렌더링할 수 있는 방법을 제공합니다. 주요 설정 단계:

1. **스냅샷 시스템 초기화**: 상태 업데이트를 담당합니다. `GlobalSnapshotManager`는 의도적으로 런타임에 포함되어 있지 않으며, Android 소스에서 복사할 수 있습니다. 이것이 현재 유일한 플랫폼 미제공 부분입니다.

2. **Recomposer용 코루틴 컨텍스트 생성**: JS 기본 `MonotonicClock`은 `requestAnimationFrame`으로 제어되며, `Dispatchers.Main`은 JS가 작동하는 유일한 스레드를 참조합니다.

3. **Composition 생성**: Android 예시와 동일하게 생성하되, `recomposer`가 최상위 Composition의 부모가 됩니다 (recomposer는 항상 최상위 Composition의 부모여야 합니다).

4. **Composition 콘텐츠 설정**: 모든 업데이트는 제공된 Composable 내부에서 발생해야 합니다. 새 `renderComposable` 호출은 모든 것을 새로 생성합니다.

5. **Recomposition 프로세스 시작**: `Recomposer.runRecomposeAndApplyChanges()`로 코루틴을 시작합니다. Android에서는 보통 activity/view 라이프사이클에 연결되며, `recomposer.cancel()`로 중단합니다. 브라우저에서는 Composition 라이프사이클이 페이지 수명에 연결되므로 취소가 필요 없습니다.

### 실제 사용

```kotlin
fun main() {
    renderComposable(document.body!!) {
        var counterState by remember { mutableStateOf(0) }

        Tag("h1") {
            Text("Counter value: $counterState")
        }

        Tag("button", onClick = { counterState++ }) {
            Text("Increment!")
        }
    }
}
```

각 태그는 클릭 리스너를 lambda 파라미터로 정의할 수 있으며, 이는 DOM 노드의 `onclick` 속성으로 전달됩니다.

---

## 요약

이 장에서 다룬 핵심 개념:

- **VectorPainter 예시**: ComposeNode의 Applier 제약 — UI 컨텍스트의 Applier와 벡터 노드는 호환되지 않으므로 별도 Composition이 필요
- **Kotlin/JS DOM 관리**: Compose 컴파일러와 런타임은 플랫폼에 종속되지 않으며, Applier만 구현하면 어떤 트리든 관리 가능
- **독립적 Composition**: `GlobalSnapshotManager`, `Recomposer`, `ControlledComposition`을 직접 초기화하면 Android 외부에서도 Compose를 실행할 수 있음
- **Mosaic**: Jake Wharton의 CLI용 Compose 구현 — 같은 원리로 터미널 UI에도 적용 가능

Compose의 핵심은 "어떤 UI 프레임워크"가 아니라 "Composition"입니다. UI 프레임워크는 Applier를 구현하는 것만으로 Compose의 혜택을 받을 수 있습니다. 이는 Compose UI, Kotlin/JS 웹, Mosaic(CLI), 벡터 그래픽 등 다양한 영역에 적용 가능합니다.

JetBrains 팀은 이미 Compose for Web을 실험 중이며, 더 발전된 버전이 개발되고 있습니다. Kotlin/JS 프로젝트에서 직접 체험해 보세요.

> Compose 팀의 주요 목표는 여전히 Compose UI이지만, Compose가 다른什么地方에서 사용되는지에 대해 매우 관심이 많습니다. #compose Kotlin 슬랙 채널에서 피드백을 제공하세요!

---

## 이전 글

- [6장. Effects and Effect Handlers](/jetpack-compose-internals-06-effects-and-effect-handlers/)
- [5장. State Snapshot System](/jetpack-compose-internals-05-state-snapshot-system/)
- [4장. Compose UI](/jetpack-compose-internals-04-compose-ui/)
- [3장. Compose Runtime](/jetpack-compose-internals-03-compose-runtime-03/)
- [2장. Compose Compiler](/jetpack-compose-internals-02-compose-compiler/)
- [1장. Composable 함수](/jetpack-compose-internals-01-composable-functions/)
