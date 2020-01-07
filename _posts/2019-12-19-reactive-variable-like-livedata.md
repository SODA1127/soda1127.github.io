---
layout: post
title: "RxJava로 LiveData 따라해보기 1"
description: Android Jetpack에서 제공하는 LiveData처럼 리액티브한 데이터 홀더 클래스를 만들어 봅시다.
image: 'https://imgur.com/2gM2T8c.jpg'
category: 'programming'
date: 2020-01-02 11:36:24
tags:
- kotlin
- idea
- android-jetpack
- android-architecture-component
- android-lifecycle
- rxjava
introduction: Reactive한 방법으로 LiveData와 같은 기능을 하는 데이터 홀더를 만들어 봅시다.
twitter_text: Reactive한 방법으로 LiveData와 같은 기능을 하는 데이터 홀더를 만들어 봅시다.

---

*본 내용은 **Android Developers** 공식 문서에서 발췌한 내용을 기반으로 합니다.*

# Android Jetpack - LiveData

최근에 많은 안드로이드 어플리케이션이 `Android Jetpack` 라이브러리를 기반으로 좀 더 쉽고 빠르게, 보일러 플레이트 등의 코드를 줄이는 목적으로 제작이 되고 있습니다.  [Android Developers 공식 문서](https://developer.android.com/jetpack?hl=ko)에서는 `Android Jetpack` 라이브러리를 이렇게 소개하고 있습니다.



### `Android Jetpack`

> Jetpack은 개발자가 고품질 앱을 손쉽게 개발할 수 있게 돕는 라이브러리, 도구, 가이드 모음입니다. 이 구성요소를 통해 권장사항을 따르고, 상용구 코드 작성에서 벗어나며, 복잡한 작업을 간소화함으로써 중요한 코드에만 집중할 수 있습니다.

제공하는 요소들은 다음과 같습니다.

![Android Jetpack 구성요소](https://imgur.com/zk3uyTI.jpg)



## 그래서 **LiveData**가 무엇인가요?

**LiveData**라는 이름만 봐서는 굉장히 역동적인 이름을 갖고 있음을 느낍니다. 직역해 보면 `살아있는 데이터`로 번역해 볼 수 있겠죠. **LiveData**는 앞에서 설명했듯이 옵저버 패턴을 따르는 데이터 홀더 클래스입니다. 그말인 즉, 옵저버 패턴을 기반으로 홀딩 된 데이터의 Value가 바뀔 때마다 콜백메서드로 변경된 Value를 받게 됩니다.

**LiveData**라는 이름에 걸맞게 VC(Activity, Fragment, Service 등)의 라이프사이클에 따라 수명주기를 인식하여 활성 수명 주기 상태에 있는 앱 구성요소만을 업데이트 하게됩니다.

이 덕분에 옵저버 패턴을 이용함에도 자동으로 생명주기에 따른 구독관리를 해주는 기능이 존재합니다.

**LiveData**의 사용 이점은 다음과 같습니다.

- **UI와 데이터 상태의 일치 보장**
- **메모리 누출 없음**
- **중지된 활동으로 인한 비정상 종료 없음**
- **수명 주기를 더 이상 수동으로 처리할 필요 없음**
- **항상 최신 데이터 유지**
- **적절한 구성 변경**
- **리소스 공유**



## LiveData를 써야하는 이유, 그리고 아쉬운 부분

LiveData는 아까도 설명했듯이 LifeCycle에 따른 데이터흐름을 가지게 됩니다. 옵저버 패턴을 갖고있는 덕에 값을 콜백으로 받기 쉬운 구조로 이루어져 있습니다. 하지만, LifeCycle의 존재를 알기때문에(by LifecycleObserver) 필요하지 않을 때, 즉 VC의 라이프사이클이 비활성화 되어 있는 상태에서는 콜백을 전혀 호출하지 않습니다.



![해리의 유목코딩 LiveData 프로세스](https://miro.medium.com/max/1342/1*kjSdZieZlUhixRTjX0bMew.png)





![일반적인 LiveData를 사용하는 아키텍쳐](https://img1.daumcdn.net/thumb/R1280x0/?scode=mtistory2&fname=https%3A%2F%2Fk.kakaocdn.net%2Fdn%2FbwV3is%2FbtqyRMARlZI%2F2NRgjLBo06w5aR9ZFYaRBk%2Fimg.png)



일반적인 LiveData를 사용할 때 취하는 아키텍쳐입니다. 안드로이드 Jetpack에서 보이는 AAC-VM 패턴의 경우 위 구조를 권장하고 있습니다.

옵저버 패턴의 장점답게, 역시 어디서든지 연결이 되어 있기때문에 스트림에 데이터를 흘려보내주기만 한다면, 자동으로 갱신이 가능한 장점이라는 것인데요. 단점은 RxJava와 비교해 보았을 때, 터무늬 없이 적은 연산자를 제공한다는 것입니다. 또한, 스레드를 관리하기위한(Dispatcher와 같은) 기능을 제공하지 않습니다.

(물론 구글갓께서 그런 점을 인지하고 있어 Corotine과 함께 사용이 가능한 `Flow`라는 라이브러리를 제공합니다만.. 아직까지 실험버전인 게 흠입니다.)

Flow에 대해 아주 잘 정리해 놓은 블로그 포스트가 있습니다. 아래 글을 참고하여 보시면 더더욱 좋습니다.

[[Kotlin] 코틀린 - 코루틴 Asynchronous Flow(1/2)](https://tourspace.tistory.com/258)

[[Kotlin] 코틀린 - 코루틴 Asynchronous Flow(2/2)](https://tourspace.tistory.com/260?category=797357)



Flow의 경우 오직 RxJava의 `Observable` 와 같이 Cold한 Observable을로 변환이 가능한 형태로 이루어져 있어 구독(`collect()`) 를 해야만 구독자가 배출 된 데이터를 받아볼 수 있습니다.

당연하게도 실무에서도 마찬가지로 LiveData의 장점을 인지하고 있고, 앞으로 나올 다양한 연산자들과 RxJava 라이브러리의 대항마로 공식적으로 곧 선보일 코루틴의 Flow가 서서히 RxJava를 넘어 대세가 될 것이라는 예감을 갖고있습니다.



하지만 여전히 RxJava는 무수히 많은 연산자들, 스트림에 대한 다양한 커스텀, 스레드 변경의 용이함들을 갖추고 있기때문에 **이 둘의 장점을 잘 섞어보면 어떨까라는 생각을 하게 되었습니다.**



# 본격적으로 LiveData 따라해보기

## 상태주기 관리하는 로직 구성해보기

먼저 LiveData의 가장 핵심적인 구성요소, 상태에 따라 활성화 여부, 구독에 대한 관리가 가능한 로직을 구성해보겠습니다.

> 포스트에서 보여주는 과정은 RxJava 라이브러리를 사용하여 LiveData의 특징들을 하나하나 되짚어 보면서 코드를 구성해 나가고 있습니다.
>
> 아래 모든 코드들은 실무에 적용 된 코드임을 참고해주세요.



먼저 라이프사이클에 의존적인 자동으로 구독해제가 가능한 코드를 만들어 보도록 하겠습니다.

아래 코드는 RxJava의 Observable 클래스를 구독하기 시작하여 받게 된 `Subscription` 클래스에 대한 구독관리 로직입니다.

```kotlin
class AutoClearedSubscription
@JvmOverloads
constructor(
    private val lifecycleOwner: BaseActivity,
    private val alwaysClearOnStop: Boolean = false,
    private val compositeSubscription: CompositeSubscription = CompositeSubscription()
) : LifecycleObserver
```



코드를 보면 생성시 기본적으로 LifecycleOwner VC인 Activity를 받고 있습니다. (사실 Activity보다는 loose coupling한 방식으로 **AAC**(Android Architecture Component)의 LifecycleObserver를 갖는 게 더 좋은 방법이라 생각합니다.)

그리고 앞으로 VC의 Lifecycle에 따라 구독을 clear할 것인지를 정하는 조건변수를 주었습니다. 그리고, 생성시 가장 중요한 부분인  `Subscription` 을 담는 주머니인 `CompositeSubscription` 을 생성하여 구독에 대한 관리를 할 것을 암시하고 있습니다.



구독을 왜 해제, 비우는지 이해가 가지 않는 분들은 이전에 제가 작성한 포스트 중 [3-01.옵저버블을 구독했을 때 해제하는 방법](https://soda1127.github.io/observables-dispose/)글을 보시면 좋을 것 같습니다.



그러면 먼저 스트림을 구독하여 반환 된  `Subscription` 을 어떻게 추가할지 고민해보겠습니다.

```kotlin
class AutoClearedSubscription
... : LifecycleObserver {
  ...
  fun Subscription.add() {
    if(lifecycleOwner.lifecycle.currentState.isAtLeast(Lifecycle.State.INITIALIZED)) {
      compositeSubscription.add(this)
    }
  }

  operator fun plusAssign(subscription: Subscription) = subscription.add()
  ...
}
```

코틀린으로 구성된 프로젝트라는 것을 전제하에, 저는 두 가지의 방식으로 구독을 추가해 볼것을 생각했습니다. 코틀린에서는 확장합수와 연산자 오버로딩을 제공합니다. 그래서 이 두 방식을 사용해 Subscription을 추가하여 compositeSubscription 주머니에 담는 것을 볼 수 있습니다.



구독객체를 담았으니, 이제 Lifecycle 상태에 따라 어떤식으로 구독관리가 이뤄지는지 보겠습니다.

```kotlin
class AutoClearedSubscription
... : LifecycleObserver {
  
  @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
  fun cleanUp() {
    if (!alwaysClearOnStop) return
    compositeSubscription.clear()
  }

  @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
  fun detachSelf() {
    compositeSubscription.clear()
    lifecycleOwner.lifecycle.removeObserver(this)
  }
  
  operator fun Lifecycle.plusAssign(observer: LifecycleObserver) = this.addObserver(observer)
  ...
}
```

`AAC`, `AndroidX`가 나오기 전에는 보지못했을 어노테이션인 `@OnLifecycleEvent` 입니다.

[`androidx.lifecycle`](https://developer.android.com/reference/androidx/lifecycle/package-summary.html) 패키지는 이러한 문제를 탄력적이고 단독적인 방법으로 처리하는 데 도움이 되는 클래스와 인터페이스를 제공합니다.



![Android Lifecycle](https://i.imgur.com/QVknZfM.jpg)



위 코드와 같이 observer class를 등록하면 lifecycle에 따라 해당 동작을 확인할 수 있습니다.



그러면, 이제 만들어본 구독관리 클래스를 어떻게 사용하는지 보겠습니다.

```java
public abstract class BaseActivity ... {
	  ...
    // 앞으로 Rx Observable 사용 시 구독 관리하도록
    protected AutoClearedSubscription compositeSubscription;
  	...
    @Override
    protected void onCreate(Bundle savedInstanceState) {
    	super.onCreate(savedInstanceState);		     
      // 생성 후 라이프사이클에 옵저버 바인딩(구독관리)
      compositeSubscription = new AutoClearedSubscription(this);
      getLifecycle().addObserver(compositeSubscription);
	    ...
  }
  ...
  public AutoClearedSubscription getCompositeSubscription() {
     return compositeSubscription;
  }
  ...
}
```



BaseActivity에서는 **compositeSubscription**을 **`AutoClearedSubscription`**으로  초기화 한 것을 볼 수 있습니다. 위 코드를 보면  **`AutoClearedSubscription`**가 `CompositeSubscription` 을 상속받은 클래스인 것을 떠올릴 수 있습니다.

그리고 BaseActivity의 Lifecycle을 기준으로 각 VC, 로직에서 옵저버 객체를 구독하는 모든 로직들은 자동해제를 위해 `getCompositeSubscription()`메서드로 꺼내진 주머니에 담는 것을 해야하는 것을 알 수 있습니다.

```java
Subscription subscription = builder.build().request(response -> finishRating(response, mCode));
compositeSubscription.add(subscription);
```



이렇게 VC의 Lifecycle의 주기 시점을 알아채 자동으로 해제가 가능한 로직을 구성해보는 시간을 가져봤습니다. 다음시간에는 **생명주기에 따른 활성화 여부 로직**, LiveData의 핵심인 **항상 최신 데이터 유지**, **옵저버 패턴을 통한 값 갱신 로직**을 구성해보도록 하겠습니다.



[>> 포스트 RxJava로 LiveData 따라해보기 2](https://soda1127.github.io/reactive-variable-like-livedata-2/)