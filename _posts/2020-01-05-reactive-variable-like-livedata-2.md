---
layout: post
title: "RxJava로 LiveData 따라해보기 2"
description: Android Jetpack에서 제공하는 LiveData처럼 리액티브한 데이터 홀더 클래스를 만들어 봅시다.
image: 'https://imgur.com/2gM2T8c.jpg'
category: 'programming'
date: 2020-01-05 14:40:30
tags:
- kotlin
- idea
- android-zetpack
- android-architecture-component
- android-lifecycle
- rxjava
introduction: Reactive한 방법으로 LiveData와 같은 기능을 하는 데이터 홀더를 만들어 봅시다.
twitter_text: Reactive한 방법으로 LiveData와 같은 기능을 하는 데이터 홀더를 만들어 봅시다.


---

*본 내용은 **Android Developers** 공식 문서에서 발췌한 내용을 기반으로 합니다.*

# 활성화 여부에 따른 로직 구성하기

저번시간에는 `LiveData`의 특징 중 하나인 Lifecycle의 주기 시점을 알아채 자동으로 해제가 가능한 로직을 구성해보는 시간을 가져봤습니다. 이번에는 또다른 특징인 생명주기에 따른 활성화 여부 로직을 구성해 보도록 하겠습니다.



## 활성화 여부 로직 이해하기

`LiveData`에는 굉장히 다양한 특징들이 존재하였습니다. 그 중

> **항상 최신의 데이터를 유지할 수 있다.**

라는 특징은 LiveData의 핵심적인 특징 중 하나입니다.

항상 최신의 **`Value`**를 받아볼 수 있다는 것은 생명주기가 만약 비활성화(Inactive)상태가 되어도 최신의 데이터를 활성(active)상태일때 다시 받을 수 있다는 것을 말합니다.



```java
public abstract class LiveData<T> {
  ...
  /**
   * Called when the number of active observers change to 1 from 0.
   * <p>
   * This callback can be used to know that this LiveData is being used thus should be kept
   * up to date.
   */
  protected void onActive() {

  }

  /**
   * Called when the number of active observers change from 1 to 0.
   * <p>
   * This does not mean that there are no observers left, there may still be observers but their
   * lifecycle states aren't {@link Lifecycle.State#STARTED} or {@link Lifecycle.State#RESUMED}
   * (like an Activity in the back stack).
   * <p>
   * You can check if there are observers via {@link #hasObservers()}.
   */
  protected void onInactive() {

  }
  ...
}
```



`LiveData` 보는 것과 같이 두가지 활성화 여부 콜백 메서드가 존재합니다.

- `void onActive()`
- `void onInActive()`


LiveData에는 활성 수의 수가 0과 1 사이에서 변경 될 때 알림을받는  `onActive()`와 `onInactive()`을 통해 감시중인 옵저버가없는 경우 LiveData가 사용하지 않는 리소스를 해제 할 수 있습니다.



LiveDatad의 옵저버는 Lifecyle에서 지정된 관찰자를 관찰자 목록에 추가합니다.(`addObserver()`) 이벤트는 기본 스레드에서 전달됩니다. LiveData에 이미 데이터 세트가 있으면 관찰자에게 전달됩니다.



`LifecycleObserver`을 통해 `Lifecycle.State.STARTED` 이거나 `Lifecycle.State.RESUMED`상태 (활성) 인 경우에만 이벤트를 수신합니다 .

lifecycleOwner가 `Lifecycle.State.DESTROYED`상태로 이동 하면 관찰자가 자동으로 제거됩니다.



위 메서드가 핵심적인 Lifecycle에 대한 활성화 여부 콜백메서드로 제공되는데, 이는 LiveData의 구조를 깊게 들여보아야합니다. 이 구조를 여러분들과 함께 천천히 들여다보도록 하겠습니다.



구조를 이해하기 전에 이전부터 사용되고 있던 생명주기에 따라 데이터를 관리하는 데이터 홀더인 `Loader` 에 대해 보도록 하겠습니다.

## `Loader` 로더

먼저 로더에 대해 알아보기 전 참고해야 할 사항입니다.

>Google I/O 18에서는 Fragment 로부터 Loader를 분리했습니다. 새로운 Primitive를 사용하여 LiveData와 ViewModel를 가지고 독립적으로 만듭니다. 이제 LifecycleOwner를 가지는 클래스에서 Loader를 사용할 수 있습니다.
>
>
>
>로더는 Android P(API 28)부터 사용이 중단되었습니다. 액티비티와 프래그먼트 수명 주기를 처리하면서 데이터 로딩을 처리할 때는 [`ViewModels`](https://developer.android.com/reference/androidx/lifecycle/ViewModel?hl=ko)와 [`LiveData`](https://developer.android.com/reference/androidx/lifecycle/LiveData?hl=ko)를 함께 사용하는 방법을 추천합니다. 

`Loader` 로더는 안드로이드 3.0에서 소개 된 VC에서 비동기 데이터 로딩을 쉽게 하기 위한 안드로이드 유틸리티 데이터 홀더입니다.

Loader API를 사용하면 [콘텐츠 제공자](https://developer.android.com/guide/topics/providers/content-providers.html?hl=ko) 또는 `FragmentActivity` 또는 `Fragment`에서 표시하기 위한 다른 데이터 소스에서 데이터를 로드할 수 있습니다. 특징은 다음과 같습니다.

-  모든 액티비티와 프래그먼트에서 사용할 수 있습니다.
-  어플리케이션 UI를 Blocking 하지 않도록 비동기 데이터 로딩을 제공합니다.
-  데이터를 모니터 하기 때문에 데이터가 변경되었을때 변경사항을 확인할 수 있습니다.



API 중 저희는 `LoaderManager`에 대해 알아 볼 필요가 있습니다. 왜냐하면 앞에서 언급한 활성화 여부로직에 대한 판단을 LoaderManager를 통해 관리하기 때문입니다.



**`LoaderManager`** 에대한 도큐먼트 소개를 보면 다음과 같습니다.

>`FragmentActivity` 또는 `Fragment`와 연결된 추상 클래스로, 하나 이상의 `Loader` 인스턴스를 관리하는 데 쓰입니다. 각 액티비티나 프래그먼트에는 `LoaderManager`가 하나뿐이지만 `LoaderManager`는 여러 로더를 관리할 수 있습니다.



맞습니다. 바로 `LoaderInfo<D>`에서 Loader의 생명주기에 따라 활성화 여부를 판단하기 때문에, Loader를 연계하고 있는 VC(또는 VM)의 Lifecycle에 따라 관리되게 됩니다.



![LoaderManager getInstance()](https://imgur.com/uxenUOa.jpg)



해당 코드를 보시면 `LoaderManager` 인스턴스를 생성 시 LifecyleOwner, 그리고 그 상태의 정보값을 갖고 생성을 하고 있습니다.



```java 
public abstract class LoaderManager {
  ...
  
  // Loader가 초기화되고 활성화됩니다.
  @MainThread
  @NonNull
  public abstract <D> Loader<D> initLoader(int id, @Nullable Bundle args,
                                           @NonNull LoaderManager.LoaderCallbacks<D> callback);

  ...
  
  // LoaderManager에서 Loader를 새로 시작하거나 재시작 할 때 콜백메서드를 등록해줍니다.
  @MainThread
  @NonNull
  public abstract <D> Loader<D> restartLoader(int id, @Nullable Bundle args,
                                              @NonNull LoaderManager.LoaderCallbacks<D> callback);

  ...
  
  // ID를 가진 로더를 제거 및 중지합니다. 
  @MainThread
  public abstract void destroyLoader(int id);
  
  ...
  
  // LoadManager와 연관된 재전달할 현재 갖고있는 어떤 데이터든 마킹을 해두고, 현재 중지 된 상태인 경우 대기할 수 있도록 합니다.
  public abstract void markForRedelivery();
  
  ...
}
```

위 메서드들은 `LoaderManager`에서 사용되는 데이터 홀더의 상태관리 콜백 메서드들입니다. 해당 메서드에 구현은  `LoaderManagerImpl` 에서 구현이 되고 있는데, 해당 메서드를 보면 어노테이션을 통해 메인스레드에서만 동작하는 것을 알 수 있습니다.



더 자세한 내용은 [Android Developers - LoaderManager](https://developer.android.com/reference/androidx/loader/app/LoaderManager.html?hl=ko)의 내용을 참고하시기 바랍니다.



## 원리를 이해하고 구성해보자

앞에서 보았던 `Loader` 로더와 마찬가지로 **`LiveData`**도 LifecycleOwner를 통해 상태값을 관리를 합니다.



**`LiveData`**는 `observe()`메소드를 통해 Observer를 붙일 수 있으며, `observe()`는 LifecycleOwner 인스턴스를 포함해야 합니다. (왜냐하면 VC가 LifeCycleOwner를 implement 하고 있기 때문입니다.)

![LiveData observe()](https://imgur.com/Lh14r28.jpg)



다음은 `observe()` 메서드의 로직입니다. 인자로 `LifecycleOwner`와 `Observer` 인스턴스를 받고 있는 것을 볼 수 있으며, `LifecycleBoundObserver` 인스턴스를 통해 생명주기에 대한 콜백 메서드가 호출 될때마다 래퍼클래스에서 활성화 여부를 판단합니다.

![LiveData ObserverWrapper](https://imgur.com/utvLaHN.jpg)

위 메서드에서 `activeStateChanged`로 생명주기의 활성화 여부를 인자로 받아 그에 따라 활성화/비활성화 여부를 설정하고 있는 것을 볼 수 있습니다.



이와 다르게 Lifecycle의 활성화/비활성화 여부와 상관없이 관찰을 지속적으로 하는 메서드도 제공합니다.

![LiveData observeForever()](https://imgur.com/Uc4saKR.jpg)

해당 메서드는 위 **ObserverWrapper**와 다르게 `shouldBeActive()`메서드를 항상 **true**로 반환하고 있어 LifecyleCycle과는 무관하게 항상 관찰을 하고 있습니다.



구조에 대한 이해를 토대로 코드를 작성해보록 하겠습니다.

```kotlin
class AutoActivatedSubscription(
  private val lifecycleOwner: LifecycleOwner,
  private val func: () -> Subscription
) : LifecycleObserver
```

먼저 자동으로 Lifecycle의 활성화 유무를 판단하여 관찰자의 구독을 관리하는 클래스를 만들었습니다.

**LiveData**의 `observe()` 메서드와 마찬가지로 `lifecycleOwner`를 통해 활성화 여부를 설정 할 것이고, Cold 옵저버블 인스턴스에서 구독시 처리되는 로직을 구성하기 위해 다음과 같이 구성하였습니다.



```kotlin
class AutoActivatedSubscription ... {

  private var subscription: Subscription? = null
  
  init {
    subscription = func.invoke()
  }

  @OnLifecycleEvent(Lifecycle.Event.ON_START)
  fun active() {
    subscription = func.invoke()
  }

  @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
  fun deActive() = subscription?.let { if (!it.isUnsubscribed) it.unsubscribe() }

  @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
  fun detachSelf() = lifecycleOwner.lifecycle.removeObserver(this)
}
```



외부에서 사용 시 해당 인스턴스 생성시에 `observe()`와 마찬가지로 `LifecycleOwner` 인스턴스와 옵저버  인스턴스를 구성하고, 구독처리를 했을 것입니다.



먼저 `init()` 메서드로 구독 시 반환된 Subscriprion 인스턴스를 반환 받습니다.



이후에 모든 생명주기에 따른 활성화 여부 로직은 **`@OnLifecycleEvent`** 어노테이션을 통해 콜백으로 관리하게 됩니다.



전 포스트에서 VC의 Lifecycle의 주기 시점을 알아채 자동으로 해제가 가능한 로직을 구성해 보았고, 이번 포스트에선 활성화 여부에 따른 로직을 구성해 보았습니다. 최종적으로 정리 된 코드는 아래와 같습니다.



{% gist watcha-jimmy/39aa3a949899c0e6df062eb27e2bc3e9 %}



그러면 이번에는 두 로직을 합쳐 `LiveData`의 특징들을 모방하여 **`Variable`** 데이터 홀더 클래스를 만들어보겠습니다.



## LiveData 따라하기 - Variable

본격적으로 앞에서 보여드린 로직을 합쳐 Lifecycle에 따른 관리가 가능한 데이터 홀더 클래스를만들어 보겠습니다.



본격적으로 데이터의 스트림을 형성하여 데이터를 배출하고, 홀드하는 로직을 보겠습니다.

![LiveData setValue()](https://imgur.com/bR0nJGR.jpg)

![LiveData postValue()](https://imgur.com/PlFXbYQ.jpg)

위에 두 메서드는 동일한 기능을 수행하고 있습니다. 다만 사용처가 다른 것을 알아야합니다.

`setValue(T)` 메서드의 경우 메인(UI)스레드에서만 사용 가능하며 값을 넣어주면 메인스레드에서 바로 dispatch하게 됩니다., 서브스레드에서 호출하는 경우 **llegalStateException**이 발생하게 됩니다.

반면에 `postValue(T)`는 다른 서브스레드에서 받은 값을 메인스레드로 전달하는 기능을 제공하기 때문에,  내부적으로는 백그라운드에서 받은 **Value**를 메인스레드에 전달하는 과정을 거치게 됩니다. 쉽게 설명하자면

```kotlin 
Handler(Looper.mainLooper()).post { setValue() }
```

다음과 같은 기능을 수행한다고 생각하면 됩니다.



다음 기능을 직접 구성한 데이터 홀더 클래스에서 동일한 기능을 수행하는 함수를 구성해보겠습니다.

다만 여기서는 LiveData와 다르게 스레드에 대한 디스패치하는 로직을 넣지는 않았습니다. Variable은 결국 RxJava의 Obsevable 스트림 관리를 하는 데이터 홀더이기때문에, 충분히 Rx에서 제공하는 빌더패턴를 통해 어떤 스레드에서 디스패치 할지 구성이 가능하다는 것을 알고 있으면 되겠습니다.

``` kotlin
class Variable<T>
@JvmOverloads
constructor(
    @Volatile var value: T? = null,
    private val alwaysClearOnStop: Boolean = false
) : LifecycleObserver
```

`Variable` 생성자입니다. `LiveData`와 유사하게 초기화 시 초기 **Value**를 받는 것을 볼 수 있고, 구독에 대한관리를 Lifecycle에 따라 구독을 clear할 것인지를 정하는 조건변수를 주었습니다.



그리고 **LiveData**와 같이 구독 시 스트림의 최신값을 받아올 수 있도록 RxJava의 `BehaviorSubject`를 사용하였습니다.



`BehaviorSubject`의 성질에 대해서 잠깐 언급하자면,

![BehaviorSubject](https://imgur.com/uTV6I88.jpg)

>옵저버가 `BehaviorSubject`를 구독하기 시작하면, 옵저버는 소스 `Observable`이 가장 최근에 발행한 항목(또는 아직 아무 값도 발행되지 않았다면 맨 처음 값이나 기본 값)의 발행을 시작하며 그 이후 소스 Observable(들)에 의해 발행된 항목들을 계속 발행합니다.

라는 특징을 갖고 있습니다. 



```kotlin
class Variable<T> ... {
  ...
  private val behaviorSubject by lazy<BehaviorSubject<T>> { BehaviorSubject.create() }


  init {
    value?.let { behaviorSubject.onNext(it) }
  }

  @Synchronized
  fun get(): T? = value

  fun set() {
    try {
      val voidConstructor = Void::class.java.getDeclaredConstructor()
      voidConstructor.isAccessible = true
      val v = voidConstructor.newInstance()
      publishSubject.onNext(v as T)
    } catch (e: Exception) {
      WLog.e(e.printStackTrace())
    }

  }

  @Synchronized
  fun set(value: T) {
    this.value = value
    behaviorSubject.onNext(this.value)
  }
  ...
}
```

초기화 시 받아두었던 **Value**를 `BehaviorSubject`를 생성하여 onNext로 넣어줍니다.

만약 초기 값이 비어있는 경우를 대비하여 `set()` 함수를 만들어 Void 객체를 발행하여 방출하는 방식을 택했습니다.(Void 인스턴스를 발행한 이유는 RxJava에서 null 값을 스트림에 방출시키는 것이 좋지 않는 방법이며, RxJava2에서는 더 이상 사용되지 않기 때문입니다.)



`Variable`은 가장 최신 **Value**를 홀드하고 있기때문에, 언제든지 최신 값을 `get()` 메서드를 통해 꺼낼 수 있습니다.



그러면 이제 방출할 **Value**를 구독하여 관찰 하는 메서드를 보겠습니다.

```kotlin
class Variable<T> ... {
  ...
  fun asObservable(): Observable<T> = behaviorSubject.serialize()

  fun Observable<T>.subscribe(lifecycle: Lifecycle, action: (T) -> Unit, onError: ((Throwable) -> Unit)? = null): Subscription? {
    weakLifecycle = WeakReference(lifecycle).apply {
      get()?.addObserver(this@Variable)
    }
    subscription = this.subscribe({ action.invoke(it) }, { onError?.invoke(it) })
    this@Variable.action = action
    this@Variable.onError = onError
    return subscription
  }

  fun subscribe(lifecycle: Lifecycle, action: (T) -> Unit, onError: ((Throwable) -> Unit)? = null): Subscription? {
    weakLifecycle = WeakReference(lifecycle).apply {
      get()?.addObserver(this@Variable)
    }
    subscription = asObservable().subscribe({ action.invoke(it) }, { onError?.invoke(it) })
    this.action = action
    this.onError = onError
    return subscription
  }
}
```

`Variable`에서는 구독하는 메서드를 두가지로 두었습니다. 하나는 `Subject` 상태서 `Observable`로 컨버팅 후 구독을 하는 것과, 하나는 `Subject`에서 직접 `asObservable()` 메서드를 통해 `Observable`로 변환해 구성하는 방식입니다.



`LiveData`와 마찬가지로 관찰 시작 이후 Lifecycle을 받아 옵저버 등록을 하는 것을 볼 수 있습니다. 이를 통해 생명주기에 따른 콜백메서드로 구독관리가 가능하게 되었습니다.



최종적인 코드는 다음과 같습니다.



{% gist watcha-jimmy/5886d8443f927a13ff77b5df0f1f189c %}



## 최종적으로 실무에서의 경험

### Loading 만들기

```kotlin
/**
 * @author jimmy
 * @since 2019.10.25
 */
data class Loading(
  val lifecycleOwner: BaseActivity,
  val loadingState: LoadingState?= null,
  val loadingText: String? = null): Item() {
  val stateVariable = Variable(loadingState)
  fun setState(loadingState: LoadingState) {
    stateVariable.set(loadingState)
  }
}
enum class LoadingState(val loadingStateText: String) {
  PREPARE("prepare"),
  LOADING("loading"),
  COMPLETE("complete")
}
```



본인의 경우 이번에 플레이어를 새로 리뉴얼 하게 되면서 상태관리에 대한 개선을 필요로 하게되었고, 이를 데이터 홀더인 `Variable`를 사용하여 구현하게 되었습니다.



### 에피소드리스트의 로딩 상태관리하기

다음은 에피소드 리스트를 띄울 때 상태에 따라 로딩에 대한 State를 바꾸고, 바꿀 때 UI에 반영되는 로직의 일부입니다.

```kotlin
class ModalEpisodesListFragment : BaseFragment<Episodes>() ... {
  ...
  private val loading by lazy { Loading(baseActivity) }
  ...
  private fun setupEpisodes() {
    loading.setState(LoadingState.LOADING)
    loading_container.visibility = View.VISIBLE
    currDirection = Direction.NONE
    requestData(dataProvider)
  }
  ...
  private fun bindAdapter() {
    ... // 어댑터 바인딩
    loading.apply {
      stateVariable.subscribe(baseActivity.lifecycle, {
        (adapter as BasicRecyclerViewAdapter).apply {
          when (it) {
            ... // Loading State에 따른 비즈니스 로직 처리 
          }
        }
      })
    }
  }
}
```



전체적인 로직을 이해하기 위해 이해해야 할 것들이 많기 때문에, 어느정도 기본지식을 쌓은 상태로 해당 포스트를 보면 도움이 될 것이라 생각합니다.



**긴 글 읽어주셔서 감사합니다.**