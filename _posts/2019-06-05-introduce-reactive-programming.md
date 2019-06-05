---
layout: post
title: "01.리액티브 프로그래밍의 소개"
description: 리액티브 프로그래밍이란 무엇일까요?
image: 'https://i.imgur.com/anSw4YT.png'
category: 'programming'
date: 2019-06-05 14:54:00
tags:
- kotlin
- ReactiveX
- Reactive Programming
introduction: 리액티브 프로그래밍을 소개합니다.
twitter_text: 리액티브 프로그래밍을 소개합니다.
---

# 리액티브 프로그래밍이란?

`Reactive` 반응형이라는 용어는 최근에 많은 사람들 입에서 오르고 내리고 있습니다.

프로그래머라면 누구나 관심을 갖고 있지만, 초반 학습곡선이 높은것으로 유명하여 쉽게 접근하지 못하는 경우가 많습니다.

리액티브 프로그래밍은 무엇일까요?



`Reactive Programming`리액티브 프로그래밍은 데이터 스트림과 변경 사항에 대한 `Propagation`전파를 중심으로 하는 프로그래밍 패러다임 중 하나입니다.

예를 들어보겠습니다.

제가 이전에 맡았던 서비스인 성형 및 뷰티 커뮤니티 플랫폼 [바비톡]에서는 사용자가 어떤 화면을 바라보고 있던 항상 같은 데이터에 대한 동적인 갱신이 요구되었습니다.

모바일 앱의 비즈니스 로직 상 그중에 '성형톡'이라는 탭의 카테고리에서 특정 타임라인에서 특정 글에 대한 행동이 일어난다면, 그 글을 보여주고 있는 다른 카테고리에서도 마찬가지로 그 특정 행동에 대한 반응 및 동적인 데이터 갱신이 요구가 되었습니다.

해당 문제를 해결하기 위해 해당 데이터를 지닌 모델 클래스에 동적인 갱신을 할 수 있는 `Observable`한 객체를 두고 각 프로퍼티를 동적인 갱신을 할 수 있는 로직을 뒀습니다.

성형톡의 ID가 36번인 글의 좋아요를 눌렀으면, 해당 카테고리의 타임라인 외에 다른 카테고리에서 해당 36번글의 좋아요에 대한 리액션을 동적으로 반영해 주어야 할 것입니다. 이러한 작용을 `Reactive`리액티브 라고합니다.

해당 예시와 같이 어떻게든 연결되어 있는 다른 데이터 또는 프로그램에 전파하는 프로그램 방식을  **`Reactive Program`리액티브 프로그램**이라고 합니다.

```kotlin
fun main(args: Array<String>) {
    var number = 4
    var isEven = isEven(number)
    println("The number is" + (if (isEven) "Even" else "Odd"))
    number = 9
    println("The number is" + (if (isEven) "Even" else "Odd"))
}

fun isEven(n: Int): Boolean = (n % 2 == 0)
```

위의 예시는 짝수와 홀수를 구별하는 일반적인 코틀린 코드입니다.



프로그램의 출력을 보면 당연하게도 출력되는 결과는

> The number isEven
>
> The number isEven

일 것입니다. `number` 변수에 새로운 값이 할당됐음에도 `fun isEvent(Int)` 함수에서 변경사항을 체크하지 않았기 때문입니다. 만약 이것이 동적으로 추적이 되는 상황이었다고 하면, 이것은 `Reactive Programming`일 것입니다.