---
layout: post
title: "Jetpack Compose Internals - 10장. State Snapshot System (스냅샷 상태 시스템 심층 분석)"
description: Compose의 리액티브 런타임을 구동하는 핵심 엔진인 State Snapshot System의 내부 동작 원리, MVCC 다중 버전 동시성 제어, StateObject와 StateRecord 연결 리스트, 그리고 원자적 변경 전파 메커니즘을 심층 분석합니다.
image: 'https://imgur.com/o8Jk4Is.png'
category: 'programming'
date: 2026-09-03 09:00:00
tags:
- kotlin
- android
- jetpack compose
- compose internals
- state snapshot
- mvcc
- concurrency
- state record
- recomposition
introduction: Compose가 변경 사항을 감지하고 필요한 부분만 다시 그릴 수 있는 비결은 MVCC 기반의 State Snapshot System에 있습니다. 격리된 스냅샷 트리와 StateRecord 연결 리스트를 통해 동시성 환경에서도 락 프리(lock-free)에 가깝게 원자적 상태 변경을 보장합니다.
twitter_text: Compose의 State Snapshot System 심층 분석: MVCC 동시성 제어, StateRecord 체인, 원자적 Apply 및 병합 메커니즘을 정리합니다.
---

# 10장. State Snapshot System (스냅샷 상태 시스템 심층 분석)

Jetpack Compose는 선언적 UI 프레임워크입니다. 개발자는 데이터의 현재 상태를 기반으로 UI를 기술하고, 상태가 변경되면 컴포즈 런타임이 이를 감지하여 변경이 필요한 컴포저블만 정밀하게 **재구성(Recomposition)**합니다.

과거 안드로이드 뷰(View) 시스템에서는 상태 변경을 UI에 반영하기 위해 수동으로 `notifyDataSetChanged()`, `setText()`를 호출하거나 옵저버 패턴을 일일이 연결해야 했습니다. 반면 Compose에서는 `mutableStateOf`로 감싼 변수의 값을 바꾸기만 해도 관련된 UI가 자동으로 갱신됩니다.

이처럼 마법처럼 보이는 반응형 경험의 심장부에는 **State Snapshot System(상태 스냅샷 시스템)**이라는 정교한 동시성 제어 엔진이 자리 잡고 있습니다. 이번 글에서는 스냅샷 시스템의 설계 사상인 **MVCC(Multiversion Concurrency Control)**부터 시작하여 스냅샷 트리, `StateObject`와 `StateRecord`의 내부 구조, 상태 읽기/쓰기 및 충돌 해결 파이프라인까지 단계별로 깊이 있게 파헤쳐 봅니다.

---

## 1. Snapshot State와 동시성 제어 철학

### 1.1 Snapshot State란 무엇인가?

Compose에서 `mutableStateOf`, `mutableStateListOf`, `derivedStateOf` 등의 API를 통해 생성되는 상태를 흔히 **스냅샷 상태(Snapshot State)**라고 부릅니다. 일반적인 코틀린 변수와 스냅샷 상태의 결정적인 차이는 다음 두 가지입니다:

1. **관찰 가능성(Observability):** 컴포저블 함수 실행 중에 어떤 스냅샷 상태를 읽었는지 런타임이 자동으로 추적합니다.
2. **격리 및 원자성(Isolation & Atomicity):** 특정 연산(예: Composition 생성 또는 백그라운드 계산)이 진행되는 동안 외부 스레드의 변경에 의해 상태가 중간에 깨지지 않도록 독립된 뷰를 제공합니다.

### 1.2 동시성 제어 모델: 왜 MVCC인가?

멀티스레드 환경에서 가변 상태(Mutable State)를 다루는 고전적인 방법은 뮤텍스(Mutex)나 락(Lock)을 사용하는 **비관적 동시성 제어(Pessimistic Concurrency Control)**입니다. 하지만 UI 렌더링 파이프라인에서 락을 사용하면 다음과 같은 심각한 문제가 발생합니다:

- UI 스레드가 백그라운드 쓰기 락에 의해 블로킹되어 프레임 드랍(Jank)이 발생할 수 있습니다.
- 락의 획득 순서에 따라 교착 상태(Deadlock)의 위험이 항상 존재합니다.

반면 함수형 패러다임처럼 완전한 **불변성(Immutability)**을 적용하거나 **액터(Actor) 모델**을 도입할 수도 있지만, UI 컴포넌트처럼 빈번하고 작은 상태 변경이 일어나는 환경에서는 객체 할당 오버헤드와 메시지 패싱 비용이 커집니다.

Jetpack Compose는 데이터베이스 엔진에서 오랫동안 검증된 **MVCC(Multiversion Concurrency Control, 다중 버전 동시성 제어)**와 **낙관적 동시성 제어(Optimistic Concurrency Control)**를 메모리 내 상태 관리에 채택했습니다.

- **스냅샷 격리(Snapshot Isolation):** 각 트랜잭션(스냅샷)은 자신이 생성된 시점의 유효한 데이터 버전만을 일관되게 바라봅니다. 다른 트랜잭션이 상태를 변경하더라도 현재 스냅샷에는 영향을 주지 않습니다.
- **다중 버전 보관(Multi-versioning):** 하나의 상태 객체에 대해 여러 버전의 값을 메모리에 유지합니다.
- **낙관적 쓰기 및 커밋:** 쓰기 작업은 락 없이 로컬 격리 버퍼에 기록되며, 작업이 끝난 후 `apply()` 시점에 충돌 여부를 검사하여 원자적으로 전역 상태에 병합합니다.

---

## 2. Snapshot의 생명주기와 스냅샷 트리 (The Snapshot Tree)

### 2.1 Read-only Snapshot과 Mutable Snapshot

스냅샷 시스템의 기본 진입점은 `Snapshot` 클래스입니다:

```kotlin
// 읽기 전용 스냅샷 생성
val snapshot = Snapshot.takeSnapshot { readObject ->
    // 읽기 관찰자 (선택 사항)
}
try {
    snapshot.enter {
        // 이 블록 안에서 읽는 모든 상태는 스냅샷 생성 시점의 값으로 고정됨
    }
} finally {
    snapshot.dispose()
}
```

- **`Snapshot.takeSnapshot()`**: 읽기 전용 스냅샷(`ReadonlySnapshot`)을 반환합니다. 이 스냅샷 내부에서는 상태 수정이 불가능하며 쓰기를 시도하면 예외가 발생합니다.
- **`Snapshot.takeMutableSnapshot()`**: 가변 스냅샷(`MutableSnapshot`)을 반환합니다. 블록 내에서 상태를 자유롭게 변경할 수 있으며, 변경된 내용은 해당 스냅샷 내부에만 국소적으로 반영됩니다.
- **스냅샷 생명주기:** 스냅샷은 생성(take)되어 활성(active) 상태가 되고, 사용이 끝나면 반드시 `dispose()`를 호출하여 자원을 정리해야 합니다.

```kotlin
// MutableSnapshot의 로컬 격리 동작
var address by mutableStateOf("강남대로")

val snapshot = Snapshot.takeMutableSnapshot()
println(address) // "강남대로"

snapshot.enter {
    address = "테헤란로"
    println(address) // "테헤란로" (스냅샷 내부에서는 변경값 확인)
}

println(address) // "강남대로" (전역 스냅샷에서는 아직 이전 값 유지)

snapshot.apply() // 변경 사항을 전역으로 병합!
println(address) // "테헤란로"
snapshot.dispose()
```

### 2.2 스냅샷 계층 구조 (Snapshot Tree)

Compose 내부에서 스냅샷들은 트리 구조로 계층을 형성합니다.

![스냅샷 트리 구조](/assets/img/compose-internals/chapter10/img-000.png)
*GlobalSnapshot을 루트로 형성되는 계층형 스냅샷 트리*

- **`GlobalSnapshot`**: 트리의 루트에 위치하며, 전체 프로그램의 전역 상태를 보유합니다.
- **중첩 스냅샷(Nested Snapshot):** `takeNestedSnapshot()` 또는 `takeNestedMutableSnapshot()`을 통해 부모 스냅샷 아래에 자식 스냅샷을 생성할 수 있습니다.
  - `NestedReadonlySnapshot`
  - `NestedMutableSnapshot`

이 계층 구조는 Compose의 **서브컴포지션(Subcomposition)**과 완벽히 맞물립니다. `LazyColumn`의 각 아이템, `BoxWithConstraints`, `VectorPainter` 등은 독립적인 무효화 범위를 가지기 위해 자체 서브컴포지션을 생성합니다. 이때 부모 스냅샷 아래에 **중첩 스냅샷**을 만들어 상태를 격리하고, 아이템이 화면 밖으로 스크롤되어 서브컴포지션이 파괴되면 해당 중첩 스냅샷만 단독으로 `dispose()`합니다. 자식 스냅샷의 변경 사항은 먼저 부모 스냅샷으로 전파된 후 최종적으로 루트로 올라갑니다.

### 2.3 스레드와 스냅샷의 관계

스냅샷은 특정 스레드에 종속되지 않는 독립적인 자료구조입니다.

- 어떤 스레드든 `snapshot.enter { ... }` 블록을 실행하여 스냅샷 컨텍스트에 진입하고 빠져나올 수 있습니다.
- 자식 스냅샷은 다른 백그라운드 스레드에서 진입하여 실행될 수 있으며, 이것이 바로 컴포즈가 향후 **병렬 컴포지션(Parallel Composition)**을 수행할 수 있는 이론적 토대가 됩니다.
- 현재 스레드에 바인딩된 스냅샷은 `Snapshot.current`를 통해 조회하며, 스레드에 지정된 스냅샷이 없다면 기본적으로 최상위 `GlobalSnapshot`이 반환됩니다.

---

## 3. 상태 관찰 파이프라인 (Observing Reads and Writes)

Compose 런타임이 "어떤 상태가 변경되었을 때 어느 컴포저블을 다시 호출해야 하는가?"를 알 수 있는 이유는 스냅샷 생성 시 전달되는 **관찰자(Observer)** 덕분입니다.

### 3.1 Read Observer와 `snapshotFlow`

읽기 전용 스냅샷을 생성할 때 `readObserver`를 전달하면, `enter` 블록 내에서 `StateObject`를 읽을 때마다 해당 콜백이 호출됩니다.

코루틴 기반의 `snapshotFlow`가 이 메커니즘을 사용하는 대표적인 예입니다:

```kotlin
fun <T> snapshotFlow(block: () -> T): Flow<T> = flow {
    val readSet = mutableSetOf<Any>()
    val readObserver: (Any) -> Unit = { readSet.add(it) }
    
    // 1. readObserver와 함께 스냅샷을 실행하여 읽은 상태 객체들을 수집
    var lastValue = Snapshot.takeSnapshot(readObserver).run {
        try {
            enter(block)
        } finally {
            dispose()
        }
    }
    emit(lastValue)

    // 2. 이후 전역 스냅샷 apply 감지 시 readSet에 포함된 상태가 변경되었는지 확인하고 재실행
    // ...
}
```

중첩 스냅샷에서 상태를 읽으면 자신의 `readObserver`뿐만 아니라 부모 스냅샷의 `readObserver`에도 상향 전파되어 트리 전체가 관찰 상태를 공유합니다.

### 3.2 Write Observer와 Recomposer의 `composing`

컴포지션(Composition)과 재구성(Recomposition)이 일어날 때, Compose의 `Recomposer`는 다음과 같이 `MutableSnapshot`을 열어 진입합니다:

```kotlin
// Recomposer 내부의 composing 메서드 요약
private inline fun <T> composing(
    composition: ControlledComposition,
    modifiedValues: IdentityArraySet<Any>?,
    block: () -> T
): T {
    val snapshot = Snapshot.takeMutableSnapshot(
        readObserver = readObserverOf(composition),
        writeObserver = writeObserverOf(composition, modifiedValues)
    )
    try {
        return snapshot.enter(block)
    } finally {
        applyAndCheck(snapshot)
    }
}
```

- **`readObserverOf(composition)`**: 컴포저블 실행 중 읽히는 모든 상태를 해당 컴포지션의 스코프(`RecomposeScopeImpl`)에 등록합니다.
- **`writeObserverOf(composition, modifiedValues)`**: 컴포지션 도중 상태 수정이 일어나면 변경된 값을 `modifiedValues` 세트에 기록합니다.
- **`applyAndCheck(snapshot)`**: 실행이 끝나면 변경 사항을 적용하고 유효성을 검사합니다.

---

## 4. StateObject와 StateRecord: 다중 버전 상태 모델링

이제 스냅샷 시스템의 내부 데이터 구조를 살펴보겠습니다. Compose의 모든 스냅샷 상태는 **`StateObject`**와 **`StateRecord`**라는 두 가지 핵심 추상화로 모델링됩니다.

### 4.1 StateObject 인터페이스

모든 관찰 가능한 스냅샷 상태 클래스(`SnapshotMutableStateImpl`, `SnapshotStateList`, `SnapshotStateMap` 등)는 `StateObject` 인터페이스를 구현합니다:

```kotlin
interface StateObject {
    val firstStateRecord: StateRecord

    fun prependStateRecord(value: StateRecord)

    fun mergeRecords(
        previous: StateRecord,
        current: StateRecord,
        applied: StateRecord
    ): StateRecord? = null
}
```

![StateObject와 StateRecord 관계](/assets/img/compose-internals/chapter10/img-002.png)
*StateObject와 StateRecord 연결 리스트의 기본 구조*

`StateObject`는 상태의 버전 기록인 **`StateRecord`들의 단방향 연결 리스트(Singly Linked List)**의 헤드 포인터(`firstStateRecord`)를 가집니다. 새로운 버전이 생기면 `prependStateRecord`를 통해 리스트의 맨 앞에 새 레코드를 추가합니다.

### 4.2 StateRecord 추상 클래스

각 레코드는 특정 스냅샷 시점의 실제 데이터를 담고 있는 불변 단위입니다:

```kotlin
abstract class StateRecord {
    internal var snapshotId: Int = currentSnapshot().id // 레코드가 생성된 스냅샷 ID
    internal var next: StateRecord? = null              // 다음 이전 버전 레코드 포인터

    abstract fun assign(value: StateRecord)
    abstract fun create(): StateRecord
}
```

![StateRecord 연결 리스트 구조](/assets/img/compose-internals/chapter10/img-004.png)
*snapshotId와 next 포인터로 연결된 StateRecord 체인*

- **`snapshotId`**: 이 레코드가 생성된 당시의 스냅샷 ID입니다. 이 ID를 바탕으로 어떤 스냅샷이 이 레코드를 읽을 수 있는지 유효성을 판단합니다.
- **`next`**: 바로 이전 버전의 `StateRecord`를 가리킵니다.

### 4.3 구현체 예시: `mutableStateOf`와 `mutableStateListOf`

`mutableStateOf("Hello")`를 호출하면 내부적으로 `SnapshotMutableStateImpl<T>`가 생성됩니다:

```kotlin
internal open class SnapshotMutableStateImpl<T>(
    value: T,
    override val policy: SnapshotMutationPolicy<T>
) : StateObject, SnapshotMutableState<T> {

    // 단일 값을 감싸는 StateStateRecord
    private var next: StateStateRecord<T> = StateStateRecord(value)
    override val firstStateRecord: StateRecord get() = next

    override var value: T
        get() = next.readable(this).value
        set(value) = next.withCurrent {
            if (!policy.equivalent(it.value, value)) {
                next.overwritable(this, it) { this.value = value }
            }
        }
    // ...
}
```

여기서 사용되는 레코드는 제네릭 값 하나를 보관하는 `StateStateRecord<T>`입니다.

반면, `mutableStateListOf()`의 경우 `SnapshotStateList`라는 `StateObject`가 생성되며, 자체적인 `StateListStateRecord`를 유지합니다. 이 레코드는 Kotlin의 불변 컬렉션 라이브러리인 **`PersistentList`**를 내부 필드로 참조합니다.

![SnapshotStateList와 StateListStateRecord 구조](/assets/img/compose-internals/chapter10/img-006.png)
*SnapshotStateList와 PersistentList를 참조하는 StateListStateRecord 체인*

리스트에 요소를 추가하거나 삭제할 때 기존 리스트를 복사하는 대신 불변 `PersistentList`의 구조 공유(structural sharing)를 활용하므로, 메모리 낭비 없이 스냅샷별 리스트 버전을 고속으로 분기할 수 있습니다.

---

## 5. 상태 읽기 및 쓰기 알고리즘 (Reading & Writing State)

### 5.1 레코드 유효성 검사 (Validity Rules)

어떤 스냅샷 $S$가 `StateObject`를 읽으려 할 때, 연결 리스트에 여러 버전의 `StateRecord`가 존재한다면 어떤 것을 반환해야 할까요?

스냅샷 $S$의 관점에서 유효(Valid)한 레코드는 다음 조건을 만족해야 합니다:
1. **생성 시점 조건:** `record.snapshotId <= S.id` (현재 스냅샷이 생성된 이후에 만들어진 미래의 레코드는 볼 수 없음)
2. **미확정 스냅샷 제외:** $S$가 생성되던 시점에 아직 열려(open) 있던 다른 스냅샷들의 ID 집합(`invalid set`)에 `record.snapshotId`가 포함되지 않아야 함
3. **폐기 여부:** 적용되지 않고 폐기(`disposed`)된 스냅샷에서 생성된 레코드가 아니어야 함

![스냅샷 유효 버전 탐색 과정](/assets/img/compose-internals/chapter10/img-008.png)
*StateRecord 연결 리스트를 순회하며 유효 조건을 만족하는 최신 snapshotId 레코드를 탐색*

Compose는 `firstStateRecord`부터 `next` 링크를 따라가며 위 조건을 만족하는 레코드 중 **가장 높은 `snapshotId`를 가진 유효 레코드**를 선택합니다.

### 5.2 읽기 메커니즘: `readable()`

프로퍼티 getter를 호출하면 `next.readable(this)`가 실행됩니다:

1. 현재 스레드의 스냅샷(`Snapshot.current`)을 획득합니다.
2. 등록된 `readObserver`가 있다면 현재 `StateObject`를 인자로 넘겨 통지합니다.
3. `readable` 헬퍼 함수가 연결 리스트를 순회하여 현재 스냅샷에 대해 유효한 최신 `StateRecord`를 찾아 반환합니다.

### 5.3 쓰기 메커니즘: `withCurrent` & `overwritable()`

프로퍼티 setter를 호출하면 다음 흐름으로 안전한 쓰기가 이루어집니다:

1. **`withCurrent`**: 내부적으로 `readable()`을 호출하여 현재 유효한 레코드를 가져옵니다.
2. **동등성 검사(`policy.equivalent`)**: 새 값이 현재 값과 같은지 비교합니다. 같다면 쓰기 작업을 조기 종료하여 불필요한 무효화를 방지합니다.
3. **`overwritable()`**:
   - 현재 가변 스냅샷에서 이미 수정된 적이 있는 레코드가 있다면 해당 레코드를 재사용합니다.
   - 그렇지 않다면 새 `StateRecord` 인스턴스를 생성(`create()`)하고 현재 스냅샷 ID를 부여한 뒤, `prependStateRecord`로 리스트의 맨 앞에 삽입합니다.
4. 블록을 실행하여 새 값을 할당합니다.
5. 현재 스냅샷의 `writeObserver`에 해당 객체의 수정을 통지합니다.

---

## 6. 만료된 레코드의 정리 및 재활용 (Recycling Obsolete Records)

MVCC는 버전이 누적될수록 메모리를 소비한다는 단점이 있습니다. Compose는 이를 방지하기 위해 정교한 **레코드 재활용(Record Recycling)** 시스템을 갖추고 있습니다.

### 6.1 열린 스냅샷(Open Snapshots) 추적

Compose 런타임은 단조 증가하는 정수 ID를 발급하며, 현재 시스템에 열려 있는 모든 스냅샷 ID를 비트셋 형태의 추적 집합으로 관리합니다.

- 스냅샷이 생성되면 `open snapshots` 집합에 추가됩니다.
- 스냅샷이 `close`되거나 `dispose`되면 집합에서 제거됩니다.

### 6.2 가려진 레코드(Obscured Records)의 판별과 재사용

Compose는 현재 열려 있는 스냅샷 중 **가장 낮은 ID(Lowest open snapshot ID)**를 지속적으로 추적합니다.

> **규칙:** 만약 어떤 레코드보다 더 최신인 유효 레코드가 이미 존재하고, 그 레코드의 ID가 시스템 내의 최하위 열린 스냅샷 ID보다 작거나 같다면, 이전 레코드는 앞으로 어떤 스냅샷에 의해서도 영원히 읽힐 수 없습니다.

이렇게 가려진(obscured) 오래된 레코드는 GC의 대상이 되도록 연결을 끊거나, 다음 쓰기 시 새 레코드를 할당하는 대신 덮어씌워 재활용합니다. 이 덕분에 실무에서 대부분의 `StateObject`는 **단 1개 또는 2개의 레코드만 유지**하며, 메모리 오버헤드가 극히 적습니다.

---

## 7. 변경 사항의 원자적 전파 및 충돌 병합 (Change Propagation & Merging)

가변 스냅샷에서 로컬 변경을 마친 후 이를 시스템 전체에 반영하려면 `snapshot.apply()`를 호출해야 합니다.

### 7.1 스냅샷의 닫기(Closing)와 전진(Advancing)

- **Closing:** 스냅샷 ID를 열린 스냅샷 세트에서 제거합니다. 이 순간부터 해당 ID로 생성된 레코드들은 새롭게 열리는 스냅샷들에게 즉시 유효(valid)한 것으로 인식됩니다.
- **Advancing:** 기존 스냅샷을 닫음과 동시에 ID를 1 증가시킨 새 스냅샷으로 즉시 교체하는 과정입니다. 최상위 `GlobalSnapshot`은 명시적으로 닫히지 않고 변경이 전파될 때마다 끊임없이 **전진(Advance)**합니다.

### 7.2 `apply()`의 실행 단계

`snapshot.apply()`가 호출되면 다음 두 가지 시나리오로 처리됩니다:

#### 1) 로컬 변경 사항이 없는 경우
- 가변 스냅샷을 즉시 닫습니다.
- 전역 스냅샷을 전진시키고 자원을 정리합니다.

#### 2) 로컬 변경 사항이 있는 경우 (수정된 StateObject 목록 존재)
1. **충돌 감지:** 현재 스냅샷이 실행되는 동안 전역 상태나 부모 스냅샷에서 동일한 `StateObject`가 수정되었는지 확인합니다.
2. **낙관적 병합 시도:** 충돌이 감지되면 각 객체의 `mergeRecords()`를 호출하여 충돌 해결을 시도합니다.
3. **레코드 추가:** 병합이 성공하면 새 레코드를 생성하여 리스트의 맨 앞에 추가합니다.
4. **전역 반영 및 통지:**
   - 부모 스냅샷 또는 전역 스냅샷에 수정된 객체 세트를 합칩니다.
   - 전역 스냅샷을 전진시킵니다.
   - 시스템에 등록된 모든 **`Snapshot.registerApplyObserver`** 콜백을 트리거합니다.
5. **Recomposition 유도:** `Recomposer`가 등록해 둔 Apply Observer가 호출되면, 수정된 상태 객체를 읽었던 컴포저블들의 `RecomposeScope`를 `invalidate()`하여 다음 프레임에 재구성이 일어나도록 스케줄링합니다.

### 7.3 충돌 병합 전략과 `SnapshotMutationPolicy`

스냅샷 간의 쓰기 충돌을 해결하기 위해 `StateObject.mergeRecords`는 `SnapshotMutationPolicy`에 정의된 병합 규칙을 위임받아 수행합니다:

```kotlin
interface SnapshotMutationPolicy<T> {
    fun equivalent(a: T, b: T): Boolean
    fun merge(previous: T, current: T, applied: T): T? = null
}
```

- **기본 정책:** Compose가 제공하는 기본 정책(`structuralEqualityPolicy`, `referentialEqualityPolicy`)은 `merge`를 지원하지 않고 `null`을 반환합니다. 따라서 두 스냅샷이 동일 상태를 동시에 변경하고 둘 다 `apply()`를 시도하면 런타임 충돌 예외(`SnapshotApplyConflictException`)가 발생합니다.
- Compose UI는 컴포지션 시 각 상태에 고유한 슬롯 위치를 부여하므로 일반적인 UI 단일 스레드 환경에서는 충돌이 일어나지 않도록 보장합니다.

하지만 개발자가 직접 커스텀 병합 정책을 정의하여 **충돌 없는 상태 객체(Conflict-free State)**를 만들 수도 있습니다. 대표적인 예가 카운터(Counter) 정책입니다:

```kotlin
// 충돌 없이 증분값을 자동 합산하는 CounterPolicy
fun counterPolicy(): SnapshotMutationPolicy<Int> = object : SnapshotMutationPolicy<Int> {
    override fun equivalent(a: Int, b: Int): Boolean = a == b

    // previous: 변경 전 기준 값, current: 현재 부모/전역 값, applied: 이번 스냅샷이 쓰려는 값
    override fun merge(previous: Int, current: Int, applied: Int): Int {
        return current + (applied - previous) // 이번 스냅샷의 델타(증분)를 현재 값에 누적
    }
}
```

```kotlin
// 사용 예시
val counter = mutableStateOf(0, counterPolicy())

val snapshot1 = Snapshot.takeMutableSnapshot()
val snapshot2 = Snapshot.takeMutableSnapshot()

try {
    snapshot1.enter { counter.value += 10 }
    snapshot2.enter { counter.value += 20 }

    snapshot1.apply().check() // counter = 10
    snapshot2.apply().check() // 충돌 없이 자동 병합되어 counter = 30이 됨!
} finally {
    snapshot1.dispose()
    snapshot2.dispose()
}
```

이와 같은 원리로 추가만 가능한 Set(Add-only Set)이나 CRDT(Conflict-free Replicated Data Types) 자료구조를 스냅샷 시스템 위에 안전하게 구축할 수 있습니다.

---

## 요약

| 핵심 컴포넌트 | 내부 메커니즘 및 주요 역할 |
| :--- | :--- |
| **MVCC 동시성 모델** | 락(Lock) 없이 스냅샷 격리를 통해 안전한 동시 읽기/쓰기 환경 제공 |
| **Snapshot Tree** | `GlobalSnapshot`을 루트로 중첩 스냅샷을 구성하여 서브컴포지션 라이프사이클과 상태를 격리 |
| **StateObject & StateRecord** | 단방향 연결 리스트 체인으로 버전별 상태를 관리하며 최신 유효 버전을 O(1)~O(k)로 조회 |
| **Record Recycling** | 최하위 열린 스냅샷 ID를 추적하여 불필요해진 이전 레코드를 메모리에서 즉시 덮어쓰기 재사용 |
| **Apply & Notify** | 로컬 변경 사항을 검증/병합 후 전역으로 전진시키고, Apply Observer를 통해 `RecomposeScope`를 무효화 |

Jetpack Compose의 눈부신 선언적 UI와 자동 리컴포지션은 표면적인 문법 설탕(Syntax Sugar)이 아니라, 이처럼 데이터베이스 수준의 견고한 **State Snapshot System**이라는 기반 인프라 위에서 동작하고 있습니다.

---

## 이전 글

- [9장. Modifier 체인, 노드 드로잉 및 시맨틱스(Semantics)](/jetpack-compose-internals-09-modifiers-drawing-and-semantics/)
- [8장. LookaheadLayout과 애니메이션 레이아웃](/jetpack-compose-internals-08-lookahead-layout/)
- [7장. Advanced Compose Runtime Use Cases](/jetpack-compose-internals-07-advanced-compose-runtime-use-cases/)
- [7장. Layouts and Measurement](/jetpack-compose-internals-07-layouts-and-measurement/)
- [6장. Effects and Effect Handlers](/jetpack-compose-internals-06-effects-and-effect-handlers/)
- [5장. State Snapshot System](/jetpack-compose-internals-05-state-snapshot-system/)
- [4장. Compose UI](/jetpack-compose-internals-04-compose-ui/)
- [3장. Compose Runtime (3)](/jetpack-compose-internals-03-compose-runtime-03/)
- [2장. Compose Compiler](/jetpack-compose-internals-02-compose-compiler/)
- [1장. Composable 함수](/jetpack-compose-internals-01-composable-functions/)
