---

layout: post
title: "Kotlin Basic EX01. Kotlin Scoping Function 1"
description: 코틀린에서 제공하는 강력한 기능인 범위 지정 함수(Scoping Function)에 대해 알아봅시다.
image: 'https://imgur.com/GmkGaOn.jpg'
category: 'programming'
date: 2019-12-15 11:24:30
tags:
- kotlin
- kotlin-basic
introduction: 범위 지정 함수(Scope Function)에 대해 알아봅시다.
twitter_text: 범위 지정 함수(Scope Function)에 대해 알아봅시다.
---

# EX01) 범위 지정 함수

오늘은 잠깐 쉬어가는 타이밍으로 코틀린의 기본으로 제공하는 기능 중 강력하다고 생각하는 `Scope Function`에 대해 소개하도록 하겠습니다.

보통 개발자들이 개발을 할 때 특정 객체에 있는 함수를 연속적으로 사용한다던지, 다른 변수에 동일한 타입의 객체에 다른 속성을 가진 값을 넣을 때 반복적으로 객체의 프로퍼티에 접근을 하는 등의 경우가 많이 생기곤 합니다.

코틀린이라는 언어에서는 이런 경우에 아주 유용하게 사용할 수 있는 함수를 기본적으로 표준 라이브러리를 통해 제공합니다.



## `let()` 함수

T.let() 함수는 이 함수를 호출한 객체를 이어지는 함수 블록의 인자로 전달합니다. 특히 제가 Scoping Function 중 가장 애용하는 함수 중 하나입니다. 함수의 정의는 다음과 같습니다.

> 이 함수를 호출하는 객체를 이어지는 함수형 인자 block의 인자로 전달하며, block 함수의 결과를 반환합니다.

![let function](https://imgur.com/rXFBPE1.jpg)

`let()` 함수를 사용하면 불필요한 변수선언을 하는 것을 방지할 수 있습니다. 커스텀 뷰를 작성하다 보면 길이를 계산한 값을 변수에 저장해 두고, 이를 함수 호출 시 인자로 전달하는 경우가 흔합니다.



다음은 커스텀 이미지 뷰에서 이미지를 세팅해주기 위해 앱 컨텍스트를 전달받는 코드입니다.

```kotlin
class UrlImageView : AppCompatImageView {
  ...
  fun render(url: String) {
        AttoApplication.appContext?.let {
            Glide.with(it)
                .asBitmap()
                .load(url)
                .apply(RequestOptions()
                    .diskCacheStrategy(DiskCacheStrategy.RESOURCE)
                    .skipMemoryCache(false)
                    .centerCrop().format(DecodeFormat.PREFER_ARGB_8888))
                .into(this)
        }
  }
  ...
}
```



이 외에도, NULL 값이 아닌 경우를 체크한 후 특정 작업을 수행하는 코드에도 `let()` 함수를 사용할 수 있습니다. if문을 사용하는 예로는 다음과 같습니다.

```kotlin
open fun bindViews() {
	viewModel.dataListToAdd.asObservable().subscribe { dataList ->
		// dataList가 null이 아닌 경우에만 어댑터를 갱신한다.
		if(dataList != null) {
			dataRecyclerAdapter.submitList(dataList)
			...
		}
	}.add()
  ...
}
```

여기에 `let()` 함수와 안전한 호출인 `?` 연산자를 같이 사용하면 앞의 예시와 동일한 기능을 간편하게 구현이 가능합니다.

```kotlin
open fun bindViews() {
	viewModel.dataListToAdd.asObservable().subscribe { dataList ->
		// dataList가 null이 아닌 경우에만 어댑터를 갱신한다.
		dataList?.let {
			dataRecyclerAdapter.submitList(it)
      ...	
    }
	}.add()
  ...
}
```



## `apply()` 함수

`apply()` 함수는 이 함수를 호출한 객체를, 이어지는 함수 블록의 리시버로 전달합니다. 다음은 이 함수의 정의를 보여줍니다.

> 이 함수를 호출하는 객체를 이어지하는 함수형 인자 block의 리시버로 전달하며, 함수를 호출한 객체를 반환합니다.

![apply 함수](https://imgur.com/u8nZrgz.jpg)

block의 리시버로 전달하기 때문에 블록내에서는 해당 객체 내 프로퍼티나 함수를 직접 호출이 가능합니다. 따라서, 객체이름을 명시하지 않아도 되기때문에 코드를 간편하게 작성할 수 있다는 장점이 있습니다.



다음은 피드 인스턴스에서 블러온 이미지들을 받아 갯수에 따라 피드 이미지 셀의 높이를 조절하는 코드입니다.

Scoping Function을 이용하지 않는다면 Param 객체를 받아 헤당 객체에 여러 속성을 변경하기 위해 객체 이름을 갖고 호출을 해줘야 하는 불편함이 있습니다.

```kotlin
override fun bindData(data : Data, viewModel: DataCellViewModel) {
  super.bindData(data, viewModel)
  if (data is Feed && viewModel is FeedCellViewModel) {
    val feed = data
    ...
    feed.images?.let {
      val params = imageCellView.layoutParams
      if(it.size > 2) {
        params.height = ViewUtil.dpToPixel(240)
      } else {
        params.height = ViewUtil.dpToPixel(180)
      }
      ...
      imageCellView.layoutParams = params
      imageCellView.setImages(it)
    }
  }
}
```

 `apply()` 함수를 사용하게 되면 아래와 같이 바꿀 수 있습니다. `apply()` 함수에 이어지는 블록에 params 객체를 리시버로 전달하여 객체 없이 내부 속성에 접근이 용이합니다.

```kotlin
override fun bindData(data : Data, viewModel: DataCellViewModel) {
  super.bindData(data, viewModel)
  if (data is Feed && viewModel is FeedCellViewModel) {
    val feed = data
    ...
    feed.images?.let {
      imageCellView.layoutParams = imageCellView.layoutParams.apply {
        height = if(it.size > 2) {
          ViewUtil.dpToPixel(240)
        } else {
          ViewUtil.dpToPixel(180)
        }
      }
      ...
      imageCellView.setImages(it)
    }
  }
}
```



## `with()` 함수

`with()` 함수는 인자로 받은 객체를 이어지는 함수 블록의 리시버로 전달합니다. 함수의 정의는 다음과 같습니다.

> 인자로 받은 객체 receiver를 이어지는 함수형 인자 block의 리시버로 전달하며, block 함수의 결과를 반환합니다.

![with 함수](https://imgur.com/nWcqRhL.jpg)



`with()` 함수는 앞에서 설명한 `let()` 함수, `apply()` 함수와는 다르게 이 함수에서 사용할 객체를 인자를 통해 받습니다. 그렇기 때문에 안전한 호출을 사용하여 인자로 전달되는 리시버 객체가 NULL 값이 아닌경우 함수 호출을 막을 수 있는 방법이 없기 때문에 가능하면 상위 코드 블록에서 NULL 값에 대한 체크를 한 후 해당 함수를 사용할 것을 권장합니다.



다음은 `with()` 함수를 사용한 간단한 예입니다. User 인스턴스를 받아 사전에 NULL인지 체크하고, `with()` 함수를 이용하여 User 인스턴스의 프로퍼티에서 값을 꺼내 세션에 저장하는 코드입니다.

```kotlin
private fun doAfterCheckUser(firebaseUser: FirebaseUser? = null, kakaoUser: KakaoUser? = null) {
  // TODO 유저 정보 등록 및 후 기능...
  if(firebaseUser != null || kakaoUser != null) {
    firebaseUser?.let { user ->
			with(user) {
        SessionHelper.userEmail = email.toString()
        SessionHelper.userName = displayName.toString()
        SessionHelper.userPhone = phoneNumber.toString()
        ...
      }
			...
	}
}
```



## `run()` 함수

`run()` 함수는 인자가 없는 익명 함수처럼 사용하는 형태와, 객체에서 호출하는 형태를 제공합니다. 각 함수들의 정의는 다음과 같습니다.

### `run(() ->R):R`

>함수형 인자 block을 호출하고, 그 결과를 반환합니다.

![run 함수 1](https://imgur.com/zYcFjFE.jpg)



### `T.run(() ->R):R`

> 이 함수를 호출한 객체를 함수형 인자 block의 리시버로 전달 후, 그 결과를 반환합니다.

![run 함수 2](https://imgur.com/qsnSWt9.jpg)



`run()` 함수를 인자가 없는 익명함수처럼 사용하는 경우, 복잡한 계산을 위해 여러 임시 변수가 필요할 수 있는데, 외부 또는 전역 변수로 선언할 필요 없이 내부에서 선언이 가능하여 변수 선언 영역을 분리하여 작성이 가능합니다.



다음은 제가 실무에서 사용했던 방식인데, 커스텀 토스트 뷰를 구현할 때 캔버스를 사용하여 drawing 할 때 계산값을 쉽게 반환하기 위해 사용한 방식입니다.

```kotlin
override fun draw(canvas: Canvas) {
  val count = canvas.save()
  val path = Path()
  path.addRoundRect(RectF(0f, 0f, width.toFloat(), height.toFloat()), corner.toFloat(), corner.toFloat(), Path.Direction.CW)
  canvas.run {
    clipPath(path)
    super.draw(this)
    restoreToCount(count)
  }
}
```

객체에서 `run()` 함수를  호출하는 경우 `with()` 함수와 유사한 목적으로 사용할 수 있습니다. 하지만 `with()` 함수와는 다르게 안전한 호출을 사용할 수 있기 때문에 좀 더 유용합니다.



이외에도 `also()` 함수 등 다양한 함수를 지원하고 있습니다. 좀 더 규칙을 정확하게 알아보고 적합한 방법으로 사용해보고자 하는 분은 개인적으로 가장 잘 정리했다 생각한 글인 [코틀린 의 apply, with, let, also, run 은 언제 사용하는가?](https://medium.com/@limgyumin/코틀린-의-apply-with-let-also-run-은-언제-사용하는가-4a517292df29) 글을 읽어보시길 추천드립니다.



다음번에는 좀 더 코틀린에 관한 유익하고 알찬 내용으로 찾아 뵙겠습니다.















