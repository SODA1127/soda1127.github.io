---
layout: post
title: "Jetpack Compose Internals - 1장. Composable 함수"
description: Jetpack Compose의 핵심인 Composable 함수의 내부 동작 원리를 깊이 있게 살펴봅니다.
image: 'https://i.imgur.com/klrxZcG.png'
category: 'programming'
date: 2026-02-02 09:00:00
tags:
- kotlin
- android
- jetpack compose
- compose internals
introduction: Jetpack Compose의 핵심 구성요소인 Composable 함수에 대해 깊이 있게 알아봅니다.
twitter_text: Jetpack Compose의 핵심인 Composable 함수의 내부 동작 원리를 살펴봅니다.
---

# 1장. Composable 함수

Jetpack Compose 내부 동작을 이해하기 위해 가장 먼저 알아야 할 것은 바로 **Composable 함수**입니다. Composable 함수는 Jetpack Compose의 가장 기본적인 구성 요소이며, Composable 트리를 작성하는 데 사용하는 핵심 구조입니다.

여기서 "트리"라고 표현한 이유가 있습니다. Composable 함수는 Compose 런타임이 메모리에 표현할 더 큰 트리의 노드로 이해할 수 있기 때문입니다.

## Composable 함수의 의미

문법적으로 보면, 일반적인 Kotlin 함수에 `@Composable` 어노테이션을 붙이는 것만으로 Composable 함수가 됩니다:

```kotlin
@Composable
fun NamePlate(name: String) {
    // Composable 코드
}
```

이렇게 어노테이션을 붙이면, 컴파일러에게 "이 함수는 데이터를 Composable 트리에 등록할 노드로 변환하려 한다"라고 알려주는 것입니다.

`@Composable (Input) -> Unit` 형태로 읽으면, Input은 데이터이고 Output은 함수에서 반환되는 값이 아니라 **트리에 요소를 삽입하는 액션**입니다. 이것은 함수 실행의 부수 효과(side effect)로 발생합니다.

> Unit을 반환하는 함수가 입력을 받는다는 것은, 함수 본문 내에서 해당 입력을 어떤 방식으로든 소비하고 있다는 의미입니다.

이러한 액션을 Compose에서는 **"emitting(방출)"**이라고 부릅니다. Composable 함수는 실행될 때 emit하며, 이는 composition 과정에서 발생합니다. "composing"한다는 것은 "실행"한다는 것과 동일하게 생각하면 됩니다.

Composable 함수를 실행하는 유일한 목적은 **트리의 메모리 내 표현을 구축하거나 업데이트**하는 것입니다. 트리를 업데이트하기 위해 새 노드를 삽입하는 액션을 emit할 수 있고, 노드를 제거, 교체, 이동할 수도 있습니다. 또한 트리에서 상태를 읽거나 쓸 수 있습니다.

---

## Composable 함수의 속성

`@Composable` 어노테이션은 함수의 타입을 실질적으로 변경하며, 이에 따른 몇 가지 제약과 속성이 부여됩니다. 이러한 속성들은 Jetpack Compose의 핵심 기능을 가능하게 합니다.

Compose 런타임은 Composable 함수가 이러한 속성을 준수할 것으로 기대하므로, 다음과 같은 런타임 최적화를 수행할 수 있습니다:

- 병렬 composition
- 우선순위 기반의 임의 순서 composition
- 스마트 recomposition
- 위치 기반 메모이제이션 (Positional memoization)

런타임 최적화는 런타임이 실행할 코드에 대해 어느 정도 확신을 가질 수 있을 때만 가능합니다. 예를 들어, 코드의 요소들이 서로 의존적인지, 병렬로 실행할 수 있는지, 각 원자적 로직을 완전히 독립된 단위로 해석할 수 있는지 등을 알 수 있어야 합니다.

---

### 호출 컨텍스트 (Calling Context)

Composable 함수의 대부분의 속성은 Compose 컴파일러에 의해 활성화됩니다. Kotlin 컴파일러 플러그인으로서 일반 컴파일러 단계에서 실행되며, Kotlin 컴파일러가 접근할 수 있는 모든 정보에 접근할 수 있습니다. 이를 통해 소스의 모든 Composable 함수의 IR(중간 표현)을 가로채고 변환하여 추가 정보를 덧붙일 수 있습니다.

각 Composable 함수에 추가되는 것 중 하나는 파라미터 목록 끝에 새 파라미터인 **Composer**입니다. 이 파라미터는 암시적이며, 개발자는 이를 알 필요가 없습니다. 런타임에 인스턴스가 주입되고, 모든 자식 Composable 호출에 전달되어 트리의 모든 레벨에서 접근할 수 있습니다.

```kotlin
// 개발자가 작성한 코드
@Composable
fun NamePlate(name: String, lastname: String) {
    Column(modifier = Modifier.padding(16.dp)) {
        Text(text = name)
        Text(text = lastname, style = MaterialTheme.typography.subtitle1)
    }
}

// 컴파일러가 변환한 코드
fun NamePlate(name: String, lastname: String, $composer: Composer<*>) {
    Column(modifier = Modifier.padding(16.dp), $composer) {
        Text(text = name, $composer)
        Text(text = lastname, style = MaterialTheme.typography.subtitle1, $composer)
    }
}
```

Composer는 본문 내의 모든 Composable 호출에 전달됩니다. 이것이 **Composable 함수는 다른 Composable 함수에서만 호출될 수 있다**는 규칙의 이유입니다. 이를 통해 트리가 오직 Composable 함수로만 구성되고, Composer가 아래로 전달될 수 있습니다.

Composer는 개발자가 작성하는 Composable 코드와 Compose 런타임 사이의 **연결 고리**입니다.

---

### 멱등성 (Idempotent)

Composable 함수는 생성하는 노드 트리에 대해 **멱등적**이어야 합니다. 같은 입력 파라미터로 여러 번 실행해도 동일한 트리가 생성되어야 합니다. Jetpack Compose 런타임은 recomposition과 같은 기능을 위해 이 가정에 의존합니다.

**Recomposition**은 입력이 변경되었을 때 Composable 함수를 다시 실행하여 업데이트된 정보를 emit하고 트리를 갱신하는 과정입니다. 런타임은 임의의 시간에, 다양한 이유로 Composable 함수를 recompose할 수 있어야 합니다.

recomposition 과정은 트리를 순회하며 어떤 노드가 재구성되어야 하는지 확인합니다. 입력이 변경된 노드만 recompose되고, 나머지는 **스킵**됩니다. 노드를 스킵하는 것은 해당 Composable 함수가 멱등적일 때만 가능합니다.

---

### 제어되지 않는 부수 효과 금지

**부수 효과(side effect)**란 함수의 제어를 벗어나 예기치 않은 작업을 수행하는 것입니다:

- 로컬 캐시에서 읽기
- 네트워크 호출
- 전역 변수 설정

이러한 부수 효과는 함수를 외부 요인에 의존하게 만들어 예측 불가능(비결정적)하게 만듭니다.

```kotlin
// ❌ 나쁜 예시 - 네트워크 요청을 직접 호출
@Composable
fun EventsFeed(networkService: EventsNetworkService) {
    val events = networkService.loadAllEvents() // 위험!
    
    LazyColumn {
        items(events) { event ->
            Text(text = event.name)
        }
    }
}
```

이 코드는 매우 위험합니다. Compose 런타임이 짧은 시간 내에 이 함수를 여러 번 재실행할 수 있어, 네트워크 요청이 통제 불능으로 트리거될 수 있습니다. 더 나쁜 것은, 이러한 실행이 조정 없이 다른 스레드에서 발생할 수 있다는 것입니다.

또한 다음과 같이 실행 순서에 의존하는 코드도 피해야 합니다:

```kotlin
@Composable
fun MainScreen() {
    Header()         // 이 세 함수는 어떤 순서로든,
    ProfileDetail()  // 심지어 병렬로 실행될 수 있습니다.
    EventList()      // 순서에 의존하면 안 됩니다!
}
```

Composable 함수는 **상태가 없어야(stateless)** 합니다. 모든 입력을 파라미터로 받고, 그것만을 사용하여 결과를 생성해야 합니다. 그러나 상태 있는 프로그램을 작성하려면 부수 효과가 필요합니다. 이를 위해 Jetpack Compose는 **Effect Handlers**를 제공합니다.

Effect handler는 부수 효과가 Composable 라이프사이클을 인식하도록 하여:
- Composable이 트리를 떠날 때 자동으로 dispose/cancel
- Effect 입력이 변경되면 다시 트리거
- 여러 실행(recomposition)에 걸쳐 한 번만 호출

---

### 재시작 가능 (Restartable)

Composable 함수는 **recompose**할 수 있으므로, 호출 스택의 일부로 한 번만 호출되는 일반 함수와 다릅니다.

일반 호출 스택에서 각 함수는 한 번 호출되고, 하나 이상의 다른 함수를 호출할 수 있습니다. 반면 Composable 함수는 여러 번 재시작(재실행, recompose)될 수 있으므로, 런타임이 이를 수행하기 위해 참조를 유지합니다.

Compose는 메모리 내 표현을 항상 최신 상태로 유지하기 위해 트리의 어떤 노드를 재시작할지 선택적으로 결정합니다. Compose 컴파일러는 상태를 읽는 모든 Composable 함수를 찾아 런타임이 그것들을 재시작하는 방법을 알려주는 코드를 생성합니다.

---

### 빠른 실행 (Fast Execution)

Composable 함수와 Composable 함수 트리는 메모리에 유지되고 나중 단계에서 해석/구체화될 **프로그램의 설명을 빠르고 선언적이며 가볍게 구축하는 방법**입니다.

Composable 함수는 UI를 빌드하고 반환하지 않습니다. 단순히 메모리 내 구조를 빌드하거나 업데이트하기 위한 데이터를 emit합니다. 이로 인해 **매우 빠르며**, 런타임이 두려움 없이 여러 번 실행할 수 있습니다. 애니메이션의 모든 프레임마다 발생하기도 합니다.

개발자는 코드 작성 시 이 기대를 충족해야 합니다. 비용이 많이 드는 계산은 코루틴으로 오프로드하고, 항상 라이프사이클 인식 effect handler로 래핑해야 합니다.

---

### 위치 기반 메모이제이션 (Positional Memoization)

**위치 기반 메모이제이션**은 함수 메모이제이션의 한 형태입니다. 함수 메모이제이션은 함수가 입력을 기반으로 결과를 캐시하여, 같은 입력으로 함수가 호출될 때마다 다시 계산할 필요가 없게 하는 기능입니다. 이는 순수(결정적) 함수에서만 가능합니다.

Compose에서는 추가 요소가 고려됩니다: Composable 함수는 **소스에서의 위치에 대한 상수 지식**을 가집니다. 런타임은 같은 함수가 같은 파라미터 값으로 호출되더라도 다른 위치에서 호출되면 다른 id를 생성합니다:

```kotlin
@Composable
fun MyComposable() {
    Text("Hello") // id 1
    Text("Hello") // id 2  
    Text("Hello") // id 3
}
```

메모리 내 트리는 각각 다른 identity를 가진 세 개의 다른 인스턴스를 저장합니다.

#### 리스트에서의 Identity 문제

때때로 고유한 identity를 할당하기 어려울 수 있습니다. 루프에서 생성된 Composable 리스트가 그 예입니다:

```kotlin
@Composable
fun TalksScreen(talks: List<Talk>) {
    Column {
        for (talk in talks) {
            Talk(talk)  // 매번 같은 위치에서 호출됨
        }
    }
}
```

리스트 끝에 새 요소를 추가할 때는 잘 작동하지만, 맨 위나 중간에 추가하면 어떻게 될까요? 런타임은 위치가 이동한 모든 Talk를 recompose할 것입니다. 이는 매우 비효율적입니다.

이를 해결하기 위해 Compose는 `key` Composable을 제공합니다:

```kotlin
@Composable
fun TalksScreen(talks: List<Talk>) {
    Column {
        for (talk in talks) {
            key(talk.id) {  // 고유 키
                Talk(talk)
            }
        }
    }
}
```

#### remember 함수

때로는 Composable 함수 범위보다 더 세밀한 방식으로 메모리 내 구조에 접근해야 합니다. `remember` 함수가 이를 위해 제공됩니다:

```kotlin
@Composable
fun FilteredImage(path: String) {
    val filters = remember { computeFilters(path) }
    ImageWithFiltersApplied(filters)
}
```

`remember`는 호출 위치와 함수 입력(이 경우 파일 경로)을 기반으로 캐시된 값을 인덱싱합니다.

> Compose에서 메모이제이션은 애플리케이션 전체가 아닙니다. 메모이제이션된 것은 그것을 호출하는 Composable의 컨텍스트 내에서 수행됩니다.

---

## suspend 함수와의 유사점

Kotlin `suspend` 함수도 다른 suspend 함수에서만 호출될 수 있으므로, 호출 컨텍스트가 필요합니다. 이를 통해 suspend 함수가 함께 연결되고, Kotlin 컴파일러가 모든 계산 레벨에 걸쳐 런타임 환경을 주입하고 전달할 수 있습니다.

이 런타임은 각 suspend 함수에 파라미터 목록 끝에 추가 파라미터로 추가됩니다: **Continuation**. 이 파라미터도 암시적이어서 개발자가 알 필요가 없습니다.

```kotlin
// 원본 코드
suspend fun publishTweet(tweet: Tweet): Post = ...

// Kotlin 컴파일러가 변환한 코드
fun publishTweet(tweet: Tweet, callback: Continuation<Post>): Unit
```

Continuation은 Kotlin 런타임이 프로그램의 다양한 suspension point에서 실행을 일시 중단하고 재개하는 데 필요한 모든 정보를 담고 있습니다.

`@Composable`도 언어 기능으로 이해할 수 있습니다. 표준 Kotlin 함수를 **재시작 가능하고, 반응적**으로 만듭니다.

---

## Composable 함수의 색깔 (The Color)

Composable 함수는 표준 함수와 다른 제한과 기능을 가집니다. 다른 타입(나중에 자세히 설명)을 가지며, 매우 특정한 관심사를 모델링합니다. 이 차별화는 **"함수 색칠(function coloring)"**의 한 형태로 이해할 수 있습니다.

"함수 색칠"은 Google Dart 팀의 Bob Nystrom이 2015년 "What color is your function?"이라는 블로그 포스트에서 설명한 개념입니다. async와 sync 함수가 함께 잘 구성되지 않는다고 설명했습니다.

Kotlin에서 `suspend`도 색칠되어 있습니다. suspend 함수는 다른 suspend 함수에서만 호출할 수 있기 때문입니다. 표준 함수와 suspend 함수의 혼합으로 프로그램을 구성하려면 별도의 통합 메커니즘(코루틴 launch point)이 필요합니다.

Jetpack Compose에서 Composable 함수의 경우도 동일합니다. 표준 함수에서 Composable 함수를 투명하게 호출할 수 없습니다. 통합 지점(예: `Composition.setContent`)이 필요합니다.

```kotlin
@Composable
fun SpeakerList(speakers: List<Speaker>) {
    Column {
        speakers.forEach {
            Speaker(it)  // forEach 람다에서 Composable 호출?
        }
    }
}
```

`Speaker` Composable이 `forEach` 람다에서 호출되는데, 컴파일러가 불평하지 않습니다. 어떻게 이런 방식으로 함수 색을 혼합할 수 있을까요?

그 이유는 **inline**입니다. 컬렉션 연산자는 inline으로 선언되어 람다를 호출자에게 인라인합니다. 위 예제에서 `Speaker` Composable 호출은 `SpeakerList` 본문 내에 인라인되며, 둘 다 Composable 함수이므로 허용됩니다.

색칠이 정말 문제일까요? 두 타입의 함수를 결합하고 서로 점프해야 한다면 문제가 될 수 있습니다. 그러나 `suspend`나 `@Composable` 모두 그런 경우가 아닙니다. 두 메커니즘 모두 통합 지점이 필요하며, 그 지점 이후로 완전히 색칠된 호출 스택을 얻습니다.

이는 실제로 **장점**입니다. 컴파일러와 런타임이 색칠된 함수를 다르게 처리하고, 표준 함수에서는 불가능했던 더 고급 언어 기능을 활성화할 수 있기 때문입니다.

---

## Composable 함수 타입

`@Composable` 어노테이션은 컴파일 시 함수의 타입을 효과적으로 변경합니다. 문법적 관점에서 Composable 함수의 타입은 `@Composable (T) -> A`이며, A는 Unit이거나 함수가 값을 반환하는 경우 다른 타입일 수 있습니다(예: `remember`).

개발자는 이 타입을 사용하여 Kotlin에서 일반 람다를 선언하듯이 Composable 람다를 선언할 수 있습니다:

```kotlin
// 모든 Composable 트리에서 재사용 가능
val textComposable: @Composable (String) -> Unit = {
    Text(
        text = it,
        style = MaterialTheme.typography.subtitle1
    )
}

@Composable
fun NamePlate(name: String, lastname: String) {
    Column(modifier = Modifier.padding(16.dp)) {
        Text(text = name, style = MaterialTheme.typography.h6)
        textComposable(lastname)
    }
}
```

Composable 함수는 `@Composable Scope.() -> A` 타입도 가질 수 있으며, 특정 Composable에만 정보를 스코핑하는 데 자주 사용됩니다:

```kotlin
inline fun Box(
    ...,
    content: @Composable BoxScope.() -> Unit
) {
    Layout(
        content = { BoxScopeInstance.content() },
        measurePolicy = measurePolicy,
        modifier = modifier
    )
}
```

언어 관점에서 타입은 컴파일러에 정보를 제공하여 빠른 정적 검증을 수행하고, 때로는 편리한 코드를 생성하며, 런타임에 데이터 사용 방법을 구분/정제하기 위해 존재합니다. `@Composable` 어노테이션은 함수의 검증 방식과 런타임 사용 방식을 변경하며, 이것이 일반 함수와 다른 타입을 가지는 이유입니다.

---

## 마무리

Composable 함수는 Jetpack Compose의 핵심입니다. `@Composable` 어노테이션이 단순해 보이지만, 그 뒤에는 컴파일러 변환, 런타임 최적화, 그리고 선언적 UI의 원칙들이 숨어 있습니다.

핵심 속성들을 다시 정리하면:
- **호출 컨텍스트**: Composer 파라미터가 암시적으로 전달됨
- **멱등성**: 같은 입력 = 같은 결과
- **부수 효과 제어**: Effect Handler 사용
- **재시작 가능**: recomposition 지원
- **빠른 실행**: UI를 반환하지 않고 데이터만 emit
- **위치 기반 메모이제이션**: 소스 위치 + 입력 기반 캐싱
- **함수 색칠**: 별도의 통합 지점 필요

다음 글에서는 **Compose 컴파일러**가 어떻게 동작하는지 자세히 살펴보겠습니다.

> 이 시리즈는 "Jetpack Compose Internals" 책의 내용을 바탕으로 정리한 것입니다.
