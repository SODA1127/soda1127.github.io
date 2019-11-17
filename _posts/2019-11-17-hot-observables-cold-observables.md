---
layout: post
title: "3-02. 핫 옵저버블 & 콜드 옵저버블"
description: 행동방식에 따라 나뉘는 옵저버블의 종류에 대해 알아봅시다.
image: 'https://imgur.com/pRSbwbs.jpg'
category: 'programming'
date: 2019-09-17 11:18:00
tags:
- kotlin
- ReactiveX
- Reactive Programming
introduction: 행동방식에 따라 나뉘는 옵저버블의 종류에 대해 알아봅시다.
twitter_text: 행동방식에 따라 나뉘는 옵저버블의 종류에 대해 알아봅시다.
---

# 3-02. Hot & Cold Observables

이제 막 `Observables` 와 `Observer`에 대해 알게 되었을 것입니다.  그럼 이제는 좀 더 흥미로운 주제인 행동방식에 따라 나뉘는 `Observables` 의 종류를 알아보도록 하겠습니다.



## Cold Observable

```kotlin
fun main(args: Array<String>) {
    val observable: Observable<String> = listOf("String 1","String 2","String 3","String 4").toObservable()//1

    observable.subscribe({//2
        println("Received $it")
    },{
        println("Error ${it.message}")
    },{
        println("Done")
    })

    observable.subscribe({//3
        println("Received $it")
    },{
        println("Error ${it.message}")
    },{
        println("Done")
    })
}
```

위 예제를 보면, 같은 `Observable` 인스턴스에 대해 두 번 구독한 것을 볼 수 있습니다.



위 로직은 매우 간단합니다. (1)에서는 발행할 데이터에 대한 옵저버블을 선언하고, (2)와 (3)에서 구독을 한 것을 볼 수 있습니다.

이에대한 결과는 다음과 같습니다.



> Received String 1
>
> Received String 2
>
> Received String 3
>
> Received String 4
>
> Done
>
> Received String 1
>
> Received String 2
>
> Received String 3
>
> Received String 4
>
> Done

결과를 보면 매우 당연하게도 두번 구독이 이루어졌기 때문에 구독해 발행된 데이터가 두번 출력되는 것을 볼 수 있습니다. 그 중에서도 `Cold Observable` 은 각 구독마다 처음부터 데이터를 배출하게 됩니다. 다시 풀어서 설명하면, 콜드 옵저버블은  **구독시에**  데이터 발행을 시작하게 되는데, `subscribe()` 메서드 호출 시 각 데이터를 동일한 순서로 푸시한다는 것입니다.

이번 장까지 사용하는 모든 옵저버블 인스턴스는 콜드 옵저버블로 동작하고 있습니다. 콜드 옵저버블은 데이터작업에 특히 많이 쓰입니다. 예를 들어 안드로이드의 경우 `sqLite`나 `Room` 데이터베이스 작업을 하는 동안은 더욱 많이 의존을 하게 됩니다.



## Hot Observable

콜드 옵저버블이 수동적이며, `subscribe()` 메서드가 호출될 때까지 아무것도 내보내지 않는 것과 반대로, `Hot Observable`은 `emit`배출을 시작하기 위헤 구독을 할 필요가 없습니다.

해당 책에서는 비교를 이렇게 하였습니다.

`Cold Observable`이 CD/DVD 레코딩으로 본다면, `Hot Observable`은 TV 채널과 같이 시청자가 시청을 하고있는 것과 상관없이 컨텐츠를 `Broadcasting` 송출 한다는 것입니다.



핫 옵저버블은 데이터보다는 이벤트와 유사합니다. 물론 이벤트라는 것은 데이터를 포함할 수 있으나 시간의 흐름에 민감한 특징을 갖고 있습니다. 이러한 특징은 특히 안드로이드/iOS와 같이 UI이벤트에 상호작용이 중요한 플랫폼에 많은 유용성을 가져다줍니다. 그리고 서버 요청을 흉내낼 수 있습니다.



## Hot Observable - ConnectableObservable

`ConnectableObservable`은 유용한 핫 옵저버블이며, 좋은 예시입니다. 해당 옵저버블의 가장 큰 특징은 콜드 옵저버블을 핫 옵저버블로 변환이 가능하다는 것인데, 특징적으로 데이터 배출 시 `subscribe`() 메서드를 사용하기 앞서 `connect()` 메서드를 호출하여 핫한 옵저버블로 활성화하여 사용이 가능합니다. 다음 예시를 보겠습니다.



```kotlin
fun main(args: Array<String>) {
    val connectableObservable = listOf("String 1","String 2","String 3","String 4","String 5").toObservable()
            .publish()//1
    
    connectableObservable.
            subscribe { println("Subscription 1: $it") }//2
    
    connectableObservable.map(String::reversed)//3
            .subscribe { println("Subscription 2 $it")}//4
    
    connectableObservable.connect()//5

    connectableObservable.
            subscribe { println("Subscription 3: $it") } //6 구독을 받지못함
}
```

위 예시에서 기존 코드와 다르게 추가된 부분은 (1)의 `publish()` 라는 함수인데, 해당 메서드를 이용하여 콜드한 옵저버블은 핫한 옵저버블로 변환 한것을 볼 수 있습니다.

그리고 (5)를 보면 `connect()` 메서드를 호출하여 위 (2), (3) 옵저버에서 모두 데이터가 배출이 되는 것을 볼 수 있습니다. 

결과는 다음과 같습니다.

>Subscription 1: String 1
>
>Subscription 2 1 gnirtS
>
>Subscription 1: String 2
>
>Subscription 2 2 gnirtS
>
>Subscription 1: String 3
>
>Subscription 2 3 gnirtS
>
>Subscription 1: String 4
>
>Subscription 2 4 gnirtS
>
>Subscription 1: String 5
>
>Subscription 2 5 gnirtS

`ConnectableObservable` 의 가장 큰 목적은 한 옵저버블에 여러 구독자를 연결하여 하나의 푸시에 한꺼번에 대응을 하는 데 있습니다. 이는 푸시를 반복해 각 구독바다 푸시를 보내는 콜드 옵저버블과는 반대되는 성향을 보이고 있습니다. 

`ConnectableObservable`은 `connect()` 메서드 이전에 호출된 모든 subscriptions(옵저버)를 연결해 모든 옵저버에 단일 푸시를 전달하게 됩니다. 그러면 옵저버든 해당 푸시에 대응할 수 있게됩니다.



![Connected Observable](https://imgur.com/iGLYvRs.jpg)



이와같이 옵저버블에서 단 한번의 배출로 모든 구독자(관찰자)에게 배출을 전달하는 방식을 `Multicasting`**멀티캐스팅** 이라고합니다. Connected Observable에 대한 다른 예제를 보도록하겠습니다.

```kotlin
fun main(args: Array<String>) {
    val connectableObservable = Observable.interval(100,TimeUnit.MILLISECONDS)
            .publish()//1

    connectableObservable.
            subscribe { println("Subscription 1: $it") }//2

    connectableObservable
            .subscribe { println("Subscription 2 $it")}//3

    connectableObservable.connect()//4

    runBlocking { delay(500) }//5

    connectableObservable.
            subscribe { println("Subscription 3: $it") }//6

    runBlocking { delay(500) }//7
}
```

위 예제는 앞에서 보여드린 예제를 약간 수정한 것입니다.



위 예제에서는 100ms간격으로 데이터를 발행하는 옵저버블 객체를 생성했습니다. `interval()` 메서드를 사용하는 이유는 각 배출마다 간격이 생겨 (5)를 이용해 `connect()` 메서드로 핫 옵저버블 활성화 이후에 약간의 공간을 주기위함입니다.

(2), (3)에서 구독을 수행 후 (4)에서 핫 옵저버블로 변환 한 이후에 (5) 이후 (6)에서 세 번째 구독자가 두 구독자가 발행시작 500ms 이후 데이터를 배출하는 것을 볼 수 있습니다.

결과는 다음과 같습니다.

>Subscription 1: 0
>
>Subscription 2 0
>
>Subscription 1: 1
>
>Subscription 2 1
>
>Subscription 1: 2
>
>Subscription 2 2
>
>Subscription 1: 3
>
>Subscription 2 3
>
>Subscription 1: 4
>
>Subscription 2 4
>
>Subscription 1: 5
>
>Subscription 2 5
>
>Subscription 3: 5
>
>Subscription 1: 6
>
>Subscription 2 6
>
>Subscription 3: 6
>
>Subscription 1: 7
>
>Subscription 2 7
>
>Subscription 3: 7
>
>Subscription 1: 8
>
>Subscription 2 8
>
>Subscription 3: 8
>
>Subscription 1: 9
>
>Subscription 2 9
>
>Subscription 3: 9



다음장에선 `Hot Observable` 을 사용하는 데 있어 유용하고 좋은 방법인 `Subject`에 대해 알아보고, 더 나아가 다양한 `Subject`의 유형을 알아보는 것으로 하겠습니다.