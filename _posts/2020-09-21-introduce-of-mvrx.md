---
layout: post
title: "Airbnb - MvRx 아키텍쳐 소개"
description: 이전에 사용해보았던 MvRx아키텍쳐 소개 및 리뷰
image: 'https://imgur.com/1Z21UTK.jpg'
category: 'programming'
date: 2020-09-21 00:45:00
tags:
- kotlin
- coroutines
- android
- android-jetpack
- rxjava
- android-lifecycle
introduction: 에어비앤비에서 만든 MvRx 아키첵쳐를 소개합니다.
twitter_text: 에어비앤비에서 만든 MvRx 아키첵쳐를 소개합니다.

---

# MvRx(a.k.a. mavericks): Android on Autopilot

에어비엔비에서 만든 현 프로덕트에서 쓰이는 안드로이드 프레임워크

이전에 본인이 개발한 앱의 경우 80% MvRx 아키텍쳐로 구성되어 있음.

MvRx는 해당 기술들을 기반으로 동작하도록 되어 있다.

- Kotlin
- Android Architecture Components
- RxJava
- React (conceptually)
- [Epoxy](https://github.com/airbnb/epoxy) (optional) - 리액트 컴포넌트 방식에서 착안

컨셉은 UDA(Uni-Direction Architecture)에서 고안하였으며, Redux 패턴을 참고함.

### Core Concept - State

- **State - 기본적으로 MvRxState 인터페이스를 기반으로 구현한 데이터 클래스**

```kotlin
data class MyState(val listing: Async<Listing> = Uninitialized) : MvRxState
```

- **Immutabillity**

MvRxState는 강제로 불변적이고, 디버그 모드에서는 아예 체크해서 Exception 에러 날려버림.

선언형 프로그래밍 방식이며, 기본적으로 단방향 Flow이기 때문에 데이터는 Immutable함. 따라서, 매번 데이터 변경 시 data class copy()함수를 사용하여 state를 복제하고, reduce 해주어야 함

```kotlin
setState {
    copy(
				a = aValue
    )
}
```

- **State Creation**

```kotlin
internal class AViewModel(
    state: AState,
) : MvRxViewModel<AState>(state) {
	...
	companion object : MvRxViewModelFactory<AViewModel, AState> {
        override fun create(viewModelContext: ViewModelContext, state: SignUpState): AViewModel {
            return AViewModel(state) // 필요시 DI가 필요한 객체를 viewModelContext를 통해 주입 가능
        }
    }
}
```

### Core Concept - ViewModel<S>

![ViewModel](https://imgur.com/x3um4uy.jpg)

AAC ViewModel의 라이프사이클을 따르며, 차이점이라면 MvRxViewModel은 불변성의 단일 MvRxState를 가지며, AAC ViewModel에서 가지는 동적 변경을 위한 데이터 홀더인 LiveData 대신 뷰에서는 오직 State를 관찰함.

- **Initial State**

MvRxViewModels(ViewModel 집합)은 생성 시 `initalState()`  메서드를 호출해야 하며, 이때 기본적으로 state의 기본값으로 새 인스턴스를 만들어 state를 set해줌

만약 외부 의존성이 없는경우, ViewModel은 State를 프로퍼티로 등록해주면됨.

다만, 외부 의존성이 있는 경우 `MvRxViewModelFactory` 를 사용하여 create해주면 됨.

```kotlin
class MyViewModel(initialState: MyState, dataStore: DataStore) : BaseMvRxViewModel(initialState, debugMode = true) {
		...
    companion object : MvRxViewModelFactory<MyViewModel, MyState> {

				override fun create(viewModelContext: ViewModelContext, state: MyState): MyViewModel {
            val dataStore = if (viewModelContext is FragmentViewModelContext) {
              // If the ViewModel has a fragment scope it will be a FragmentViewModelContext, and you can access the fragment.
              viewModelContext.fragment.inject()
            } else {
              // The activity owner will be available for both fragment and activity view models.
              viewModelContext.activity.inject()
            }
            return MyViewModel(state, dataStore)
        }
    } 
}
```

- **State factory**

초기화 할 state 생성 시 `initalState()` 메서드를 호출하며, 이때 받아오는 viewModelContext에는 Fragment의 Args를 꺼내서 받을수도 있고, attached한 parent activity를 통해 DI도가능

```kotlin
class MyViewModel(initialState: MyState, dataStore: DataStore) : BaseMvRxViewModel(initialState, debugMode = true) {
    companion object : MvRxViewModelFactory<MyViewModel, MyState> {

        override fun initialState(viewModelContext: ViewModelContext): MyState? {
            // Args are accessible from the context.
            // val foo = vieWModelContext.args<MyArgs>.foo

            // The owner is available too, if your state needs a value stored in a DI component, for example.
            val foo = viewModelContext.activity.inject()
            return MyState(foo)
        }

    } 
}
```

- Accessing State

`withState` 블록함수를 통해 비동기적으로 쓰레드를 생성하며(new Thread), 각각의 동작은 순차적이지 않음.

![Accessing State](https://imgur.com/GXWrkLG.jpg)

따라서, 순차적인 방식의 데이터 갱신 로직이 들어가는 경우, withState 블록을 여러개 생성하여 setState를 해주게되면 쓰레드 스케쥴링에 따라 순서가 달라지게 되니 최종적으로 한 값을 setState로 값을 reduce 해주는 것이 좋음.

![RealMvrxStateStore](https://imgur.com/uzTTEOE.jpg)

보는 것과 같이 새 쓰레드를 생성하여 state의 프로퍼티를 조작하지만, 최종적으로는 flush를 통해 큐에 쌓인 state를 reduce하여 reduced state로 반영한다.

- **Updating State(Mutating State)**

`setState` 함수 블록 스코프내에서 현재 ViewModel에서 hold중인 State를 수정할 수 있다. BaseMvRxViewModel에서 호출 가능한 함수이며, state를 리시버로 받기때문에 immutable한 새 state를 copy()해준다.

**[참고사항]**

1. 같은 쓰레드에서 동기적으로 호출하는 것은 퍼포먼스 이슈때문에 동작하지 않는다.
2. 람다 호출시점의 경우 count → count + 1은 실제로 count + 1의 값을 리시버 타입으로 받기 때문에 최종적으로 count + 1의 값으로 반영된다.
3. 디버그모드의 경우 새 인스턴스가 아닌 기존 state의 프로퍼티 값을 변형하는 경우 예외 발생시킴.

![setState](https://imgur.com/r67n5q6.jpg)

ex) access & mutate state

```kotlin
data class AState(
		val aList: Async<List<A>> = Uninitialized,
		...
): MvRxState

// withState 블록에서 새로운 쓰레드를 생성함
fun onResponseWith(...) = withState { state -> // AState애 있는 불변의 프로퍼티를 꺼내 사용
		setState { //this@AState
				... //어떠한 로직을 처리한 이후 (S.() -> S)로 reduce한다.
				copy( // return 생략
						aList = Success(list)
				)
		}		
}
```

**Async<T>**

- sealed class로 구성된 하위 4개의 subClass가 존재
  - Uninitialized - 말 그대로 초기화가 되지 않은 상태 - 이때는 data hold하지 않음.
  - Loading - 이때부터 value를 부여할 수 있음. 아직 completed된 값은 아님.
  - Success - 성공적으로 값을 부여받음.
  - Fail - 값과 함께 error(Exception)을 넘겨받을 수 있음.
- **Complete / ShouldLoad** 두가지의 상태를 가지며, 값이 오기까지 비동기적으로 대기해야할 프로퍼티인 경우 해당 객체를 사용

자세한 사용방법은 아래 링크를 참조.

[airbnb/MvRx](https://github.com/airbnb/MvRx/wiki)

### Core Concept - MvRxView

MvRxView는 유저가 바라보고, 상응하는 것이고, 비즈니스 로직, 네트워크 연동과 같은 로직에서 자유롭다.

굉장히 단순한 인터페이스이며, state가 바뀔때마다 View가 dirty할 때 업데이트를 해줘야 하는경우 메인 함수인 `invalidate()` 가 호출된다.

에어비앤비에서는 Fragment에서 MvRxView를 구현했으며, 그러한 이유로 Fragment에서 갖고 있는 고질적인 이슈에서 벗어나도록 하고, 심플한 작성이 가능하다.

뷰들은 `invalidate()` 함수만 바라보고 있으면 되며, state가 변경이 될 때마다 호출이 된다. 이것을 이용하여 더 나아가서 [epoxy](https://github.com/airbnb/epoxy)를 사용하는 것도 방법임.

- **Aceessing State on View**

뷰에서 state에 접근하는 방법은 다음과 같이 확장함수를 통해 withState로 ViewModel의 State를 꺼내 사용이 가능하다.

![subscribe](https://imgur.com/Ct6c4dv.jpg)

```kotlin
// ViewModel이 하나인경우
withState(viewModel) { state ->
	  ...
}

// ViewModel이 여러개인경우
withState(viewModel1, viewModel2...) { state1, state2... ->
		...
}
```

필자의 경우 state안의 결과 리스트가 비어있는지 체크하기 위해 이러한 방법으로도 사용함.

```kotlin
open fun isListEmpty() = withState(listViewModel) { // it: listState
      if (it.list is Success) {
          val list = it.list.invoke()
          return@withState list.isEmpty()
      }
      return@withState true
  }
```

- **Observe State Mutating**

```kotlin
fun observeViews() = with(listViewModel) {
		...
		selectSubscribe(
        prop1 = ListState::list,
        deliveryMode = RedeliverOnStart
    ) {
        when (it) {
            is Uninitialized -> {
                recyclerAdapter?.submitList(listOf())
            }
            is Loading -> {
                val list = it.invoke()
                recyclerAdapter?.submitList(dataList ?: listOf())
								// loading 중인 시점의 로직, 이 때 데이터가 있을 수도, 없을수도 있음
            }
            is Success -> {
                val list = it.invoke()
                checkListEmpty(list)
                recyclerAdapter?.submitList(list)
            }
            is Fail -> { // 필요시 value도 꺼낼 수 있음.
                it.error.printStackTrace()
            }
        }
    }.addTo(compositeDisposable)
}
```

MvRxView에서는 확장함수로 BaseMvRxViewModel를 Context Instance로 가지는 `subscribe(DeliveryMode, (S) -> Unit`

& `selectSubscribe(Property1<S, A>, DeliveryMode, (A) -> Unit)`

&  `asyncSubscribe(KProperty1<S, Async<T>>, DeliveryMode, ((Throwable) -> Unit)?, ((T) -> Unit)?)`

함수를 제공한다.

예시와 같이 다음처럼 사용이 가능하다.

```kotlin
subscribe { state -> }

selectSubscribe(YourState::propA) { a -> }

selectSubscribe(YourState::propA, YourState::propB, YourState::propC) { a, b, c -> }

asyncSubscribe(YourState::asyncProp, onFail = { error -> ... }) { successValue -> ... }

// or
asyncSubscribe(YourState::asyncProp) { successValue -> ... }
```

### Simple Example

```kotlin
data class MyState(val listing: Async<Listing> = Uninitialized) : MvRxState

class MyViewModel(override val initialState: MyState) : MvRxViewModel<MyState>() {
    init {
        fetchListing()
    }

    private fun fetchListing() {
        ListingRequest.forId(1234).execute { copy(listing = it) }
    }
}

class MyFragment : MvRxFragment() {
    private val viewModel: MyViewModel by fragmentViewModel()

    override fun invalidate() = withState(viewModel) { state ->
        loadingView.isVisible = state.listing is Loading
        titleView.text = listing()?.title
    }
}
```

---

## Conclusion - 내가 생각하는 **주관적인 장/단점**

**장점**

- RxJava를 굉장히 쉽게 풀어쓴 느낌, 그리고 RxJava와 결합하여 적극 활용 가능하다.
- 단방향 흐름이며 불변적인 State를 넘기기 때문에 특정 쓰레드에서 데이터를 가공하여 넘기면서 중간에 데이터가 변형될 일이 없다. 따라서, 반드시 비즈니스 로직에서 기존 state에 프로퍼티 값을 변형한 새 객체를 던져주기 때문에 확실한 결과를 얻을 수 있다.
- Activity/fragment scope 데이터 관리를 손쉽게 할 수 있다. (DI시 scope 설정 가능)
- AAC ViewModel 가이드를 따르고 있기 때문에 ViewModelScope를 적극 활용 가능하다.
- 이것도 장점이라면 장점인데, 문서가 완전 자세하게 설명되어 있다.
- 에어비앤비에서 만드는 아키텍쳐라 앞으로도 지원이 빠방할 느낌이다.

**단점**

- 아직 버그 및 이슈가 존재함. 실무에서 충분히 적용 가능한 수준이라고 생각하나, 여전히 불안정한 코드들
- 선언형 프로그래밍, 비동기적 프로그래밍에 익숙하지 않은 경우 초반 러닝커브가 높다고 느껴질 수 있다.(리액트 개발자의 경우 쉽게 적응 가능할듯!)
- state를 한곳에서 관리하기 때문에 컨셉을 정확하게 이해하지 못한채로 개발하면(물론 개발자의 잘못이나) state 관리하는데 어려움이 있다.
- 쓰레드 관리방식이 추상적이라고 느꼈다. (따라서, 개발자가 코드를 보며 어떤 흐름으로 갈지 예상하기 쉽지 않음)
- Rx의존적이다.(단점일수도 있고 장점일수도 있음.)