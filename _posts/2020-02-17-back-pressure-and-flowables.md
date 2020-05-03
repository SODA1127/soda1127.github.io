---
layout: post
title: "4-00.백프레셔와 플로어블"
description: 백프레셔가 언제, 어떤 이유로 발생하는지, 어떻게 해결할지 알아봅시다.
image: 'https://imgur.com/0fRT8te.jpg'
category: 'programming'
date: 2020-02-17 15:30:30
tags:
- kotlin
- ReactiveX
- Reactive Programming
introduction: 백프레셔가 언제, 어떤 이유로 발생하는지, 어떻게 해결할지 알아봅시다.
twitter_text: 백프레셔가 언제, 어떤 이유로 발생하는지, 어떻게 해결할지 알아봅시다.
---

# `BackPressure`배압

3장까지는 푸시기반의 리액티브 프로그래밍에 대해 접해보았습니다. 지금까지 공부한 것으로는 옵저버블이 무엇인지, 그리고 옵저버블을 통해 어떠한 스트림을 구성이 가능한지 고민해볼 수 있었습니다.

옵저버블에서 배출한 데이터를 옵저버에 데이터를 전달하는 것까지는 문제가 없는데, 우리는 더 나아가서 생각을 할 필요가 있습니다. 만약 스트림에서 배출되는 데이터의 양이 옵저버가 소비할 수 있는 양보다 많게되면 어떤 현상이 발생하게 될까요?

옵저버블은 기본적으로 데이터를 동기적으로 하나씩 옵저버에 하나씩 푸시하여 동작합니다. 하지만, 옵저버가 받은 데이터를 하나씩 처리하는 과정에서 속도가 현저히 차이가 나게됩니다.

이를 **`BackPressure`배압현상**이라고 합니다.



잘 이해가 되지 않을 수 있으니 예제를 함께 보도록 하겠습니다.

```kotlin
fun main(args: Array<String>) {  
  val observable = Observable.just(1,2,3,4,5,6,7,8,9)//(1)
  val subject = BehaviorSubject.create<Int>()
  subject.observeOn(Schedulers.computation())//(2)
  .subscribe {//(3)
    println("Subs 1 Received $it")
    runBlocking { delay(200) }//(4)
  }

  subject.observeOn(Schedulers.computation())//(5)
  .subscribe {//(6)
    println("Subs 2 Received $it")
  }
  observable.subscribe(subject)//(7)
  runBlocking { delay(2000) }//(8)
  kot
```

코드는 매우 간단하게 구성되어 있습니다.

주석1에서 옵저버블 객체를 생성하고, 주석2에서 `BehaviorSubject`를 생성하여 3과 6에서 이를 구독합니다. 추가적으로 주석7에서 해당 Subject를 사용하여 옵저버블 객체를 구독하게되면, 배출된 데이터를 Subject의 옵저버를 통해 받을 수 있습니다.

주석4에서 `runBlocking` 메서드로 200ms를 준 것을 볼 수 있는데, 의도적으로 구독내에서 시간이 오래걸리는 것을 표현하였습니다.

여기서 처음 보게되는 코드인 `subject.observeOn(Scheduler)`가 보이는데, 지금은 자세하게는 몰라도 되지만, 구독중에 실행할 스레드를 명시를 한다고 보시면 됩니다. 그중, `Schedulers.computation()`으로 계산을 수행할 스레드로 구성한다고 보면 되겠습니다.

결과는 다음과 같습니다.

>Subs 1 Received 1
>Subs 2 Received 1
>Subs 2 Received 2
>Subs 2 Received 3
>Subs 2 Received 4
>Subs 2 Received 5
>Subs 2 Received 6
>Subs 2 Received 7
>Subs 2 Received 8
>Subs 2 Received 9
>Subs 1 Received 2
>Subs 1 Received 3
>Subs 1 Received 4
>Subs 1 Received 5
>Subs 1 Received 6
>Subs 1 Received 7
>Subs 1 Received 8
>Subs 1 Received 9

결과를 보았듯이, 이는 예상치 못한 결과로 내비칠 수 있습니다. 예상대로라면, 핫 옵저버블의 행동처럼 옵저버에 번갈아 가며 값을 전달해야 했을것입니다. 하지만, 그렇지 않은 것을 볼 수 있습니다.

그럴 수 있는 이유는 마치 계산이 오래 걸리는 작업처럼 표현을 했기 때문입니다.

주석4와 같이 delay를 200ms 주었기 때문에 첫번째 데이터를 배출받은 이후 오랜 시간 처리가 걸려 대기열로 들어가게되어 데이터를 두번째 subject에서 데이터가 다 배출되어 받아본 이후에 받아보게 된 것입니다.

하지만 해당방식을 이용하게 되면 `OutOfMemory`가 발생할 수 있으니 별로 좋은 방법은 아닙니다. 이번에는 다른 예를 보겠습니다.

```kotlin
fun main(args: Array<String>) {
  val observable = Observable.just(1,2,3,4,5,6,7,8,9)//(1)
  observable
  .map { MyItem(it) }//(2)
  .observeOn(Schedulers.computation())//(3)
  .subscribe {//(4)
    println("Received $it")
    runBlocking { delay(200) }//(5)
  }
  runBlocking { delay(2000) }//(6)
}

data class MyItem (val id:Int) {
  init {
    println("MyItem Created $id")//(7)
  }
}
```

이번에는 `map(Function<T, R>)` 이 들어간 것을 볼 수 있습니다. 위에서 말한 `BackPressure`배압현상을 잘표현할 수 있는 것이라고 생각합니다. `map` 함수의 용도는 데이터의 흐름을 추적하기 위함인데, MyItem으로 객체를 생성 시 초기화 때 id를 받아 어느시점에 생성되어 어느시점에 옵저버가 데이터를 전달받는지를 알 수 있게 하기 위합니다.

결과는 다음과 같습니다.

>MyItem Created 1
>MyItem Created 2
>MyItem Created 3
>MyItem Created 4
>MyItem Created 5
>MyItem Created 6
>MyItem Created 7
>MyItem Created 8
>MyItem Created 9
>Received MyItem(id=1)
>Received MyItem(id=2)
>Received MyItem(id=3)
>Received MyItem(id=4)
>Received MyItem(id=5)
>Received MyItem(id=6)
>Received MyItem(id=7)
>Received MyItem(id=8)
>Received MyItem(id=9)

예상했듯이, 데이터를 생성했음에도 불구하고 배출할 데이터들이 대기열에 쌓여져 있고, `Consumer` 소비자는 구독을 했음에도 불구하고 이전 데이터의 처리때문에 받아보지 못하고 있는 상황입니다.



이 문제의 해결책은 `Consumer`소비자와 `Producer`생산자 간 서로 피드백이 오갈 수 있는 채널일 것입니다. 다시 한번 쉽게 설명하면, 소비자가

> "이전 배출에 대한 소비가 완료가 될때까지 기다려줘"

라고 얘기하는 것과 같습니다.



RxJava에서는 옵저버 패턴으로 이루어져 있기때문에 해당 방식이 구현이 불가능했지만, RxJava2에서는 컨슈머패턴으로 이를 지원합니다. 해당 방식은 이전부터 있어온 방식이기 때문에, 이에대해 자세히 읽어보고 싶으시다면, [프로듀서-컨슈머패턴(Producer-Consumer Pattern)](https://byplacebo.tistory.com/18)을 참고하시기 바랍니다.

또한, 옵저버블이 RxJava와 **RxJava2**가 어떻게 달라졌는지에 대해서는 [ReactiveX - Observable and Flowable](https://github.com/ReactiveX/RxJava/wiki/What's-different-in-2.0#observable-and-flowable)를 참고하시기 바랍니다.

## `Flowable`플로어블

우선, RxJava에서 제공하던 방식으로 옵저버블과 옵저버는 백프레셔를 지원하지 않습니다. `Flowable`플로어블은 백프레셔를 지원하는 옵저버블입니다. Flowable은 연산자를 위해 최대 128개의 버퍼를 제공하는데, 컨슈머가 시간이 걸리는 작업이 존재할 때 버퍼에 담아 대기가 가능합니다.

```java
public abstract class Flowable<T> implements Publisher<T> {
  /** The default buffer size. */
  static final int BUFFER_SIZE;
  static {
    BUFFER_SIZE = Math.max(16, Integer.getInteger("rx2.buffer-size", 128));
  }
  ...
}
```



그러면, 다시 코드로 돌아와 보겠습니다. 본격적으로 `Observable`과 `Flowable`의 비교를 위해 에제를 들어보겠습니다.

```kotlin
fun main(args: Array<String>) {
  Observable.range(1,1000)//(1)
  .map { MyItem3(it) }//(2)
  .observeOn(Schedulers.computation())
  .subscribe({//(3)
    print("Received $it;\n")
    runBlocking { delay(50) }//(4)
  },{it.printStackTrace()})
  runBlocking { delay(80000) }//(5)
}


data class MyItem3 (val id:Int) {
  init {
    print("MyItem Created $id;\n")
  }
}
```

옵저버블을 1부터 1000까지 생성해 `map()` 함수로 MyItem3 인스턴스로 가공하여 배출하는 간단한 코드입니다. 이전 예제에서와 같이 구독자가 오래 걸리는 연산에 대한 처리를 흉내내기 위해 delay를 50ms만큼 주었습니다. 결과는 다음과 같습니다.

>MyItem Created 1;
>MyItem Created 2;
>MyItem Created 3;
>...
>Received MyItem3(id=1);
>MyItem Created 17;
>
>...
>MyItem Created 998;
>MyItem Created 999;
>MyItem Created 1000;
>Received MyItem3(id=2);
>Received MyItem3(id=3);
>...
>Received MyItem3(id=998);
>Received MyItem3(id=999);
>Received MyItem3(id=1000);

결과를 보면 옵저버블이 계속 데이터를 배출해 내는것을 볼 수 있습니다. 그에 반해 옵저버는 그 사이 한개의 데이터에 대해 처리를 할 수 있었습니다. 이후 모든 배출이 완료된 이후, 모든 데이터에 대한 처리가 이루어진 것을 볼 수 있습니다.

아까 언급헀듯이 해당방식으로 구성하게 되면 `OOM(OutOfMemory)`와 더불어 많은 문제가 발생을 하게 되는데, 이번에는 `Flowable`을 이용해 어떤 부분이 달라지는지 보겠습니다.

```kotlin
fun main(args: Array<String>) {
  Flowable.range(1,1000)//(1) Observable -> Flowable
  .map { MyItem4(it) }//(2)
  .observeOn(Schedulers.computation())
  .subscribe({//(3)
    print("Received $it;\n")
    runBlocking { delay(50) }//(4)
  },{it.printStackTrace()})
  runBlocking { delay(70000) }//(5)
}


data class MyItem4 (val id:Int) {
  init {
    print("MyItem Created $id;\n")
  }
}
```

결과는 다음과 같습니다.

>MyItem Created 1;
>MyItem Created 2;
>MyItem Created 3;
>...
>Received MyItem4(id=1);
>MyItem Created 8;
>
>...
>MyItem Created 128;
>Received MyItem3(id=2);
>Received MyItem3(id=3);
>...
>MyItem Created 129;
>... // 128 버퍼 크기를 기준으로 옵저버블에서 생성된 데이터 갖고 있음,

이런 동작을 통해 `OOM`문제를 충분히 해결 가능하다고 볼 수 있습니다.



## 그럼 Flowable은 언제 사용하는 것이 좋을까요?

Flowable이 상황과 상관없이 모든 옵저버블을 대체하는데 더 효율적인것은 아닙니다. 따라서 상황에따라, `Observable`과 `Flowable`을 상황에 맞게 알맞게 사용하는 것이 좋습니다.

### `Flowable`이 필요한 경우

- `Flowable`은 백프레셔를 지원하기때문에, 많은양의 데이터가 생성이되어 처리가 필요할 때 유용합니다. 따라서, 약 10,000개 이상의 데이터가 배출이되는 환경이라면 더 효율적일 수 있습니다. 특히 옵저버블 소스가 비동기적으로 동작하여 배출량을 규제할 때 용이합니다.
- 파일이나 데이터베이스 처리 시 유용합니다.
- 비동기적인 네트워크 처리, I/O소스 양을 조절가능한 네트워크, 스트리밍 작업 시 유용합니다.
- 많은양의 UI이벤트(터치 포인트, 제스쳐 등..)에 대한 특정 처리가 필요시 유용합니다.

### `Observable`이 필요한 경우

- 약 10,000개 미만의 소량의 데이터를 다룰 때
- 동기방식의 작업을 원할 때
- 소량의 UI이벤트 발생 시(버튼클릭 등..)



다음시간에는 Flowable의 구현방법과, `Observable`과 `Flowable`에서 어떤 차이점인 큰지, 구독자에서 어떤 부분을 어떻게 사용해야 하는지 보도록 하겠습니다.



[>>포스트 4-01.플로어블과 구독자](https://soda1127.github.io/flowable-and-subscriber/)