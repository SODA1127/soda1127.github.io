---
layout: post
title: "Jetpack Compose Internals - 6장. Effects and Effect Handlers"
description: Compose에서 사이드 이펙트를 안전하게 제어하고 라이프사이클에 맞춰 실행하는 방법을 정리합니다.
image: 'https://imgur.com/o8Jk4Is.png'
category: 'programming'
date: 2026-02-13 23:00:00
tags:
- kotlin
- android
- jetpack compose
- compose internals
- effects
- side effect
- coroutine
introduction: Effects and Effect Handlers는 Composable의 재실행 특성 속에서도 부수효과를 안전하게 실행하고 종료하는 핵심 메커니즘입니다.
twitter_text: Compose의 SideEffect/LaunchedEffect/DisposableEffect와 Effect Handler 구조를 한 번에 정리합니다.
---

# 6장. Effects and Effect Handlers

Composable은 **재시작 가능(restartable)**하기 때문에, 함수 본문에서 곧바로 부수효과를 실행하면 의도치 않은 반복 실행이나 누수로 이어질 수 있습니다. 이번 장에서는 **Side Effect의 위험성**, **Effect Handler들이 이를 어떻게 제어하는지**, 그리고 **키(key)에 따라 재실행/정리되는 라이프사이클**을 정리합니다.

---

## Side Effect란?

Side effect는 함수의 입력/출력과 무관하게 **외부 상태에 영향을 주는 작업**입니다. 캐시 접근, 네트워크 호출, 파일 IO, UI 업데이트 등이 모두 여기에 포함됩니다. Compose에서 이런 작업을 본문에 직접 넣으면 **리컴포지션마다 다시 실행**되어 예측 불가능해집니다.

![Side Effect Problem](/assets/img/compose-internals/chapter6/img-000.png)

예를 들어 네트워크 요청을 Composable 본문에서 실행하면, 짧은 시간 내 반복 요청이 발생할 수 있습니다. **효과 실행의 시점과 범위를 제어**해야 합니다.

---

## Effect Handler의 목적

Compose는 Side effect를 **Composable 라이프사이클에 맞춰 실행/해제**할 수 있도록 Effect Handler들을 제공합니다. 핵심 목적은 다음과 같습니다:

1. **한 번만 실행**하거나
2. **키가 바뀔 때만 재실행**하거나
3. **Composable이 사라질 때 정리(clean-up)**하기

![Effect Handler Flow](/assets/img/compose-internals/chapter6/img-002.png)

이를 통해 “효과의 범위(scope)”를 명확히 관리합니다.

---

## SideEffect

`SideEffect`는 **매번 성공적인 composition 이후** 실행됩니다. 보통 외부 상태 동기화에 사용합니다.

```kotlin
SideEffect {
    systemUiController.setStatusBarColor(color)
}
```

- 매 recomposition 이후 실행
- 간단한 동기화 작업에 적합

---

## LaunchedEffect

`LaunchedEffect(key)`는 **key가 변할 때만 코루틴을 재시작**합니다. 이전 작업은 자동 취소됩니다.

```kotlin
LaunchedEffect(userId) {
    val profile = repository.load(userId)
}
```

- key가 동일하면 **재실행하지 않음**
- Composable이 사라지면 **자동 취소**
- 네트워크 요청, 비동기 작업에 적합

---

## DisposableEffect

`DisposableEffect(key)`는 **등록-해제 패턴**이 필요한 작업에 적합합니다.

```kotlin
DisposableEffect(lifecycleOwner) {
    val observer = ...
    lifecycle.addObserver(observer)
    onDispose { lifecycle.removeObserver(observer) }
}
```

- key가 바뀌면 기존 onDispose 실행 후 재등록
- 리스너 등록/해제, 브로드캐스트 수신 등에서 활용

---

## rememberCoroutineScope

Composable 내부에서 **UI 이벤트에 반응하는 코루틴**을 실행할 때 사용합니다.

```kotlin
val scope = rememberCoroutineScope()
Button(onClick = { scope.launch { ... } })
```

- Composable이 사라지면 scope도 함께 취소
- 사용자 이벤트 처리에 적합

---

## 요약

Effects and Effect Handlers의 핵심은 **부수효과를 라이프사이클에 맞게 제어**하는 것입니다.

1. ✅ SideEffect: 매 composition 이후 동기화
2. ✅ LaunchedEffect: key 기반 재실행/자동 취소
3. ✅ DisposableEffect: 등록/해제 관리
4. ✅ rememberCoroutineScope: 이벤트 기반 코루틴

Composable의 순수성을 지키면서도 **안전하게 상태와 외부 세계를 연결**하는 것이 핵심입니다.

---

### 다음 장 예고

다음 장에서는 **Advanced Compose Runtime use cases**를 다룹니다:
- ✨ 런타임 고급 API 활용
- ✨ 커스텀 런타임 통합
- ✨ 성능 최적화 사례
