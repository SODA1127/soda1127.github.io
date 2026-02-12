---
layout: post
title: "Jetpack Compose Internals - 5장. State Snapshot System"
description: Compose의 상태 스냅샷 시스템이 동시성 환경에서 상태를 안전하게 관리하고 변경을 전파하는 방식을 살펴봅니다.
image: 'https://imgur.com/o8Jk4Is.png'
category: 'programming'
date: 2026-02-13 23:00:00
tags:
- kotlin
- android
- jetpack compose
- compose internals
- state
- snapshot
- concurrency
introduction: State Snapshot System은 Compose가 상태를 안전하게 격리하고 변경을 원자적으로 적용해 recomposition을 트리거하는 핵심 메커니즘입니다.
twitter_text: Compose의 State Snapshot System이 상태 격리, 동시성 제어, 변경 전파를 어떻게 수행하는지 정리합니다.
---

# 5장. State Snapshot System

Compose가 “반응형”으로 동작하는 핵심은 **State Snapshot System**입니다. 이 시스템은 상태를 안전하게 격리하고, 변경을 원자적으로 반영하며, 필요한 범위만 재구성(Recomposition)되도록 연결합니다. 이번 장에서는 **스냅샷 상태가 무엇인지**, **동시성 제어가 왜 필요한지**, **스냅샷이 어떻게 변경을 적용하고 충돌을 해결하는지**를 따라가 봅니다.

---

## Snapshot State란?

Snapshot state는 `mutableStateOf`, `derivedStateOf`, `mutableStateListOf` 같은 API로 만들어지는 **관찰 가능한 상태**입니다. 컴파일러가 Composable 내부에서 **state 읽기**를 자동 추적하고, 값이 바뀌면 RecomposeScope를 무효화해 다시 실행되도록 해 줍니다.

![Snapshot State Overview](/assets/img/compose-internals/chapter5/img-000.png)

핵심은 **상태를 읽는 곳과 변경을 반영해야 하는 곳을 자동으로 연결**하는 것입니다. 이 덕분에 View 시스템처럼 수동으로 notify를 호출할 필요가 없습니다.

---

## 동시성 제어와 스냅샷의 역할

Compose는 멀티 스레드에서도 Composition이 가능하기 때문에, **공유된 변경 가능 상태**를 안전하게 다룰 장치가 필요합니다. 여기서 스냅샷 시스템은 **동시성 제어(Concurrency Control)** 모델을 적용합니다.

- **낙관적(Optimistic)** 접근: 읽기/쓰기 차단 없이 진행하고, 충돌 시 롤백
- **비관적(Pessimistic)** 접근: 충돌을 미리 막기 위해 잠금

Compose는 **낙관적 모델에 가까운 방식**으로 동작하며, 스냅샷을 통해 변경을 모았다가 **원자적으로 적용**합니다.

![Concurrency Control Model](/assets/img/compose-internals/chapter5/img-001.png)

이 설계 덕분에 **동시 쓰기 충돌을 최소화**하면서도 성능을 유지할 수 있습니다.

---

## Snapshot과 StateRecord

스냅샷은 상태의 **특정 시점 버전**을 의미합니다. 모든 스냅샷 상태 객체는 내부에 **StateRecord**를 가지고 있으며, 이 레코드는 **버전과 값**을 함께 관리합니다.

![StateRecord Versioning](/assets/img/compose-internals/chapter5/img-002.png)

동작 흐름은 다음과 같습니다:

1. 스냅샷이 열리면 현재 버전을 기준으로 읽기/쓰기를 수행
2. 변경은 **스냅샷 내부에만 반영** (격리)
3. 스냅샷이 적용될 때 **전역 상태에 병합**

즉, 스냅샷은 **"격리된 변경 버퍼"** 역할을 합니다.

---

## Apply / Merge와 충돌 해결

스냅샷이 닫히면서 `apply()`가 호출되면, 변경 사항이 전역 상태로 병합됩니다. 이때 Compose는 **StateRecord의 버전 충돌**을 확인합니다.

- **충돌 없음** → 변경 적용
- **충돌 발생** → 병합 실패, 재시도 or 무시

![Snapshot Apply Flow](/assets/img/compose-internals/chapter5/img-004.png)

이 과정은 **트랜잭션 메모리 모델**과 유사하며, 모든 변경이 **원자적(Atomic)**으로 반영되도록 보장합니다.

---

## Snapshot Observer와 변경 전파

스냅샷이 성공적으로 적용되면 **전역 스냅샷 옵저버**가 이를 감지해, 해당 상태를 읽었던 Composable을 무효화합니다. 즉, **변경 전파의 출발점**이 됩니다.

![Global Snapshot Observer](/assets/img/compose-internals/chapter5/img-006.png)

이 구조 덕분에 Compose UI는 **상태 변화 → invalidation → recomposition** 흐름을 안정적으로 유지합니다.

---

## 요약

State Snapshot System은 다음을 담당합니다:

1. ✅ 상태 읽기 추적 및 자동 무효화
2. ✅ 동시성 환경에서 상태 격리
3. ✅ 변경의 원자적 적용 및 충돌 처리
4. ✅ 변경 전파를 통한 recomposition 유도

**Compose의 반응성은 결국 스냅샷 시스템이 만들어낸 결과**입니다. 이 레이어가 탄탄하기 때문에 상위 레벨의 UI 코드는 단순하고 선언적으로 유지될 수 있습니다.

---

### 다음 장 예고

다음 장에서는 **Effects와 Effect Handlers**를 다룹니다:
- ✨ LaunchedEffect / DisposableEffect
- ✨ SideEffect와 remember
- ✨ 코루틴과 Compose의 연결
- ✨ 안전한 부수효과 처리
