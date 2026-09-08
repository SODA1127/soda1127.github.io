---
layout: post
title: "Jetpack Compose Internals - 11장. Effects and Effect Handlers (이펙트와 이펙트 핸들러 심층 분석)"
description: Compose의 라이프사이클에 안전하게 부수효과(Side Effect)를 바인딩하는 Effect Handler의 내부 구조, RememberObserver 생명주기 계약, DisposableEffect/SideEffect/LaunchedEffect/produceState의 구현 메커니즘과 서드파티 스트림 어댑터 설계 패턴을 심층 분석합니다.
image: 'https://imgur.com/o8Jk4Is.png'
category: 'programming'
date: 2026-09-08 09:00:00
tags:
- kotlin
- android
- jetpack compose
- compose internals
- effect handlers
- side effect
- launchedeffect
- disposableeffect
- sideeffect
- coroutine
introduction: Compose 함수는 여러 번 재실행(Recomposition)될 수 있고 병렬 스레드에서 임의의 순서로 실행될 수 있습니다. 제어되지 않은 부수효과는 상태 불일치와 메모리 누수의 온상이 됩니다. Compose가 어떻게 이펙트 핸들러를 통해 부수효과를 결정론적으로 제어하고 수명 주기에 바인딩하는지 내부 소스 레벨에서 분석합니다.
twitter_text: Compose의 Effect Handler 심층 분석: RememberObserver 수명 주기 계약, LaunchedEffect/DisposableEffect/SideEffect 내부 구조 및 RxJava/Flow 어댑터 패턴을 정리합니다.
---

# 11장. Effects and Effect Handlers (이펙트와 이펙트 핸들러 심층 분석)

Jetpack Compose는 선언적이고 반응형인 UI 툴킷입니다. 컴포저블 함수는 상태를 UI 트리(Node Tree)로 변환하는 순수성에 가까운 변환기 역할을 수행해야 합니다. 하지만 실제 애플리케이션은 결코 순수한 연산만으로 존재할 수 없습니다. 화면에 표시할 데이터를 네트워크나 데이터베이스에서 가져와야 하고, 센서나 위치 리스너를 등록해야 하며, 사용자의 상호작용에 반응하여 비동기 작업을 처리하고, 화면 이탈 시 리소스를 정리해야 합니다.

이러한 모든 작업은 함수의 스코프를 벗어나 외부 세계에 영향을 미치는 **부수효과(Side Effect)**에 해당합니다. 

문제는 Compose의 런타임 모델이 명령형 뷰(View) 시스템과 근본적으로 다르다는 점입니다. 컴포저블 함수는 상태 변화에 따라 초당 수십 번 이상 **재구성(Recomposition)**될 수 있고, 스마트 스킵(Smart Skipping)에 의해 건너뛰어질 수 있으며, 심지어 런타임 최적화에 의해 서로 다른 스레드에서 병렬로 실행되거나 임의의 순서로 평가될 수 있습니다. 

만약 컴포저블 함수 본문에서 아무런 제어 장치 없이 일반적인 명령형 코드로 부수효과를 실행한다면 시스템은 걷잡을 수 없는 리소스 누수와 경쟁 상태(Race Condition)에 빠지게 됩니다.

Compose는 이러한 부수효과를 안전하고 예측 가능하게 제어할 수 있도록 **Effect Handler(이펙트 핸들러)** 시스템을 제공합니다. 이번 글에서는 부수효과가 Compose 런타임에서 왜 위험한지부터 시작하여, 런타임의 근간인 `RememberObserver` 계약, 비일시 중단 및 일시 중단 이펙트 핸들러들의 내부 구현, 그리고 서드파티 리액티브 스트림(LiveData, RxJava, Flow)을 Compose State로 감싸는 어댑터 메커니즘까지 소스 레벨에서 심층적으로 살펴봅니다.

---

## 1. 부수효과(Side Effect)의 정의와 Compose에서의 치명적 위험성

### 1.1 순수 함수와 비순수 함수

컴퓨터 과학에서 **순수 함수(Pure Function)**는 다음 두 가지 조건을 만족하는 함수를 의미합니다:
1. 동일한 인자(Input)가 주어지면 항상 동일한 결과(Output)를 반환한다 (결정론적 특성).
2. 함수의 실행 도중 외부 세계의 어떤 상태도 변경하지 않으며, 외부 세계의 가변 상태에 의존하지 않는다 (부수효과 부재).

```kotlin
// 순수 함수 (Pure Function)
fun add(a: Int, b: Int): Int = a + b
```

반면, 함수 본문 내에서 외부 캐시, 전역 변수, 데이터베이스, 네트워크 등 함수의 제어 범위를 벗어나는 요소를 참조하거나 수정하는 순간 그 함수는 **비순수 함수(Impure Function)**가 되며 부수효과를 갖게 됩니다.

```kotlin
// 부수효과가 포함된 비순수 함수
fun add(a: Int, b: Int): Int {
    return calculationsCache.get(a, b) ?: (a + b).also { 
        calculationsCache.store(a, b, it) // Side Effect!
    }
}
```

위의 `add` 함수는 외부 캐시(`calculationsCache`)에 접근합니다. 만약 다른 스레드에서 이 캐시를 조작하거나 데이터베이스 연결이 끊어지면, 동일한 인자 `(1, 2)`를 넘겼음에도 불구하고 전혀 다른 결과를 반환하거나 예외를 던질 수 있습니다. 즉, 함수의 실행 결과를 예측하기 어려워지고 테스트가 불안정(Flaky)해집니다.

### 1.2 Compose 환경에서 제어되지 않은 부수효과가 초래하는 문제

Compose에서 컴포저블 함수는 본질적으로 **재시작 가능(Restartable)**하고 **임의의 순서로 실행 가능**한 특성을 지닙니다:

- **빈번한 재실행(Recomposition):** 관찰 중인 상태가 바뀌면 컴포저블은 짧은 시간 동안에도 수없이 반복 실행될 수 있습니다.
- **취소 및 폐기(Abandonment):** 사용자가 화면을 빠르게 스크롤하거나 상태가 급변하면 진행 중이던 컴포지션이 폐기되고 새로운 컴포지션이 시작될 수 있습니다.
- **다중 스레드 실행:** 런타임은 멀티코어 환경을 활용하여 컴포저블을 병렬로 실행하거나 메인 스레드가 아닌 백그라운드 스레드에서 실행할 수 있는 유연성을 확보하고 있습니다.

이러한 환경에서 컴포저블 본문에 일반 부수효과 코드를 직접 배치하면 어떻게 될까요?

```kotlin
// ❌ 매우 위험한 안티 패턴
@Composable
fun EventsFeed(networkService: EventsNetworkService) {
    // 컴포저블 본문에서 직접 비동기 네트워크 요청을 트리거
    val events = networkService.loadAllEvents() // Side Effect!

    LazyColumn {
        items(events) { event ->
            Text(text = event.name)
        }
    }
}
```

이 코드는 다음과 같은 심각한 재앙을 초래합니다:
1. **네트워크 요청 폭풍:** 사소한 UI 상태 변경으로 `EventsFeed`가 재구성될 때마다 네트워크 요청이 중복 발송됩니다.
2. **동기화 및 경쟁 상태 부재:** 여러 네트워크 요청이 서로 다른 타이밍에 응답하여 오래된 응답이 최신 응답을 덮어쓰는 상태 왜곡이 발생합니다.
3. **취소 불가능:** 사용자가 뒤로 가기를 눌러 컴포저블이 화면에서 사라져도 백그라운드 요청은 취소되지 않고 메모리와 대역폭을 낭비합니다.

비동기 네트워크뿐만 아니라 단순한 외부 객체 속성 수정도 마찬가지입니다:

```kotlin
// ❌ 제어되지 않은 외부 상태 변경
@Composable
fun MyScreen(drawerTouchHandler: TouchHandler) {
    val drawerState = rememberDrawerState(DrawerValue.Closed)

    // 컴포저블 실행 본문에서 직접 외부 객체 수정
    drawerTouchHandler.enabled = drawerState.isOpen // Side Effect!

    // ...
}
```

`drawerTouchHandler`에 상태를 동기화하는 작업 역시 컴포지션 본문에서 직접 수행되면, 컴포지션이 중간에 취소되거나 실패하더라도 이미 외부 객체는 오염되며, 컴포저블이 화면을 떠났을 때 핸들러를 비활성화하거나 참조를 해제할 정리(Clean-up) 기회를 영영 잃게 됩니다.

---

## 2. Compose가 요구하는 이펙트 제어 요구사항 (What We Need)

Compose 런타임의 최적화 가능성(병렬 실행, 우선순위 기반 스케줄링)을 훼손하지 않으면서도 외부 세계와 소통하려면 다음과 같은 5가지 명확한 원칙이 보장되어야 합니다:

1. **정확한 생명주기 타이밍 (Lifecycle-aware Execution):**
   이펙트는 컴포저블이 컴포지션 트리에 성공적으로 안착(Enter)하여 모든 레이아웃 노드가 준비된 직후에만 실행되어야 합니다. 중간에 실패하거나 롤백된 컴포지션에서는 절대 실행되지 않아야 합니다.
2. **적절한 런타임 환경 (Coroutine Dispatcher):**
   일시 중단(Suspended) 이펙트는 컴포지션의 라이프사이클에 바인딩된 코루틴 스코프와 UI 조작에 안전한 디스패처(`AndroidUiDispatcher.Main`) 위에서 스케줄링되어야 합니다.
3. **참조 정리 및 누수 방지 (Clean-up Guarantees):**
   외부 이벤트 리스너, 센서, 브로드캐스트 리시버 등을 등록한 이펙트는 해당 컴포저블이 컴포지션을 떠날 때(Leave) 반드시 등록을 해제(`dispose`)할 수 있어야 합니다.
4. **자동 취소 (Automatic Cancellation):**
   비동기 작업이 아직 완료되지 않았더라도 컴포저블이 화면을 벗어나면 진행 중이던 코루틴 Job이 즉시 취소되어야 합니다.
5. **의존성 변화에 따른 재시작 (Restart on Key Variation):**
   이펙트가 참조하는 매개변수(Key)가 변경되면 기존 작업을 안전하게 취소/정리하고, 새로운 매개변수로 이펙트를 재실행할 수 있어야 합니다.

이러한 모든 요구사항을 완벽히 충족시키기 위해 설계된 시스템이 바로 **Effect Handlers**입니다.

---

## 3. 이펙트 핸들러의 근간: RememberObserver 계약과 슬롯 테이블

이펙트 핸들러들이 어떻게 컴포저블의 생명주기(Enter, Leave, Recompose)를 정밀하게 추적할 수 있는지 이해하려면, Compose 런타임의 핵심 인터페이스인 **`RememberObserver`**를 살펴보아야 합니다.

### 3.1 RememberObserver 인터페이스

```kotlin
package androidx.compose.runtime

interface RememberObserver {
    /**
     * 객체가 컴포지션에 처음 기억(remember)되고, 해당 컴포지션이 성공적으로 커밋되었을 때 호출됩니다.
     */
    fun onRemembered()

    /**
     * 객체가 슬롯 테이블에서 잊혀지거나(컴포지션을 떠남), 해당 스코프가 파괴될 때 호출됩니다.
     */
    fun onForgotten()

    /**
     * 컴포지션 과정에서 remember 되었으나, 컴포지션이 실패하거나 폐기(Abandoned)되었을 때 호출됩니다.
     */
    fun onAbandoned()
}
```

컴포저블 내부에서 `remember { ... }`를 통해 어떤 객체를 생성할 때, 그 객체가 `RememberObserver`를 구현하고 있다면 Compose 런타임(Composer)은 특별한 수명 주기 관리를 적용합니다:

1. **컴포지션 진행 중:** 객체가 생성되어 슬롯 테이블(Slot Table)에 기록됩니다. 이때는 아직 UI 트리에 변경 사항이 적용되지 않은 상태이므로 `onRemembered()`가 불리지 않습니다.
2. **컴포지션 성공 및 적용 (Commit):** 변경 목록(Change list)이 UI 트리에 완전히 적용된 직후, 런타임은 슬롯 테이블에 새로 추가된 모든 `RememberObserver`들의 `onRemembered()`를 순차적으로 호출합니다.
3. **컴포지션 실패/폐기:** 컴포지션 도중 예외가 발생하거나 새로운 상태로 인해 현재 컴포지션이 버려지면 즉각 `onAbandoned()`가 호출되어 할당된 리소스를 정리합니다.
4. **컴포지션 이탈 (Leave):** UI 노드가 트리에서 제거되거나 리컴포지션 결과 해당 슬롯이 덮어쓰여지면 `onForgotten()`이 호출됩니다.

Compose의 거의 모든 주요 이펙트 핸들러(`DisposableEffect`, `LaunchedEffect`, `rememberCoroutineScope`)는 바로 이 `RememberObserver`를 내부 구현체로 감싸서 슬롯 테이블에 저장하는 방식으로 작동합니다.

---

## 4. 비일시 중단 효과 (Non-Suspended Effects)

비일시 중단 이펙트는 코루틴을 통한 비동기 일시 중단(Suspension)이 필요 없는 동기식 부수효과를 제어할 때 사용됩니다.

### 4.1 DisposableEffect: 자원 획득과 해제의 라이프사이클 바인딩

`DisposableEffect`는 리스너 등록, 브로드캐스트 리시버 구독, 콜백 부착 등 **명시적인 정리(Clean-up) 작업**이 수반되는 부수효과를 위해 존재합니다.

```kotlin
@Composable
fun backPressHandler(onBackPressed: () -> Unit, enabled: Boolean = true) {
    val dispatcher = LocalOnBackPressedDispatcherOwner.current?.onBackPressedDispatcher ?: return

    val backCallback = remember {
        object : OnBackPressedCallback(enabled) {
            override fun handleOnBackPressed() {
                onBackPressed()
            }
        }
    }

    DisposableEffect(dispatcher) {
        dispatcher.addCallback(backCallback)

        onDispose {
            // 컴포지션을 떠나거나 dispatcher가 변경되었을 때 콜백을 제거하여 누수 방지
            backCallback.remove()
        }
    }
}
```

#### 동작 특성
- **진입 시 실행:** 컴포저블이 컴포지션에 처음 진입(Enter)할 때 효과 람다가 실행됩니다.
- **반드시 `onDispose` 반환:** 람다의 마지막 표현식으로 반드시 `onDispose { ... }` 블록을 제공해야 합니다.
- **키 변경 시 재실행:** `key` 인자로 넘겨진 값(`dispatcher`)이 이전 컴포지션과 비교하여 변경되었다면(`!currentComposer.changed(key)`가 거짓일 때), **기존의 `onDispose`를 먼저 실행하여 이전 리소스를 정리한 뒤** 새로운 효과 람다를 실행합니다.
- **이탈 시 정리:** 컴포저블이 화면에서 완전히 제거될 때 `onDispose`가 호출되어 리소스 누수를 원천 차단합니다.

#### 내부 구현 파헤치기
Compose 런타임의 `DisposableEffect` 내부 구현을 보면 `RememberObserver`와의 연결을 명확히 볼 수 있습니다:

```kotlin
// androidx.compose.runtime.DisposableEffect.kt (간소화된 런타임 소스)
private class DisposableEffectImpl(
    private val effect: DisposableEffectScope.() -> DisposableEffectResult
) : RememberObserver {
    private var onDispose: DisposableEffectResult? = null

    override fun onRemembered() {
        // 컴포지션 커밋 직후 실행되어 onDispose 결과를 보관
        onDispose = InternalDisposableEffectScope.effect()
    }

    override fun onForgotten() {
        // 컴포지션 이탈 시 정리
        onDispose?.dispose()
        onDispose = null
    }

    override fun onAbandoned() {
        // 컴포지션 실패 시 정리
        onDispose?.dispose()
        onDispose = null
    }
}
```

컴포지션이 일어날 때 `remember(key1, key2) { DisposableEffectImpl(effect) }` 형태로 슬롯 테이블에 저장되기 때문에, 키가 동일하다면 기존 인스턴스가 유지되고, 키가 달라지면 새 인스턴스가 생성되면서 이전 인스턴스는 `onForgotten()`에 의해 정리되는 구조입니다.

단 한 번만 등록하고 컴포저블이 파괴될 때 해제하려면 상수를 키로 전달합니다:
```kotlin
DisposableEffect(Unit) { // 컴포지션 생명주기 동안 1회 실행
    val listener = LocationListener { ... }
    locationManager.register(listener)
    onDispose {
        locationManager.unregister(listener)
    }
}
```

---

### 4.2 SideEffect: 성공적인 커밋 직후의 단발성 외부 상태 동기화

`SideEffect`는 이름이 매우 직관적입니다. "이번 컴포지션이 성공했을 때만 실행하고 잊어버리는(Fire-on-this-composition-or-forget)" 단발성 효과 핸들러입니다.

```kotlin
@Composable
fun MyScreen(drawerTouchHandler: TouchHandler) {
    val drawerState = rememberDrawerState(DrawerValue.Closed)

    SideEffect {
        // 성공적인 컴포지션/리컴포지션 완료 후 외부 객체에 현재 상태 반영
        drawerTouchHandler.enabled = drawerState.isOpen
    }

    // ...
}
```

#### 동작 특성 및 `DisposableEffect`와의 차이점
- **슬롯 테이블에 저장되지 않음:** `SideEffect`는 슬롯 테이블에 객체로 기억(`remember`)되지 않습니다. 따라서 컴포지션을 넘어 살아남지 않으며, 리트라이되거나 이전 상태를 보관하지 않습니다.
- **성공적인 커밋 직후에만 실행:** 컴포지션 패스 도중에는 실행되지 않고, `Composer.recordSideEffect`를 통해 큐에 등록되었다가 **모든 UI 변경 사항이 트리/화면에 완전히 커밋된 직후**에 일괄 실행됩니다.
- **컴포지션 실패 시 자동 폐기:** 컴포지션 도중 예외가 발생하거나 리컴포지션이 도중에 취소되면 등록된 `SideEffect` 람다는 실행되지 않고 그대로 버려집니다. 외부 상태가 오염되는 것을 완벽히 방지합니다.

#### 주요 활용처
Compose가 관리하지 않는 **외부 객체, 분석 도구(Analytics), 시스템 서비스**에 최신 Compose 상태를 동기화할 때 가장 적합합니다:

```kotlin
@Composable
fun TrackScreenView(screenName: String, analytics: AnalyticsTracker) {
    SideEffect {
        // 리컴포지션이 성공하여 실제 화면에 상태가 반영될 때마다 분석 서버에 이벤트 기록
        analytics.setCurrentScreen(screenName)
    }
}
```

---

### 4.3 currentRecomposeScope와 수동 무효화

안드로이드의 고전적인 뷰 시스템에서는 뷰의 크기나 내용이 바뀔 때 `View.invalidate()`나 `requestLayout()`을 호출하여 시스템에 다시 그릴 것을 명령했습니다.

Compose에서는 상태(`State<T>`)의 변경을 감지하여 자동으로 필요한 영역만 재구성하지만, `currentRecomposeScope`를 사용하면 개발자가 수동으로 현재 컴포저블 스코프를 무효화할 수 있습니다:

```kotlin
interface RecomposeScope {
    fun invalidate()
}
```

```kotlin
@Composable
fun LegacyUserView(presenter: LegacyUserPresenter) {
    // Compose State를 사용하지 않는 레거시 프레젠터에서 콜백으로 무효화 요청
    val user = presenter.loadUser { 
        currentRecomposeScope.invalidate() 
    }

    Text("사용자 이름: ${user.name}")
}
```

#### 주의점: 왜 남용하면 안 되는가?
`currentRecomposeScope.invalidate()`는 Compose의 가장 큰 장점인 **세밀한 의존성 추적(Fine-grained State Tracking)**과 **스마트 스킵(Smart Skipping)**을 무력화합니다. 런타임은 어떤 데이터가 어떻게 바뀌었는지 알지 못하므로, 해당 스코프 전체를 무조건 다시 실행해야 합니다. 따라서 레거시 라이브러리와의 어쩔 수 없는 연동을 제외하고는 항상 `State<T>` 기반의 리액티브 모델을 사용하는 것이 권장됩니다.

---

## 5. 일시 중단 효과 (Suspended Effects)와 코루틴의 통합

Compose는 코루틴(Kotlin Coroutines)과 유기적으로 결합되어 비동기 비즈니스 로직, 딜레이, 네트워크 통신 등을 선언적인 방식으로 처리합니다.

### 5.1 rememberCoroutineScope vs LaunchedEffect: 트리거의 주체

안드로이드 개발자들이 가장 많이 헷갈려하는 두 API의 본질적인 차이는 **"누가 작업을 시작하는가(Trigger Actor)"**에 있습니다.

| API | 작업의 시작 주체 (Trigger) | 수명 주기 (Lifecycle) | 주요 사용처 |
| :--- | :--- | :--- | :--- |
| **`LaunchedEffect`** | **컴포지션 자체** (선언적) | 컴포지션 진입 시 시작, 이탈 시 취소 | 화면 진입 시 데이터 로딩, 키 변경 시 재요청 |
| **`rememberCoroutineScope`** | **사용자 인터랙션** (명령형 이벤트) | 컴포지션 이탈 시 모든 진행 중인 Job 일괄 취소 | 버튼 클릭, 제스처, 스낵바 표시 등 이벤트 핸들러 |

---

### 5.2 rememberCoroutineScope: UI 인터랙션을 위한 생명주기 안전 스코프

컴포저블 함수 본문 밖(예: `Button(onClick = { ... })` 콜백)에서는 일시 중단 함수를 직접 호출할 수 없습니다. 이때 필요한 것이 컴포지션의 수명 주기에 바인딩된 `CoroutineScope`입니다.

```kotlin
@Composable
fun SearchScreen(viewModel: SearchViewModel) {
    val scope = rememberCoroutineScope()
    var currentJob by remember { mutableStateOf<Job?>(null) }
    var items by remember { mutableStateOf<List<Item>>(emptyList()) }

    Column {
        TextField(
            value = "",
            onValueChange = { text ->
                // 사용자 인터랙션 이벤트 핸들러 내에서 디바운싱 코루틴 시작
                currentJob?.cancel()
                currentJob = scope.launch {
                    delay(300) // 디바운스
                    items = viewModel.search(query = text)
                }
            }
        )
        ItemsVerticalList(items)
    }
}
```

#### 내부 동작 원리
`rememberCoroutineScope`가 반환하는 스코프는 일반 `CoroutineScope(Dispatchers.Main)`이 아닙니다:

```kotlin
// androidx.compose.runtime.Effects.kt (간소화 소스)
@Composable
fun rememberCoroutineScope(
    getContext: @DisallowComposableCalls () -> CoroutineContext = { EmptyCoroutineContext }
): CoroutineScope {
    val composer = currentComposer
    val wrapper = remember {
        CompositionScopedCoroutineScopeCanceller(
            createCompositionCoroutineScope(getContext(), composer)
        )
    }
    return wrapper.coroutineScope
}

internal class CompositionScopedCoroutineScopeCanceller(
    val coroutineScope: CoroutineScope
) : RememberObserver {
    override fun onRemembered() { /* no-op */ }

    override fun onForgotten() {
        // 컴포저블이 컴포지션을 떠나는 순간 스코프의 모든 자식 Job 취소!
        coroutineScope.cancel()
    }

    override fun onAbandoned() {
        coroutineScope.cancel()
    }
}
```

`CompositionScopedCoroutineScopeCanceller`가 `RememberObserver`를 구현하여 슬롯 테이블에 저장되어 있으므로, 사용자가 해당 화면을 닫고 나가는 순간 진행 중이던 모든 네트워크 및 백그라운드 코루틴이 일괄적으로 취소(Cancellation)됩니다.

---

### 5.3 LaunchedEffect: 컴포지션 트리거 기반의 일시 중단 효과

`LaunchedEffect`는 컴포저블이 화면 트리에 진입할 때 코루틴을 실행하고, 인자로 전달된 `key`가 변경될 때 기존 코루틴을 취소하고 새 코루틴을 시작합니다.

```kotlin
@Composable
fun SpeakerList(eventId: String, viewModel: SpeakerViewModel) {
    var speakers by remember { mutableStateOf<List<Speaker>>(emptyList()) }

    LaunchedEffect(eventId) {
        // eventId가 바뀌면 기존 요청이 자동 취소되고 새로운 이벤트의 스피커 목록을 로드
        speakers = viewModel.loadSpeakers(eventId)
    }

    ItemsVerticalList(speakers)
}
```

#### 키(Key)의 의미와 메커니즘
`LaunchedEffect`는 최소 하나 이상의 키를 필수로 받습니다:
- **`LaunchedEffect(Unit)` 또는 `LaunchedEffect(true)`**: 상수를 전달하면 컴포지션 진입 시 딱 1회만 실행되며, 화면을 떠날 때까지 재실행되지 않고 유지됩니다.
- **`LaunchedEffect(key1, key2)`**: `key1`이나 `key2`의 인스턴스 또는 값이 이전 컴포지션과 비교하여 변경되면(`currentComposer.changed(key)`), **즉각 기존 코루틴 Job을 `cancel()`하고 새로운 코루틴을 런치**합니다.

#### 내부 구현 파헤치기
```kotlin
// androidx.compose.runtime.LaunchedEffectImpl
internal class LaunchedEffectImpl(
    parentCoroutineContext: CoroutineContext,
    private val task: suspend CoroutineScope.() -> Unit
) : RememberObserver {
    private val scope = CoroutineScope(parentCoroutineContext)
    private var job: Job? = null

    override fun onRemembered() {
        // 이전 잡이 있다면 정리하고 새 코루틴 시작
        job?.cancel("Old job was still running!")
        job = scope.launch(block = task)
    }

    override fun onForgotten() {
        job?.cancel()
        job = null
    }

    override fun onAbandoned() {
        job?.cancel()
        job = null
    }
}
```

디스패처는 기본적으로 컴포지션이 구동되는 런타임 디스패처(`AndroidUiDispatcher.Main`)를 사용하므로, 코루틴 블록 내부에서 상태(`speakers`)를 직접 업데이트해도 안전한 스레드 일관성이 유지됩니다.

---

### 5.4 produceState: 비-Compose 비동기 소스를 Compose State로 변환

`produceState`는 내부적으로 `LaunchedEffect`를 활용하여 비동기 데이터 생산 로직을 감싸고, 그 결과를 읽기 전용 `State<T>`로 변환해 주는 고수준 문법 설탕(Syntactic Sugar)입니다.

```kotlin
@Composable
fun SpeakerScreen(eventId: String, viewModel: SpeakerViewModel) {
    // produceState를 사용하여 비동기 호출을 깔끔한 State<List<Speaker>>로 수신
    val uiState by produceState(initialValue = emptyList<Speaker>(), eventId) {
        // ProduceStateScope는 MutableState 인터페이스를 구현하고 있음
        value = viewModel.loadSpeakers(eventId)
    }

    ItemsVerticalList(uiState)
}
```

#### 내부 구현
```kotlin
// androidx.compose.runtime.SnapshotState.kt
@Composable
fun <T> produceState(
    initialValue: T,
    key1: Any?,
    producer: suspend ProduceStateScope<T>.() -> Unit
): State<T> {
    val result = remember { mutableStateOf(initialValue) }
    LaunchedEffect(key1) {
        ProduceStateScopeImpl(result, coroutineContext).producer()
    }
    return result
}
```

`ProduceStateScopeImpl`은 `MutableState<T>`를 래핑하여 `value` 프로퍼티에 새 값을 대입할 수 있게 해주고, 동시에 `CoroutineScope`를 제공합니다. 

> **주의 사항:** `produceState(initialValue)` 호출 시 키를 넘기지 않으면 내부적으로 `LaunchedEffect(Unit)`이 호출되어 컴포지션 생명주기 전체에 걸쳐 한 번만 실행됩니다. 의존하는 상태 변수가 있다면 반드시 키로 전달해야 합니다.

---

## 6. 서드파티 리액티브 라이브러리 어댑터 심층 분석

실무 애플리케이션에서는 순수 Compose State 외에도 안드로이드 아키텍처 컴포넌트인 **LiveData**, 반응형 스트림의 표준인 **RxJava**, 그리고 코틀린 최신 비동기 스트림인 **Flow**를 광범위하게 사용합니다. 

Compose 팀은 이러한 외부 스트림을 Compose 런타임과 원활하게 연결하기 위해 공식 어댑터 라이브러리들을 제공합니다. 이 어댑터들은 놀랍게도 모두 우리가 지금까지 배운 **`DisposableEffect`**와 **`produceState(LaunchedEffect)`**를 기반으로 구현되어 있습니다.

```groovy
// 의존성 구성
implementation "androidx.compose.runtime:runtime:$compose_version"          // Flow 기본 지원
implementation "androidx.compose.runtime:runtime-livedata:$compose_version" // LiveData 어댑터
implementation "androidx.compose.runtime:runtime-rxjava2:$compose_version"  // RxJava2 어댑터
```

---

### 6.1 LiveData: observeAsState 내부 메커니즘

```kotlin
@Composable
fun UserProfile(viewModel: UserViewModel) {
    val user by viewModel.userLiveData.observeAsState(initial = User.EMPTY)
    Text("사용자: ${user.name}")
}
```

`observeAsState`의 실제 런타임 소스 코드를 보면 `DisposableEffect`의 교과서적인 활용을 볼 수 있습니다:

```kotlin
// androidx.compose.runtime.livedata.LiveDataAdapter.kt
@Composable
fun <R, T : R> LiveData<T>.observeAsState(initial: R): State<R> {
    val lifecycleOwner = LocalLifecycleOwner.current
    val state = remember { mutableStateOf(initial) }

    DisposableEffect(this, lifecycleOwner) {
        val observer = Observer<T> { value ->
            state.value = value
        }
        observe(lifecycleOwner, observer)

        onDispose {
            // 화면 이탈 또는 LiveData/LifecycleOwner 변경 시 옵저버 해제!
            removeObserver(observer)
        }
    }
    return state
}
```

`LocalLifecycleOwner.current`를 컴포지션 로컬에서 가져와 LiveData의 옵저버로 등록하고, 컴포저블이 화면을 벗어나면 `DisposableEffect`의 `onDispose`가 정확하게 `removeObserver`를 호출하여 완벽하게 메모리 누수를 방지합니다.

---

### 6.2 RxJava2 / RxJava3: subscribeAsState 내부 메커니즘

```kotlin
@Composable
fun NewsFeed(viewModel: NewsViewModel) {
    val newsState by viewModel.newsObservable.subscribeAsState(initial = ViewState.Loading)

    when (val state = newsState) {
        is ViewState.Loading -> LoadingSpinner()
        is ViewState.Content -> NewsList(state.items)
        is ViewState.Error -> ErrorDialog(state.message)
    }
}
```

RxJava의 `Observable`이나 `Flowable` 역시 구독 후 `Disposable` 토큰을 받아 해제해야 하는 전형적인 자원 관리 패턴을 따릅니다:

```kotlin
// androidx.compose.runtime.rxjava2.RxJava2Adapter.kt
@Composable
fun <R, T : R> Observable<T>.subscribeAsState(initial: R): State<R> {
    val state = remember { mutableStateOf(initial) }

    DisposableEffect(this) {
        val disposable = subscribe(
            { value -> state.value = value },
            { error -> /* 예외 처리 로직 */ }
        )

        onDispose {
            // 컴포지션 이탈 시 구독 취소!
            disposable.dispose()
        }
    }
    return state
}
```

`DisposableEffect(this)`를 사용함으로써 `Observable` 인스턴스가 바뀌면 기존 스트림 구독을 즉시 해제(`dispose`)하고 새 스트림을 구독합니다.

---

### 6.3 Kotlin Coroutines Flow: collectAsState vs collectAsStateWithLifecycle

Flow는 비동기적으로 값을 방출하며, 이를 수집(collect)하려면 반드시 일시 중단(Suspended) 컨텍스트가 필요합니다. 따라서 동기식 `DisposableEffect`가 아닌 **`produceState`**를 기반으로 구축됩니다.

```kotlin
// androidx.compose.runtime.SnapshotState.kt (간소화)
@Composable
fun <T : R, R> Flow<T>.collectAsState(
    initial: R,
    context: CoroutineContext = EmptyCoroutineContext
): State<R> = produceState(initial, this, context) {
    if (context == EmptyCoroutineContext) {
        collect { value = it }
    } else {
        withContext(context) {
            collect { value = it }
        }
    }
}
```

#### 실무 핵심 비교: collectAsState vs collectAsStateWithLifecycle
안드로이드 개발 실무에서 `collectAsState`를 뷰모델의 UI 상태 스트림에 그대로 사용하면 치명적인 문제가 발생할 수 있습니다:
- **`collectAsState`**: 컴포저블이 화면 트리에 존재하는 한, 앱이 백그라운드로 내려가도 Flow 수집을 멈추지 않습니다. 화면이 보이지 않는 상태에서도 불필요한 네트워크 통신이나 DB 쿼리가 지속될 수 있습니다.
- **`collectAsStateWithLifecycle` (`androidx.lifecycle.compose` 패키지)**:
  `Lifecycle.repeatOnLifecycle`과 `produceState`를 결합하여 구현되어 있습니다. 앱이 백그라운드로 내려가 라이프사이클이 `STARTED` 밑으로 떨어지면 **Flow 수집 코루틴을 자동으로 취소**하고, 포그라운드로 복귀하면 다시 수집을 시작하여 기기 배터리와 리소스를 극대화하여 절약합니다.

---

### 6.4 나만의 커스텀 스트림/센서 어댑터 만들기

Compose가 제공하는 이펙트 핸들러 패턴을 이해하면, 실무에서 어떤 형태의 서드파티 이벤트 소스나 하드웨어 센서도 단 몇 줄의 코드로 완벽한 `State<T>` 어댑터로 감쌀 수 있습니다.

```kotlin
// 기기 가속도 센서를 Compose State로 변환하는 커스텀 어댑터
@Composable
fun rememberAccelerometerSensor(sensorManager: SensorManager): State<FloatArray> {
    val sensorData = remember { mutableStateOf(floatArrayOf(0f, 0f, 0f)) }

    DisposableEffect(sensorManager) {
        val sensor = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER)
        val listener = object : SensorEventListener {
            override fun onSensorChanged(event: SensorEvent?) {
                event?.values?.let { sensorData.value = it.clone() }
            }
            override fun onAccuracyChanged(sensor: Sensor?, accuracy: Int) {}
        }

        sensorManager.registerListener(listener, sensor, SensorManager.SENSOR_DELAY_UI)

        onDispose {
            sensorManager.unregisterListener(listener)
        }
    }

    return sensorData
}
```

이처럼 외부 세계의 리스너 기반 비-반응형 API도 `DisposableEffect`와 `remember { mutableStateOf }`만 조합하면 라이프사이클 누수가 전혀 없는 우아한 반응형 Compose 컴포넌트로 탈바꿈합니다.

---

## 7. 이펙트 핸들러의 내부 실행 파이프라인과 프레임 타이밍

마지막으로, Compose의 전체 렌더링 파이프라인 안에서 이펙트 핸들러들이 정확히 어느 시점에 스케줄링되고 실행되는지 그 실행 타이밍을 정리해 봅니다.

```
[ 상태 변경 감지 ]
       │
       ▼
┌─────────────────────────────────────────────────────────┐
│ 1. Composition 단계 (Composer)                           │
│   - Composable 함수들 실행                              │
│   - 슬롯 테이블 읽기/쓰기                               │
│   - 트리 변경 목록(Change List) 생성                   │
│   - DisposableEffectImpl, LaunchedEffectImpl 인스턴스화 │
│   - SideEffect 람다들을 Composer 큐에 등록              │
└─────────────────────────────────────────────────────────┘
       │ (컴포지션 성공)
       ▼
┌─────────────────────────────────────────────────────────┐
│ 2. Layout & Applier 단계                                │
│   - UI 노드 트리(LayoutNode) 생성, 이동, 속성 반영      │
│   - 측정(Measure) 및 배치(Place) 수행                   │
└─────────────────────────────────────────────────────────┘
       │ (UI 변경 커밋 완료)
       ▼
┌─────────────────────────────────────────────────────────┐
│ 3. Commit 후 알림 및 RememberObserver 발화             │
│   - 새 노드의 RememberObserver.onRemembered() 호출     │
│     * LaunchedEffect: 코루틴 launch                    │
│     * DisposableEffect: effect 람다 실행               │
│   - 제거된 노드의 RememberObserver.onForgotten() 호출   │
│     * LaunchedEffect: 코루틴 cancel                    │
│     * DisposableEffect: onDispose() 실행               │
└─────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────┐
│ 4. SideEffect 큐 실행                                   │
│   - Composer에 쌓여 있던 모든 SideEffect 람다 순차 실행 │
└─────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────┐
│ 5. Draw 단계                                            │
│   - 캔버스에 실제 픽셀 렌더링                           │
└─────────────────────────────────────────────────────────┘
```

이 파이프라인을 통해 우리는 다음 두 가지 핵심 사실을 알 수 있습니다:
1. **컴포지션 도중에 실패하거나 취소되면:** `RememberObserver.onAbandoned()`가 호출되며, `SideEffect` 큐는 완전히 폐기됩니다. 따라서 불완전한 컴포지션으로 인해 부수효과가 오작동하는 일이 시스템 레벨에서 원천 차단됩니다.
2. **이펙트 실행 시점은 항상 UI 트리가 안전하게 확정된 직후:** 노드가 생성되고 부모 트리에 정상 부착된 후에야 `onRemembered()`와 `SideEffect`가 동작하므로, 외부 객체나 리스너가 유효하지 않은 UI 상태를 참조할 위험이 없습니다.

---

## 요약

| 이펙트 핸들러 | 주요 목적 | 실행 및 재실행 시점 | 정리/취소 메커니즘 | 슬롯 테이블 보관 |
| :--- | :--- | :--- | :--- | :--- |
| **`DisposableEffect`** | 리스너 등록, 자원 획득 등 동기식 클린업 제어 | 컴포지션 진입 시 1회 실행, `key` 변경 시 재실행 | `onDispose { ... }` 블록 호출 (이탈 또는 키 변경 시) | O (`RememberObserver`) |
| **`SideEffect`** | Compose 외부 상태와 현재 상태 동기화 (단발성) | 매 성공적인 컴포지션/리컴포지션 커밋 직후 | 별도 정리 없음 (컴포지션 실패 시 자동 폐기) | X (큐에서 단발성 소비) |
| **`rememberCoroutineScope`** | 사용자 이벤트 콜백 내에서 코루틴 실행 | 컴포저블 최초 평가 시 스코프 생성 | 컴포지션 이탈 시 스코프의 모든 Job 자동 `cancel()` | O (`RememberObserver`) |
| **`LaunchedEffect`** | 컴포지션 생명주기에 바인딩된 비동기 코루틴 실행 | 컴포지션 진입 시 실행, `key` 변경 시 취소 후 재실행 | 컴포지션 이탈 또는 키 변경 시 진행 중인 Job `cancel()` | O (`RememberObserver`) |
| **`produceState`** | 외부 비동기 데이터 소스를 Compose `State<T>`로 변환 | `LaunchedEffect` 기반으로 동작 | 컴포지션 이탈 시 내부 코루틴 자동 `cancel()` | O |
| **서드파티 어댑터**<br>(LiveData/Rx/Flow) | 기존 리액티브 스트림을 Compose 상태 시스템과 통합 | `DisposableEffect` 또는 `produceState`에 위임 | 각 스트림의 `removeObserver`, `dispose`, 취소 메커니즘 호출 | O |

Jetpack Compose에서 부수효과를 다루는 일은 결코 피해야 할 기피 대상이 아닙니다. 오히려 **Effect Handlers**라는 정교한 수명 주기 바인딩 인프라를 올바르게 이해하고 적재적소에 배치할 때, 복잡한 비동기 로직과 하드웨어 리소스 관리 속에서도 누수 없이 우아하고 신뢰할 수 있는 모던 안드로이드 애플리케이션을 완성할 수 있습니다.

---

## 이전 글

- [10장. State Snapshot System (스냅샷 상태 시스템 심층 분석)](/jetpack-compose-internals-10-state-snapshot-system/)
- [9장. Modifier 체인, 노드 드로잉 및 시맨틱스(Semantics)](/jetpack-compose-internals-09-modifiers-drawing-and-semantics/)
- [8장. LookaheadLayout과 애니메이션 레이아웃](/jetpack-compose-internals-08-lookahead-layout/)
- [7장. Advanced Compose Runtime Use Cases](/jetpack-compose-internals-07-advanced-compose-runtime-use-cases/)
- [7장. Layouts and Measurement](/jetpack-compose-internals-07-layouts-and-measurement/)
- [6장. Effects and Effect Handlers (기초 요약)](/jetpack-compose-internals-06-effects-and-effect-handlers/)
- [5장. State Snapshot System (기초 요약)](/jetpack-compose-internals-05-state-snapshot-system/)
- [4장. Compose UI](/jetpack-compose-internals-04-compose-ui/)
- [3장. Compose Runtime (3)](/jetpack-compose-internals-03-compose-runtime-03/)
- [2장. Compose Compiler](/jetpack-compose-internals-02-compose-compiler/)
- [1장. Composable 함수](/jetpack-compose-internals-01-composable-functions/)
