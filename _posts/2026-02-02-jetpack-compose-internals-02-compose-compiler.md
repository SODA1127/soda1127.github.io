---
layout: post
title: "Jetpack Compose Internals - 2장. Compose 컴파일러"
description: Jetpack Compose 컴파일러가 어떻게 동작하는지 깊이 있게 살펴봅니다.
image: 'https://imgur.com/o8Jk4Is.png'
category: 'programming'
date: 2026-02-02 10:00:00
tags:
- kotlin
- android
- jetpack compose
- compose internals
- compiler
introduction: Kotlin 컴파일러 플러그인으로서 Compose 컴파일러의 동작 원리를 알아봅니다.
twitter_text: Jetpack Compose 컴파일러가 어떻게 동작하는지 깊이 있게 살펴봅니다.
---

# 2장. Compose 컴파일러

Jetpack Compose는 여러 라이브러리로 구성되어 있지만, 이 책에서는 세 가지에 집중합니다: **Compose 컴파일러**, **Compose 런타임**, 그리고 **Compose UI**.

Compose 컴파일러와 런타임은 Jetpack Compose의 기둥입니다.

![Compose 아키텍처](/assets/img/compose-internals/chapter2/diag-01.png)
*Compose 아키텍처: 컴파일러, 런타임, UI* Compose UI는 기술적으로 Compose 아키텍처의 일부가 아닙니다. 런타임과 컴파일러는 범용적으로 설계되어 요구사항을 충족하는 모든 클라이언트 라이브러리에서 사용할 수 있기 때문입니다.

---

## Kotlin 컴파일러 플러그인

Jetpack Compose는 코드 생성에 의존합니다. Kotlin과 JVM 세계에서 일반적인 방법은 kapt를 통한 어노테이션 프로세서이지만, Jetpack Compose는 다릅니다. **Compose 컴파일러는 Kotlin 컴파일러 플러그인**입니다.

이를 통해 라이브러리는:
- 컴파일 시간 작업을 Kotlin 컴파일 단계에 포함
- 코드 형태에 대한 더 관련성 높은 정보에 접근
- 전체 프로세스 속도 향상

kapt는 컴파일 전에 실행해야 하지만, 컴파일러 플러그인은 컴파일 프로세스에 직접 인라인됩니다.

Kotlin 컴파일러 플러그인의 또 다른 큰 장점은 **기존 소스를 원하는 대로 수정**할 수 있다는 것입니다(어노테이션 프로세서처럼 새 코드만 추가하는 것이 아님). 요소들의 출력 IR을 수정할 수 있습니다.

> KSP(Kotlin Symbol Processing)에 관심이 있다면, Google이 Kapt의 대체로 제안하는 라이브러리인 KSP를 확인해보세요. "경량 컴파일러 플러그인"을 작성하기 위한 정규화된 DSL을 제안합니다.

---

## Compose 어노테이션

Compose 컴파일러가 스캔하고 마법을 부릴 수 있도록 코드에 어노테이션을 추가합니다. 사용 가능한 모든 어노테이션은 Compose 런타임 라이브러리에서 제공됩니다.

### @Composable

가장 중요한 어노테이션입니다. 1장에서 이미 다뤘지만, 별도의 섹션이 필요합니다.

Compose 컴파일러와 어노테이션 프로세서의 가장 큰 차이점은 **Compose가 어노테이션된 선언이나 표현식을 실제로 변경**한다는 것입니다. `@Composable` 어노테이션은 실제로 타입을 변경하며, 컴파일러 플러그인은 Composable 타입이 비-Composable 타입과 같은 방식으로 처리되지 않도록 모든 종류의 규칙을 적용합니다.

`@Composable`을 통해 타입을 변경하면:
- **"메모리"를 갖게 됨**: `remember`를 호출하고 Composer/슬롯 테이블을 활용할 수 있음
- **라이프사이클을 갖게 됨**: 본문 내에서 시작된 이펙트가 준수할 수 있음
- **identity가 할당됨**: 결과 트리에서 위치를 가지며, 노드를 Composition에 emit할 수 있음

### @ComposeCompilerApi

Compose의 일부가 컴파일러에서만 사용하도록 의도되었음을 표시합니다. 주의해서 사용해야 합니다.

### @InternalComposeApi

공개 API 표면은 변경되지 않더라도 내부적으로 변경될 수 있는 API입니다. Kotlin의 `internal` 키워드보다 더 넓은 범위입니다.

### @DisallowComposableCalls

함수 내에서 Composable 호출이 발생하는 것을 방지합니다. 모든 recomposition에서 호출되지 않는 람다에 가장 적합합니다.

```kotlin
@Composable
inline fun <T> remember(calculation: @DisallowComposableCalls () -> T): T =
    currentComposer.cache(false, calculation)
```

`remember` 함수에서 계산 람다 내의 Composable 호출은 이 어노테이션 덕분에 금지됩니다. 람다는 초기 composition에서만 평가되기 때문입니다.

### @ReadOnlyComposable

Composable 함수에 적용되면 이 Composable의 본문이 composition에 **쓰기 작업을 하지 않고 읽기만** 한다는 것을 의미합니다. 본문의 모든 중첩된 Composable 호출에도 동일하게 적용됩니다.

composition에 쓰기 작업을 하는 Composable의 경우, 컴파일러는 본문을 감싸는 "그룹"을 생성합니다. 그룹은 recomposition에서 데이터를 덮어쓰거나 이동하는 방법에 대한 정보를 제공합니다.

읽기 전용 Composable 예시:
- Material Colors, Typography
- `isSystemInDarkTheme()` 함수
- `LocalContext`, `LocalConfiguration`

### @NonRestartableComposable

함수나 프로퍼티 getter에 적용하면 non-restartable Composable로 만듭니다. 컴파일러는 recompose하거나 recomposition 중에 스킵하는 데 필요한 보일러플레이트를 생성하지 않습니다.

매우 작은 함수에서만 의미가 있으며, 부모/둘러싸는 Composable에 의해 invalidation/recomposition이 구동됩니다.

### @StableMarker

`@Immutable`과 `@Stable` 같은 다른 어노테이션에 어노테이션하는 메타 어노테이션입니다.

`@StableMarker`는 데이터 안정성과 관련된 다음 요구사항을 암시합니다:
- 같은 두 인스턴스에 대해 `equals` 호출 결과가 항상 동일
- 타입의 공개 프로퍼티가 변경되면 항상 Composition에 알림
- 모든 공개 프로퍼티도 안정적

### @Immutable

클래스에 적용되어 **모든 공개 프로퍼티와 필드가 생성 후 변경되지 않는다**는 엄격한 약속입니다. Kotlin의 `val` 키워드보다 더 강력한 약속입니다. `val`은 프로퍼티가 setter를 통해 재할당될 수 없음만 보장하지만, 가변 데이터 구조를 가리킬 수 있기 때문입니다.

`@Immutable`로 안전하게 표시할 수 있는 클래스 예시:
- `val` 프로퍼티만 있는 data class
- 커스텀 getter가 없는 프로퍼티
- 모든 프로퍼티가 primitive 타입이거나 `@Immutable`로 표시된 타입

### @Stable

`@Immutable`보다 약간 가벼운 약속입니다. 적용되는 언어 요소에 따라 다른 의미를 가집니다:

- **타입에 적용**: 타입이 가변적이지만 `@StableMarker`의 의미만 가짐
- **함수나 프로퍼티에 적용**: 함수가 같은 입력에 대해 항상 같은 결과를 반환(순수 함수)

```kotlin
// 안정적으로 표시하여 스킵과 스마트 recomposition을 선호
@Stable
interface UiState<T : Result<T>> {
    val value: T?
    val exception: Throwable?
    
    val hasError: Boolean
        get() = exception != null
}
```

---

## 컴파일러 확장 등록

Compose 컴파일러 플러그인이 가장 먼저 하는 일은 `ComponentRegistrar`를 사용하여 Kotlin 컴파일러 파이프라인에 자신을 등록하는 것입니다. `ComposeComponentRegistrar`는 다양한 목적을 위한 일련의 컴파일러 확장을 등록합니다.

컴파일러 플래그에 따라 몇 가지 확장을 등록합니다:
- Live literals
- 소스 정보 포함
- remember 함수 최적화
- Kotlin 버전 호환성 검사 억제
- IR 변환에서 decoy 메서드 생성

---

## Kotlin 컴파일러 버전

Compose 컴파일러는 매우 특정한 버전의 Kotlin이 필요합니다. 일치하지 않으면 큰 차단 요소이므로 첫 번째로 확인됩니다.

`suppressKotlinVersionCompatibilityCheck` 컴파일러 인수로 이 검사를 우회할 수 있지만, 이는 사용자 책임입니다.

---

## 정적 분석

일반적인 컴파일러 플러그인처럼 첫 번째로 일어나는 것은 **린팅**입니다. 라이브러리 어노테이션을 검색하여 소스를 스캔하고 올바르게 사용되는지 확인하기 위한 중요한 검사를 수행합니다.

모든 이러한 유효성 검사는 컴파일러의 **프론트엔드 단계**에서 수행되어 개발자에게 가능한 가장 빠른 피드백 루프를 제공합니다.

### 정적 검사기

호출, 타입, 선언에 대한 검사기가 Jetpack Compose에 의해 확장으로 등록됩니다.

#### 호출 검사 (Call Checks)

- Composable 함수가 허용되지 않는 곳에서 호출되지 않는지 확인
  - try/catch 블록 내 (지원되지 않음)
  - `@Composable`로 어노테이션되지 않은 함수에서
  - `@DisallowComposableCalls`로 어노테이션된 람다에서
- 잠재적으로 누락된 `@Composable` 어노테이션 감지
- `@ReadOnlyComposable` 함수가 다른 읽기 전용 Composable만 호출하는지 확인
- Composable 함수 참조 사용 금지 (현재 지원되지 않음)

#### 타입 검사 (Type Checks)

`@Composable`로 어노테이션된 타입이 예상되었지만 어노테이션되지 않은 타입이 발견된 경우 보고합니다.

#### 선언 검사 (Declaration Checks)

- 오버라이드가 `@Composable`로 어노테이션되었는지 확인
- Composable 함수가 `suspend`가 아닌지 확인 (지원되지 않음)
- Composable main 함수 금지
- Composable 프로퍼티의 backing field 금지

### 진단 억제

Compose는 `ComposeDiagnosticSuppressor`를 등록하여 일부 언어 제한을 우회합니다:

- 호출 사이트에서 "non-source 어노테이션"으로 어노테이션된 인라인 람다
- Composable 함수에서 함수 타입에 대한 명명된 인수 허용

---

## 런타임 버전 검사

정적 검사기와 진단 억제기가 설치된 후, 코드 생성 직전에 Compose 런타임 버전 검사가 수행됩니다. 런타임이 누락되었거나 오래되었는지 감지할 수 있습니다.

---

## 코드 생성

마침내 코드 생성 단계입니다. 컴파일러 플러그인은 소스를 수정할 수 있습니다. 대상 플랫폼의 최종 코드를 생성하기 전에 언어의 중간 표현(IR)에 접근할 수 있기 때문입니다.

### Kotlin IR

JVM만 대상으로 한다면 Java 호환 바이트코드를 생성할 수 있지만, 모든 플랫폼에 대한 단일 IR 백엔드를 정규화하려는 Kotlin 팀의 최근 계획과 리팩터링에 따라 **IR을 생성하는 것이 훨씬 더 합리적**입니다. IR은 대상 플랫폼과 무관한 언어 요소의 표현으로 존재합니다.

### 로우어링 (Lowering)

"로우어링"은 컴파일러가 더 높은 수준 또는 더 고급 프로그래밍 개념을 더 낮은 수준의 원자적 개념의 조합으로 번역하는 것을 말합니다. 로우어링은 정규화의 한 형태로도 이해할 수 있습니다.

로우어링 단계에서 일어나는 의미 있는 예시:
- 클래스 안정성 추론 및 메타데이터 추가
- live literal 표현식 변환
- 모든 Composable 함수에 암시적 Composer 파라미터 주입
- Composable 함수 본문 래핑 (그룹 생성, 기본 인수 지원, 스킵 등)

---

## 클래스 안정성 추론

**스마트 recomposition**은 입력이 변경되지 않고 안정적인 것으로 간주되는 Composable의 recomposition을 스킵하는 것을 의미합니다.

안정적인 타입이 충족해야 하는 속성:
- 같은 두 인스턴스에 대해 `equals` 호출이 항상 같은 결과 반환
- 타입의 공개 프로퍼티가 변경되면 항상 Composition에 알림
- 모든 공개 프로퍼티가 primitive 타입이거나 안정적인 타입

모든 **primitive 타입**은 기본적으로 안정적이며, **String**과 모든 **함수 타입**도 마찬가지입니다. 불변이기 때문입니다.

Compose는 자동으로 안정성을 추론합니다. 알고리즘은:
1. 모든 적격 클래스를 방문
2. `@StabilityInferred` 어노테이션을 합성
3. 관련 안정성 정보를 인코딩하는 합성 `static final int $stable` 값 추가

**안정적으로 추론되는 경우:**
- `class Foo` (필드 없음)
- `class Foo(val value: Int)` (안정적인 필드만)

**불안정으로 추론되는 경우:**
- `class Foo(var value: Int)` (가변 필드)
- 제네릭 타입 파라미터 사용 (런타임에 결정)
- 내부 가변 상태

```kotlin
// 불안정 - 내부 가변 상태
class Counter {
    private var count: Int = 0
    fun getCount(): Int = count
    fun increment() { count++ }
}
```

인터페이스는 구현 방법을 알 수 없으므로 불안정으로 가정됩니다:

```kotlin
@Composable
fun <T> MyListOfItems(items: List<T>) {
    // List는 가변 방식으로 구현될 수 있음 (ArrayList 등)
    // 불안정으로 가정
}
```

---

## Live Literals 활성화

Live literals는 Compose 도구가 재컴파일 없이 프리뷰에서 변경 사항을 실시간으로 반영할 수 있는 기능입니다.

Compose 컴파일러는 상수 표현식을 `MutableState`에서 읽는 새 버전으로 대체합니다:

```kotlin
// 원본
@Composable
fun Foo() {
    print("Hello World")
}

// 변환 후
@Composable
fun Foo() {
    print(LiveLiterals$FooKt.`getString$arg-0$call-print$fun-Foo`())
}

object LiveLiterals$FooKt {
    var `String$arg-0$call-print$fun-Foo`: String = "Hello World"
    // MutableState로 값을 읽어 실시간 업데이트 가능
}
```

> 이 변환은 개발자 경험 개선을 위한 것이며, 릴리스 빌드에서는 절대 활성화하면 안 됩니다.

---

## Compose 람다 메모이제이션

### Non-composable 람다

Kotlin은 값을 캡처하지 않는 람다를 싱글톤으로 모델링하여 최적화합니다. 그러나 값을 캡처하는 람다에는 이 최적화가 불가능합니다.

Compose는 이러한 경우에 람다를 `remember`로 감싸 메모이제이션합니다:

```kotlin
@Composable
fun NamePlate(name: String, onClick: () -> Unit) {
    // onClick이 값을 캡처하면, Compose가 remember로 감싸 메모이제이션
}
```

캡처된 값이 안정적이어야 메모이제이션이 가능합니다.

### Composable 람다

Composable 람다의 경우, IR이 수정되어 `composableLambda(...)` 팩토리 함수를 먼저 호출합니다:

```kotlin
composableLambda($composer, $key, $shouldBeTracked, $arity, expression)
```

목적: 생성된 키를 사용하여 람다 표현식을 저장하기 위한 replaceable 그룹을 composition에 추가합니다.

---

## Composer 주입

Compose 컴파일러가 모든 Composable 함수를 **Composer 합성 파라미터가 추가된 새 버전**으로 대체하는 단계입니다. 이 파라미터는 트리의 모든 지점에서 항상 사용할 수 있도록 모든 Composable 호출에 전달됩니다.

![Composer 주입](/assets/img/compose-internals/chapter2/diag-02.png)
*Composer가 모든 Composable 함수에 주입됩니다*

```kotlin
fun NamePlate(name: String, lastname: String, $composer: Composer) {
    $composer.start(123)
    Column(modifier = Modifier.padding(16.dp), $composer) {
        Text(text = name, $composer)
        Text(text = lastname, style = MaterialTheme.typography.subtitle1, $composer)
    }
    $composer.end()
}
```

---

## 비교 전파 (Comparison Propagation)

모든 Composable에 추가되는 또 다른 메타데이터 조각은 `$changed` 파라미터입니다. 이 파라미터는 현재 Composable의 입력 파라미터가 이전 composition 이후 변경되었을 수 있는지에 대한 단서를 가져옵니다.

```kotlin
@Composable
fun Header(text: String, $composer: Composer<*>, $changed: Int)
```

이 파라미터는 각 함수 입력 파라미터에 대해 이 조건을 나타내는 비트 조합으로 합성됩니다.

이 정보를 통해:
- 입력 파라미터가 변경되었는지 확인하는 equals 비교 스킵 가능
- 비교가 이미 부모 Composable에 의해 수행된 경우 재비교 불필요
- 불확실한 경우, 비교 수행 및 슬롯 테이블에 저장

```kotlin
@Composable
fun Header(text: String, $composer: Composer<*>, $changed: Int) {
    var $dirty = $changed
    if ($changed and 0b0110 === 0) {
        $dirty = $dirty or if ($composer.changed(text)) 0b0010 else 0b0100
    }
    if ($dirty and 0b1011 xor 0b1010 !== 0 || !$composer.skipping) {
        f(text) // 본문 실행
    } else {
        $composer.skipToGroupEnd()
    }
}
```

---

## 기본 파라미터

Kotlin의 기본 인수 지원은 Composable 함수의 인수에 사용할 수 없습니다. Composable 함수는 함수의 생성된 그룹 범위 내에서 인수의 기본 표현식을 실행해야 하기 때문입니다.

Compose는 `$default` 비트마스크 파라미터를 사용하여 대체 구현을 제공합니다:

```kotlin
// 원본
@Composable fun A(x: Int = 0) {
    f(x)
}

// 변환 후
@Composable fun A(x: Int, $changed: Int, $default: Int) {
    // ...
    val x = if ($default and 0b1 != 0) 0 else x
    f(x)
    // ...
}
```

---

## 제어 흐름 그룹 생성

Compose 컴파일러는 각 Composable 함수의 본문에 그룹을 삽입합니다. 본문 내에서 발견된 제어 흐름 구조에 따라 다른 유형의 그룹이 생성됩니다:

### Replaceable 그룹

조건부 로직에서 그룹을 대체해야 할 때 사용됩니다:

```kotlin
if (condition) {
    Text("Hello")
} else {
    Text("World")
}
```

### Movable 그룹

identity를 잃지 않고 재정렬할 수 있는 그룹입니다. `key` 호출의 본문에만 필요합니다:

```kotlin
@Composable
fun TalksScreen(talks: List<Talk>) {
    Column {
        for (talk in talks) {
            key(talk.id) { // Movable 그룹 생성
                Talk(talk)
            }
        }
    }
}
```

### Restartable 그룹

재시작 가능한 Composable 함수에만 삽입됩니다:

```kotlin
// 원본
@Composable fun A(x: Int) {
    f(x)
}

// 변환 후
@Composable
fun A(x: Int, $composer: Composer<*>, $changed: Int) {
    $composer.startRestartGroup()
    // ...
    f(x)
    $composer.endRestartGroup()?.updateScope { next ->
        A(x, next, $changed or 0b1)
    }
}
```

---

## Klib 및 Decoy 생성

.klib(멀티플랫폼) 및 Kotlin/JS에 대한 특정 지원이 Compose 컴파일러에 추가되었습니다. JS에서 의존성의 IR이 역직렬화되는 방식 때문에 필요했습니다.

```kotlin
// 원본
@Composable
fun Counter() { ... }

// 변환 후
@Decoy(...)
fun Counter() { // 시그니처 유지
    illegalDecoyCallException("Counter")
}

@DecoyImplementation(...)
fun Counter$composable( // 시그니처 변경
    $composer: Composer,
    $changed: Int
) {
    ...transformed code...
}
```

---

## 마무리

Compose 컴파일러는 Jetpack Compose의 핵심 엔진입니다. Kotlin 컴파일러 플러그인으로서:

- **정적 분석**: 빠른 피드백 루프를 위한 프론트엔드 검사
- **클래스 안정성 추론**: 스마트 recomposition을 위한 자동 안정성 감지
- **코드 생성**: Composer 주입, 비교 전파, 그룹 생성 등

다음 글에서는 **Compose 런타임**이 어떻게 동작하는지 살펴보겠습니다.

> 이 시리즈는 "Jetpack Compose Internals" 책의 내용을 바탕으로 정리한 것입니다.
