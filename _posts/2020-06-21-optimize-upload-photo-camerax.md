---
layout: post
title: "이미지 업로드 최적화 해보기 - CameraX"
description: 사진청구라는 기능을 위해 업로드 UX 및 속도 개선을 해결해보았습니다.
image: 'https://imgur.com/DQEJV65.jpg'
category: 'programming'
date: 2020-06-21 18:30:00
tags:
- kotlin
- coroutines
- android
- android-jetpack
- rxjava
- android-lifecycle
introduction: 메디패스의 사진청구 기능을 개발하면서 최적화를 위해 해결과정을 소개합니다.
twitter_text: 메디패스의 사진청구 기능을 개발하면서 최적화를 위해 해결과정을 소개합니다.
---

# 사진 업로드를 위한 최적화 삽질기(Optimization process for uploading photos) - 카메라편 

오랜만에 인사드립니다. 최근에 바쁜일이 있어 블로그 글을 자주 못쓰게 되었는데요,

새롭게 메디블록이라는 회사에서 메디패스라는 의료 보험청구 서비스를 개발하고 있습니다. 메디블록에서 처음 들어와서 개발을 맡게 된 기능은 바로 메디패스의 사진청구 기능이었습니다.

이전에도 사진을 직접 촬영하여 여러장을 업로드하는 서비스를 개발해본 적이 있었으나, 그때 당시에는 디바이스 별 최적화를 크게 고려하지 않고 개발하다보니 업로드 속도가 꽤 오래걸리고, 몇몇 힙 메모리가 적은 저사양의 디바이스에서는 `OOM(OutOfMemory)`이 발생하거나, 처리과정에서 과부하가 발행해 배터리 소모가 빨랐던 기억이 납니다.

이번에는 해당 문제를 해결하기 위해 배포 이전에 여러 방면에서 고민을 하게 되었는데, 이에 대한 삽질기를 이야기로 풀어볼까 합니다.



## 사진청구를 위해 해결해야 할 사항들

사진청구는 현재 제공하는 로켓청구 서비스와는 다르게 병원진료를 받은 후 영수증이나 진료내역서를 촬영하여 본인/타인에 대한 정보를 입력하고 갤러리에서 사진을 선택하거나, 촬영하여 사진을 최대 12장까지 첨부하고, 청구자/피보험자의 서명을 받고 청구를하는 기능을 제공합니다. 사진청구를 원할하게 하기 위해, 해결해야 할 사항은 다음과 같았습니다.

- 사진은 촬영이후 A4용지 비율에 맞게 크롭핑하여 저장
- 첨부된 사진은 팩스화 되어 전달이 되어야 하기 때문에, 노이즈가 적어야 함(여기서 노이즈라 말함은, 특정부분의 번짐, 왜곡 현상) 따라서, 디노이즈 작업 후 업로드 필요
- 사진은 모두 가로 기준 같은 픽셀밀도를 가질 필요가 있고, 같은 포맷에 같은 사진 퀄리티로 변형 이후, 흑백화가 필요
- 최대 12장의 jpeg포맷 사진, 1~2장의 서명 png포맷 이미지 업로드 시 가공과 업로드 시간은 5 ~ 10초 내외로 업로드할 수 있도록 구현 필요(일반적인 와이파이 환경, 5년전 노트5 디바이스로도 충분히 업로드 될만한 수준)
- 현재 기획상 포그라운드 UI에서 로딩을 돌리고 있으나, 추후 변경 될 사항을 대비하여 백그라운드 프로세스에서도 돌아갈 수 있도록 구현
- 비동기 방식 업로드 프로세싱 구현



## 최적의 방식을 찾아보기

달리 생각하면 쉬운 작업일수도 있겠으나, 생각보다 이미지 업로드하는 과정에서 저사양 디바이스의 경우의 이슈들을 해결하기 위해 과정이 순탄하진 않았습니다. 여러 이슈들을 해결하기 위해 `Android Jetpack`의 여러 도구들을 활용하여 해당 기능을 쉽고 빠르게 만들고자 했습니다.(스타트업에겐 시간이 없기에..)

- **`CameraX`** - Camera2 기반 API Support로, 파편화 된 여러 디바이스의 카메라 하드웨어를 동작할 수 있도록, 사용하기 쉽도록 제공된 라이브러리입니다.
- **`WorkManager`** - Android 백/포그라운드 작업을 비동기방식으로 쉽게 예약/관리/처리 할 수 있습니다.(저희는 Coroutines를 사용중이기 때문에 **CoroutineWorker**를 사용하였습니다.)
- **`Coroutines`** - 현재 RxJava를 여러군데에서 사용하고 있으나, 좀 더 비동기 루틴을 안전한 방식으로 처리하기 위해 클린아키텍쳐 패턴에서 ViewModel/UseCase/Repository와 같은 레이어에서 사용을 하고 있습니다.
- **`LiveData`** - 처리되고 있는 로직에서 UI에 여러 상태/데이터를 쉽게 넘기기 위해 사용합니다.



## 사진 촬영 라이브러리 고민하기 - **`CameraX`**

이전에 사진업로드 기능을 구현하면서 Camera1, Camera2 하드웨어에 대해 지원을 하는 카메라 기능을 구현한 적이 있었습니다.

이번에는 여러 디바이스에 대응을 꾸준히 하고 있는 AndroidX의 **CameraX**를 사용을 고려하게 되었습니다.

!["CameraX 소개"](https://imgur.com/tu0RynB.jpg)

[이전에 나온 많은 안드로이드 디바이스의 경우 카메라 하드웨어에 대응하기 위한 Camera1, Camera2의 API를 사용했는데, 파편화가 워낙 심해 이를 해결하기위해 나오게 되었습니다.]

다만, 아직 우려되는 사항은, 보시는 것과 같이 베타단계의 라이브러리이고, 이를 서포트 하는 라이브러리도 알파단계 라이브러리를 같이 사용해야 하기때문에, 이를 유념하고 사용할 필요가 있습니다.

저는 Camera2 API를 바로 서비스에 적용하여 개발하기엔 리서칭에 필요한 리소스가 많이 필요하다 판단이 들었고, 여러 디바이스에 대한 대응이 쉽지 않을 것이라 판단하여 CameraX를 사용하게 되었습니다.

(하지만 몇몇 Android 6.0 버전대의 과거 디바이스에서 캡쳐 포커스 이슈 등 대응이 제대로 되지 않았던 문제가 있어 6.0의 디바이스에서는 Camera1 API를 이용하여 대응을 하였습니다. CameraX / Camera1 화면 분기 처리 방식으로..)



### `CameraX` 기반 사진 촬영 화면 구현 참고사항

처음에는 전체적인 코드 구성방식을 참조하기 위해 `CameraX`코드 샘플을 보게 되었습니다.

참조한 코드는 다음 레포지토리에서 확인하실 수 있습니다. [CameraX-Basic](https://github.com/android/camera-samples/tree/master/CameraXBasic)

꾸준히 CameraX 라이브러리 버전업이 되면서 변경된 사항에 대해 잘 소개하고 있으니 참고하셔도 좋습니다.(저 또한 적극적으로 참고하여 개발을 하였습니다.)



### `CameraX` 기반 촬영 화면 구현하기

먼저 화면의 틀 부터 잡아줄 필요가 있다고 판단을 하여 구현을 하기 시작했습니다.

![카메라 화면 구현하기](https://imgur.com/IkG3znx.jpg)

보는 것과 같이 Preview 화면 안에 사진 촬영 후 크롭을 위한 작업이 필요하게 되었습니다.

(기본적으로 최적화를 위해 잘리는 공간이 있을 수 있으나 4:3의 비율로 Preview를 구현하였습니다.)

촬영을 하게 되면, 기본적으로 4:3 비율의 사진이 캡쳐 후 저장되며, 이후 해당 바운더리의 비율에 맞게 크롭핑되어 첨부될 사진이 준비되게 됩니다.



### How to access Camera Configuration & Provide UseCases

카메라의 경우 보는 것과 같이 미리보기와 같은 화면이 없는 이상 유저가 어떤 사진을 촬영하고 있는지 판단하지 못합니다. 따라서 미리보기 화면 제공을위한 다음 과정이 필요합니다.

1. `CameraXConfig.Provider` 를 구성합니다.
2. 레이아웃에 미리보기 화면 뷰  `androidx.camera.view.PreviewView` 추가합니다.
3. `CameraProvider`를 요청합니다.
4. `View`를 만들 때 `CameraProvider`를 확인합니다.
5. 카메라를 선택하고 수명 주기 및 `UseCases`를 결합합니다.



#### `CameraXConfig.Provider` 구성하기

메디패스의 경우 Camera2 하드웨어 서포트를 위해 CameraXConfig를 래핑한 `Camera2Config.Provider`를 구현하였습니다.

```kotlin
class MedipassApplication : Application(), ... CameraXConfig.Provider, ... {
  ...
  override fun getCameraXConfig(): CameraXConfig = Camera2Config.defaultConfig()
  ...
}
```

#### `androidx.camera.view.PreviewView` 추가

메디패스의 경우 모든 대부분 뷰와 뷰모델에 대한 처리의 위임을 Activity가 아닌 Fragment에 넘겨 처리하고 있습니다.

따라서, Fragment layout에 4:3 비율에 맞는 `PreviewView`를 구성하게 되었습니다.

```xml
<androidx.camera.view.PreviewView
	android:id="@+id/viewFinder"
	android:layout_width="0dp"
	android:layout_height="0dp"
	...
	app:layout_constraintDimensionRatio="h,4:3"
	app:layout_constraintHeight_default="percent"/>
```

#### CameraProvider 요청.. 하기전에 먼저 권한부터 체크하기

우선적으로, 사진 촬영 이후 외장메모리에 접근하여 쓰기위해 두가지 권한을 받아서 관리하려고 합니다.

```xml
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.FLASHLIGHT" />
```

`onStart()` 호출 이후 권한을 체크하여 뷰(툴바 ,카메라, 액션이 일어나는 그 외 뷰들)를 초기화 해줍니다.

```kotlin
override fun onStart() {
  super.onStart()
  listOf(Manifest.permission.CAMERA, Manifest.permission.WRITE_EXTERNAL_STORAGE)
  .checkPermissionByRx(requireContext(), compositeDisposable, {
    initViews(binding)
  }, notGrantedActionHandler = {
    requireActivity().finish()
  }, notGrantedTitle = "실패", notGrantedMessage = "권한 허용되지 않음")
}
```



### `CameraProvider` 요청

```kotlin
private val cameraProviderFuture by lazy { ProcessCameraProvider.getInstance(requireContext()) } // 카메라 얻어오면 이후 실행 리스너 등록
...
private fun bindCameraUseCases() {
  ...
  cameraProviderFuture.addListener(
    Runnable {
      val cameraProvider: ProcessCameraProvider = cameraProviderFuture.get()
      // register UseCases
    }, ContextCompat.getMainExecutor(requireContext())
}
```

PreviewView와 Camera를 바인딩 하기전, `CameraProvider`를 통해 객체를 얻어온 것을 확인하고, UseCases를 만들어줍니다.



### Bind UseCases used for Preview, Capture, Analysis(If you want) also with lifecycle

앞서 설명드린 카메라를 선택한 이후, 라이프사이클과 결합하는 과정에서 카메라에 여러 옵션을 부여하기 위해 사용사례`UseCases`를 함께 결합을 해야합니다.

코드는 다음과 같습니다.



### Capture and Save Photo

마지막으로 사진 캡쳐 및 저장 작업이 필요합니다. 기본적으로 AutoFocus를 하고있으나, 캡쳐 전 AutoFocus를 한번 더 요청 해 사진이 흔들리지 않도록 구현을 해주었습니다.

```kotlin
private fun updateCaptureButton(...) {
  with(binding) {
    captureButton.onClickDebounced {
      ... // 여러 로직처리
      startFocus(isCapture=true) //captureCamera()
    }
  }
}
...
private fun startFocus(isCapture: Boolean = false) {
  with(binding) {
    ... // focus를 주기위한 meteringPoing 얻어오기
    
    try {
      ... // 포커스 설정 및 시작
      if (isCapture) {
        focusMeetingFuture?.addListener(Runnable {
          val focusMeetingResult = focusMeetingFuture.get()
          if (focusMeetingResult.isFocusSuccessful) {
            captureCamera()
          }
        }, ContextCompat.getMainExecutor(requireContext()))
      }
    } catch (e: CameraInfoUnavailableException) {
      ... // 카메라 정보 존재하지 않을 시 예외처리
    }
  }
}
```

이후 캡쳐를 하고 저장합니다.

```kotlin
private fun captureCamera() {
  ... // 캡쳐음, 메타 정보들..
  val outputOptions = OutputFileOptions.Builder(photoFile)
		  .setMetadata(metadata)
		  .build()
  cameraExecutor?.let { // newSingleThreadExecutor()
    imageCapture.takePicture(outputOptions, it, object : OnImageSavedCallback {
      ... // OutputFileResults 인스턴스에서 원하는 방식으로 저장 처리
    }
  }
}
```



여기까지 보았을 떄, CameraX가 Camera2에 비해 눈에 띄게 개발자들이 도메인 로직에 집중하라고 많은 부분에서 서포트를 했으며, 디바이스 별 최적화를 위한 많은 고민을 덜어준 것을 알 수 있었습니다. 

그리고, 필요한 경우 여러 필터나 캡쳐 시 적용되는 효과를 UseCases에 담아서 적용하는 것을 알 수 있습니다. 아직 베타버전의 라이브러리라 계속 코드 변경사항이 생기고, 체인지로그가 자주 올라오는 것을 볼 수 있었습니다.

정말 최근까지도 자주 바뀌는 것을 알 수 있습니다😭(계속 개발하면서 여러 버그들이 발견돼서 수정됐고, 로직상, 코드상 수정이 정말 자주 일어나 많은 대응이 요구되었습니다.)

![CameraX ChangeLog](https://imgur.com/aH3TLey.jpg)



다음 시간엔 사진을 촬영하고, 저장해보았으니, 본격적으로 사진을 업로드하기 좋게 가공하여, 업로드를 어떻게 최적화 할 수 있었는지 보도록 하겠습니다.

