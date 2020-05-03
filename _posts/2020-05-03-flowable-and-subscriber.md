---
layout: post
title: "4-01.플로어블과 구독자"
description: 플로어블의 특징과, 왜 구독자를 사용하는지 알아봅시다.
image: 'https://imgur.com/0fRT8te.jpg'
category: 'programming'
date: 2020-05-03 23:47:32
tags:
- kotlin
- ReactiveX
- Reactive Programming
introduction: 플로어블의 특징과, 왜 구독자를 사용하는지 알아봅시다.
twitter_text: 플로어블의 특징과, 왜 구독자를 사용하는지 알아봅시다.

---

# `Flowable`플로어블과 `Subscriber`구독자

`Flowable`플로어블은 `Observable`과 다르게 `BackPressure`배압 호환이 가능한 구독자를 사용합니다. 다만 람다식을 사용하는 경우, 큰 차이점은 없습니다. 그러면, 왜 옵저버 대신 구독자를 사용하는 것일까요?



구독자는 옵저버의 일부분의 기능과, 백프레셔를 동시에 지원합니다. 그 중 가장 큰 차이는 **Buffer**의 유무인데, 버퍼를 통해 얼마나 많은 아이템을 받기를 원하는지 설정이 가능하며, 만약 아무 값도 지정하지 않으면 어떤 배출도 수신하지 못할 것입니다.



앞에서 설명했듯이, 람다식을 사용한 구독자는 옵저버를 사용한 것과 유사한 코드를 가집니다. 한번 코드로 보도록 하겠습니다.

```kotlin
fun main(args: Array<String>) {
    Flowable.range(1, 15)
            .map { MyItem6(it) }
            .observeOn(Schedulers.io())
            .subscribe(object : Subscriber<MyItem6> {
                lateinit var subscription: Subscription//(1)
                override fun onSubscribe(subscription: Subscription) {
                    this.subscription = subscription
                    subscription.request(5)//(2)
                }

                override fun onNext(s: MyItem6?) {
                    runBlocking { delay(50) }
                    println("Subscriber received " + s!!)
                    if(s.id == 5) {//(3)
                        println("Requesting two more")
                        subscription.request(2)//(4)
                    }
                }

                override fun onError(e: Throwable) {
                    e.printStackTrace()
                }

                override fun onComplete() {
                    println("Done!")
                }
            })
    runBlocking { delay(10000) }
}

data class MyItem6 (val id:Int) {
    init {
        println("MyItem Created $id")
    }
}
```

코드를 함께 같이 이해해보도록 하겠습니다.

이전 코드와 비교해보면 주석2번까지는 코드가 동일합니다. 하지만. 구독자를 사용함으로써 가져가는 이점인 원하는 배출량을 설정하는 부분에서 코드를 다르게 두었습니다.



`Subscriber`구독자를 사용 시 구독 시 `onSubscribe(Subscribe)` 컬백 메서드를 보면 `Subscription` 인스턴스를 받는 것을 볼 수 있습니다. 해당 코드의 주석 2번을 보면 `request()`메서드가 호출이 된 것을 볼 수가 있는데, `request()` 메서드는 구독자가 호출되고나서 업스트림에서 대기해야하는 배출량을 설정할 수 있습니다.

따라서, 구독자가 더 요청을 하지 않는 이상, 요청이후의 더 이상의 배출은 무시를 하게됩니다.



결과적으로, 15개의 아이템은 생성은 되었으나, 50ms동안 아무런 데이터도 배출이 되지 않는 상황이었을 것입니다. 하지만 이전에 5개의 데이터를 업스트림에 담아두었기 떄문에 하나하나씩 배출을 하여 id갑시 5번인 데이터까지 호출이 된 이후, 5번 때 2개의 아이템을 업스트림에 더 요청해 출력하게 됩니다.

> MyItem Created 1
> MyItem Created 2
> MyItem Created 3
> MyItem Created 4
> MyItem Created 5
> MyItem Created 6
> MyItem Created 7
> MyItem Created 8
> MyItem Created 9
> MyItem Created 10
> MyItem Created 11
> MyItem Created 12
> MyItem Created 13
> MyItem Created 14
> MyItem Created 15
> Subscriber received MyItem6(id=1)
> Subscriber received MyItem6(id=2)
> Subscriber received MyItem6(id=3)
> Subscriber received MyItem6(id=4)
> Subscriber received MyItem6(id=5)
> Requesting two more
> Subscriber received MyItem6(id=6)
> Subscriber received MyItem6(id=7)



조금은 `Flowable`과 `Subscriber`에 대해 이해하게 된것 같으니, 다시 기초로 돌아가보도록 하겠습니다.



## 처음부터 `Flowable`플로어블 생성해보기

```kotlin
fun main(args: Array<String>) {
    val observer: Observer<Int> = object : Observer<Int> {
        override fun onComplete() {
            println("All Completed")
        }

        override fun onNext(item: Int) {
            println("Next $item")
        }

        override fun onError(e: Throwable) {
            println("Error Occured ${e.message}")
        }

        override fun onSubscribe(d: Disposable) {
            println("New Subscription ")
        }
    } // Observer 생성

    val observable: Observable<Int> = Observable.create<Int> {//1
        for(i in 1..10) {
            it.onNext(i)
        }
        it.onComplete()
    }

    observable.subscribe(observer)

}
```

위 코드는 매우 간단한 예제입니다. 먼저, `Observable.create()` 오퍼레이터로 옵저버블 객체를 생성하고, 1 ~ 10까지의 데이터 아이템을 배출시킵니다. 결과는 다음과 같습니다.

> New Subscription 
> Next 1
> Next 2
> Next 3
> Next 4
> Next 5
> Next 6
> Next 7
> Next 8
> Next 9
> Next 10
> All Completed



이번엔 옵저버블 객체 생성을 플로어블 객체 생성으로 바꿔보도록 하곘습니다.

```kotlin
fun main(args: Array<String>) {
    val subscriber: Subscriber<Int> = object : Subscriber<Int> {
        override fun onComplete() {
            println("All Completed")
        }

        override fun onNext(item: Int) {
            println("Next $item")
        }

        override fun onError(e: Throwable) {
            println("Error Occured ${e.message}")
        }

        override fun onSubscribe(subscription: Subscription) {
            println("New Subscription ")
            subscription.request(10)
        }
    }//(1)

    val flowable: Flowable<Int> = Flowable.create<Int> ({
        for(i in 1..10) {
            it.onNext(i)
        }
        it.onComplete()
    },BackpressureStrategy.BUFFER)//(2)

    flowable
            .observeOn(Schedulers.io())
            .subscribe(subscriber)//(3)

    runBlocking { delay(10000) }

}
```

주석 1에서는 구독자의 인스턴스를 생성했습니다. 그 다음 주석 2에서는 `Flowable.create()` 메서드로 플로어블의 인스턴스를 생성하고, 주석3에서 구독자를 구독하게 했습니다. 

여기 서 유심 히 볼 것은 주석 2의 `BackpressureStrategy.BUFFER` 인자를 전달하고 있는 것을 볼 수가 있습니다. 이 옵션이 어떤 것을 의미하는지 보도록 하겠습니다.



다음은 Flowable.create() 메서드의 정의입니다.

![Flowable create](https://i.imgur.com/5frmYTN.png)

첫 번째 매개변수는 배출에 원천이 되는 `FlowableOnSubscribe`, 두 번째는 `BackPressureStrategy` 인것을 볼 수 있습니다.  `BackPressureStrategy`는 Enum타입으로 이뤄져 있는데, 기본적으로 다섯가지의 옵션을 제공합니다.



![BackpressureStrategy](https://imgur.com/xUAr0PT.jpg)



![buffer strategy](https://i.imgur.com/0fRT8te.jpg)

- `BackPressureStrategy.MISSING`
  - 백프레셔 구현을 사용하지 않으며, 다운스트림이 오버플로우를 직접 처리해야합니다. 이 옵션은 `onBackpressureXXX()` 오퍼레이터를 커스텀하여 다룰 때 유용한데, 추후에 관련된 예제를 보도록 하겠습니다.
- `BackPressureStrategy.ERROR`
  - 어떤 백프레셔로도 구현하지 않는데, 다운스트림이 소스를 따라잡지 못하는 경우에 `MssingBackpressureException`을 발생시킵니다.
- `BackPressureStrategy.BUFFER`
  - 다운스트림이 배출을 소비할 수 있도록 제한이 없는 버퍼에 저장을 합니다. 물론 버퍼 크기는 지정이 가능하며, 버퍼크기를 넘어서는 경우 `OOM`OutofMemoryError가 발생할 수 있습니다.
- `BackPressureStrategy.DROP`
  - 말 그대로, 다운스트림이 데이터 배출량에 비해 속도를 따라갈 수 없을 때 배출하는 데이터를 무시하고, 드롭합니다.
  - 예를들어, 소스에서 1, 2, 3, 4, 5의 데이터를 배출하고 있는 상황에서 1을 받은 이후에 처리하느라 바쁜 와중에 2, 3, 4데이터를 소스에서 배출했으며, 5가 배출되기 이전에 다운스트림이 처리할 준비가 되었다면, 5를 제외한 나머지 데이터는 드롭시킵니다.
- `BackPressureStrategy.LATEST`
  - 다운스트림이 바쁘고, 배출을 유지할 수 없는경우, 최신 배출량만을 유지하고, 나머지는 모두 무시하는 전략입니다.
  - 다운스트림이 이전작업을 마치면, 작업이 끝나기전에 마지막으로 배출된 데이터를 받습니다.
  - 예를들어, 소스에서 1, 2, 3, 4, 5의 데이터를 배출하고 있는 상황에서 1을 받은 이후에 처리하는 와중에 소스에서는 2, 3, 4를 배출했고, 5가 배출되기 바로 전 처리할 준비가 되었다면 다운스트림에서는 4, 5 두가지의 값을 받게됩니다. 하지만, 4를 받으면서 처리가 바빠지는 경우, 5를 수신할 수 없게됩니다.



## 옵저버블로 플로어블 만들기

`Observable.toFlowable()`  오퍼레이터는 옵저버블 객체와 같이 백프레셔를 지원하지 않는 소스에서 `BackPressureStrategy` 를 구현하는 방법을 알려줍니다. 예제 코드를 보겠습니다.

```kotlin
fun main(args: Array<String>) {
    val source = Observable.range(1, 1000) //(1)
    source.toFlowable(BackpressureStrategy.BUFFER) //(2)
            .map { MyItem7(it) }
            .observeOn(Schedulers.computation())
            .subscribe{ //(3)
                print("Rec. $it;\t")
                runBlocking { delay(600) }
            }
    runBlocking { delay(700000) }
}

data class MyItem7 (val id:Int) {
    init {
        print("MyItem7 init $id;\t")
    }
}
```

주석 1에서 `Observable.range()` 메서드로 옵저버블을 생성해주고, 주석2에서 `Observable.toFlowable()` 메서드로 버퍼를 갖는 플로어블로 변환한 것을 볼 수 있습니다. 그리고, 주석 3에서 람다식을 사용해 구독을 등록한 것을 볼 수 있습니다.



결과적으로, 다운스트림이 소비될 때까지 버퍼가 모든 배출량을 버퍼에 저장하기 떄문에 모든 배출량을 처리가 가능한 것을 알 수 있습니다.



그러면, Strategy를 `BackPressureStrategy.ERROR`로 바꿔보도록 하겠습니다.

결과적으로 다운스트림이 업스트림을 따라갈 수가 없어 다음과 같은 에러가 발생됩니다.

> Caused by: io.reactivex.exceptions.MissingBackpressureException: could not emit value due to lack of requests



그러면, Strategy를 `BackPressureStrategy.DROP`로 바꿔보도록 하겠습니다.

아이템을 초기화 한과는 다르게, 다운스트림에서 데이터를 다 처리하지 못해 기본 버퍼사이즈인 128 이상은 나오지 않는 것을 볼 수 있습니다.

>MyItem Created 1
>MyItem Created 2
>MyItem Created 3
>MyItem Created 4
>MyItem10(id=1)
>MyItem Created 5
>MyItem Created 6
>...
>
>MyItem Created 127
>MyItem Created 128
>MyItem10(id=2)
>MyItem10(id=3)
>MyItem10(id=4)
>
>...



다음시간엔 `BackPressureStrategy.MISSING`에 대하여, `onBackpressureXXX()`와 같은오퍼레이터를 이용하여 BUFFER Strategy와 유사한 방법을 구현할지 고민해보도록 하겠습니다.