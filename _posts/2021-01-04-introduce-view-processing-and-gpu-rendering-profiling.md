---
layout: post
title: "Android View Processing & GPU Rendering Profiling 🔍"
description: 안드로이드 뷰 렌더링 과정을 이해하고, 이 과정에서 성능 프로파일링을 어떻게 할 수 있는지 알아보겠습니다.
image: 'https://imgur.com/Sr0lsYt.jpg'
category: 'programming'
date: 2021-01-04 00:32:00
tags:
- android
- vsync
- android-view
- android-hwui
- android=graphics
introduction: 안드로이드 뷰 렌더링 과정을 이해하고, 이 과정에서 성능 프로파일링을 어떻게 할 수 있는지 알아보겠습니다.
twitter_text: 안드로이드 뷰 렌더링 과정을 이해하고, 이 과정에서 성능 프로파일링을 어떻게 할 수 있는지 알아보겠습니다.


---

# 이 글을 작성하게 된 이유

얼마전 회사에서 코드리뷰를 하던 도중 동료분의 코드를 보면서 문득 이런생각이 들었습니다.

> 안드로이드 개발자들은 과연 현재 본인이 만드는 앱의 최적화에 얼마나 많은 관심을 두고 있을까?
>
> 그저 기능 구현에 집중하다보니 가장 기본적인 것을 놓치는 경우도 있지 않을까?



이 생각이 드는 순간 저는 현재 개발하고 있는 앱의 성능이 어느정도 수준인지 파악하기 위해 당장 HWUI 프로파일링 툴을 켜서 확인해보고자 했습니다. 그렇게 나오게 된 결과는 생각보다 좋지 않았습니다.

![GPU 렌더링 프로파일 그래프](https://imgur.com/c0C2CQq.jpg)



## 무엇이 문제인가요? 🤔

여러분들이 실제로 앱을 실행 했을 때 몇몇 앱의 경우 뚝뚝 끊기는 현상을 경험하신적이 있으실 겁니다.

이때, 개발자모드를 통해 **HWUI 프로파일링**이라는 도구를 이용하게 되면, 위와같은 알록달록한 막대기들이 화면에 반응이 있을 때 요동치는 것을 볼 수가 있을겁니다. 바로, 이 막대기가 곧 성능을 나타내기도 합니다.



그래서, 저는 여러분들께 생각보다 많은 개발자 분들이 궁금했지만 잘 모를거 같은 내용인 뷰를 어떻게 그리는지 설명하고자합니다.

이 개념을 이해하기 위해 **View Rendering Processing**과 그 과정에서 성능 지표를 확인할 수 있는 **GPU Rendering Profiling**에 대해 이야기 해보고자 합니다.



# Android - GUI Spec History

본론으로 들어가기전, 안드로이드가 어떤 그래픽 기술을 사용해 왔는지 간단히 소개하고잡합니다.

안드로이드는 약 15년전, 리눅스 커널을 기반으로 독자적으로 구성한 운영체제 안드로이드에 **Skia**라는 2D 그래픽 엔진을 사용하기 시작했습니다. 

여러분들이 안드로이드를 공부하셨음에도 **Skia**를 별로 들어보시지 못한 이유는 구글이 주도적으로 개발한 그래픽 엔진이 아니었기 때문입니다.

다만 찾을 수 있는 자료가 좀 있는데, SKIA에서는 아래와 같이 안드로이드 렌더링의 중추가되는 기술들을 제공하는 것을 알 수 있습니다.



## What is `Skia`?

Skia는 Text, 이미지, 도형, 더 나아가 WebView 등 다양한 렌더링 기법에 중추가 되는 기술입니다.

SKIA가 지원하는 기술은 다음과 같습니다.



Skia is a complete 2D graphic library for drawing Text, Geometries, and Images.

- 3x3 matrices w/ perspective
- antialiasing, transparency, filters
- shaders, xfermodes, maskfilters, patheffects
- subpixel text

Device backends for Skia currently include:

- Raster
- OpenGL
- PDF
- XPS
- Picture (for recording and then playing back into another Canvas)



좀 더 자세한 정보는 [이곳](https://skia.org/dev/gardening/android#autoroller_doc)에 가면 볼 수 있으니 한번 둘러보시는 것도 좋을 것 같습니다.



SKIA를 사용하는 안드로이드의 그래픽 서브 시스템을 도식화 하면 다음과 같습니다.

![Android GDI Components](https://imgur.com/LT8CtVd.jpg)



안드로이드의 경우 Canvas를 많이 사용하는데, 이때, Canvas가 HWUI와 Skia를 이용하여 Surface로 요청을 하는 구조를 띄고 있습니다. Android 3.0 전까지는 Skia와 HWUI를 함께 사용하였지만, 성능이슈로 인해 HWUI 모듈로 점점 대체되고 있는 추세입니다.



## Android HWUI(OpenGLRenderer - Above 3.0)

Android 3.0 Honeycomb에서 공식적으로 Tablet을 위한 OS가 지원이 되기 시작할 때 이전까지 SKIA로만 처리되던 렌더링 방식이 화면이 커지면서 Animation이 매끄럽게 동작하지 않게 되자 HWUI로 대체되기 시작했습니다.

성능적인 이슈가 생기게 되었던 이유는 안드로이드 초기 때 ldpi(~ 120dpi) mdpi(~ 160dpi) 등 저 밀도의 해상도를 가진 디바이스만 출시된 데 반해, 현재는 안드로이드 화면이 점점 대형화 되면서 처리해야할 픽셀이 급격히 많아졌기 때문입니다.

그렇기 때문에, SKIA+GPU 형태로는 Drawing 명령에 대해서 즉각적으로 처리하는 구조로 GPU와 메모리 액세스가 급격하게 증가되어 퍼포먼스가 스펙이 좋아질 수록 오히려 떨어졌기 떄문이었습니다.

이를 구조적으로 개선하기 위해 HWUI에서 다양한 개선방안으로 GPU 사용상을 최적화하여 현재 주요 렌더링 엔진으로 자리잡게 되었습니다.



# Android View Rendering Mechanism

상기에서는 Android가 스크린에 렌더링을 하기 위해 사용해온 기술에 대해 소개를 했다면, Android View가 어떤식으로 Processing되어 보여지는지 한번 같이 알아보겠습니다.

> 아래 설명은 [Android - Graphic Architecture](https://source.android.com/devices/graphics/architecture) 문서를 참고하여 설명하였습니다.

## Surface and SurfaceHolder

- Surface는 화면에 표시하는 이미지를 앱이 렌더링 할 수 있도록 해줍니다.

- SurfaceHolder 인터페이스는 앱이 노출 영역을 수정하고 제어할 수 있게 해줍니다.

클래스는 안드로이드 1.0부터 안드로이드 API에 포함되어 있었으며, 안드로이드 온라인 도움말 사이트에는 Surface 클래스에 대해서 **Screen Compositor**가 관리하는 하나의 Raw 버퍼를 가리키는 핸들이라고 설명합니다.

설명이 굉장히 복잡하니 쉽게 풀어서 설명하자면, **BufferQueue**는 그래픽 데이터의 버퍼를 생성하는 요소(생산자)를 표시 데이터 또는 추가 처리를 허용하는 요소(소비자)에 연결합니다. 

**Suface**는 자주 사용되는 **BufferQueue**를 생성하고(Producer), 버퍼에 데이터를 담아 **SurfaceFlinger**(Consumer)로 전달하여 출력합니다. 이를 통해 화면에 drawing을 하여 Application과 상호작용합니다.



안드로이드에서 모든 그려지는 UI 요소는 대부분은 [OpenGL ES](https://source.android.com/devices/graphics/arch-egl-opengl) 또는 [Vulkan](https://source.android.com/devices/graphics/arch-vulkan)을 사용하여 View를 렌더링합니다.

하지만, 예외 요소도 있기 마련입니다. 예를들면 2D게임을 구현한다던지, 스트리밍 컨텐츠를 보여주는 플레이어라던지, 카메라 등에서는 캔버스 API를 통해 뷰를 그리는 작업이 이뤄집니다. 이를 **Canvas Rendering**이라 합니다.

### Canvas Rendering

캔버스 구현은 [Skia 그래픽 라이브러리](https://skia.org/)에서 제공합니다. 직사각형을 그리고 싶은 경우 적절히 버퍼에 바이트를 설정하는 캔버스 API를 호출합니다. 

안드로이드의 경우 일반적으로 그리기를 프레임워크에 맡겨서 Rendering Process를 거치지만(우리가 알고 있는 `onDraw()` 함수를 통해 그리는 것을 말합니다.), 위에서 언급한 Canvas Rendering의 경우 쓰레드를 이용해 화면에 강제로 그려 원하는 시점에 화면에 그릴 수 있습니다.

그 과정에서 일어나는 매커니즘은 매우 복잡하지만, 결론은 Surface로 전달 되는 것입니다.



Surface까지 도달하기 위해 어떤 흐름을 타는지 알아보겠습니다.

아래 내용은 [Android Drawing Process 1(App surface, SF Layer)](https://lastyouth.tistory.com/24) 글을 참고하였습니다.

![Android Drawing Process Flow](https://imgur.com/vEIUOYU.jpg)

위 이미지는 사용자의 인터렉션에 의해 화면이 갱신되는 Flow를 계층적으로 나타낸 것입니다.



앱에서 Android Framework Level에서 SurfaceFling API를 호출하는 것을 볼 수 있는데, 이 과정에서 **VSync Processing, Screen Refreshing, Layer Management** 루틴을 구성하여 프로세싱합니다.

![Surface Flinger Management](https://imgur.com/VmvT2UB.jpg)



여기서 말하는 이 세가지가 무엇인지는 아래 설명에서 보도록 하겠습니다. 들어가기전, 앱 내 View가 그려지는 구조를 간단하게 보도록하겠습니다.

## Drawing Process(Application Side)

![Android Drawing Process](https://imgur.com/uKjiHKu.jpg)

각 어플리케이션은 여러 ViewController(Activity, Fragment)등에 여러 뷰를 가질 수 있으며, 각 View는 DecorView를 최종 루트로 가지는 트리 구조로 관리됩니다.

DecorView는 그릴 것이 있는 경우, 할당받은 Surface에 그리게 됩니다.. 따라서 Surface는 각 Application별로 최소 하나는 가지고 있게 됩니다.



그러면 위에서 말한 Layer Managent의 의미는 무엇일까요? 바로 Surface입니다. 두 단어의 의미는 동일하다 봐도 무방합니다.

Surface와 SurfaceFlinger는 서로 데이터를 주고 받기 위해 **ISurfaceComposer**를 이용합니다.

앱에서는 **SurfaceComposerClient**로 **ISurfaceComposer**를 통해 `createSurface()` 호출한 것을  `createLayer()`로 바꿔줍니다. SurfaceFliger에서는 해당 앱의 레이어를 생성하여 관리하게됩니다.

Surface가 앱에서 소유하는 개념이라면, 그것을 SurfaceFliger에서 Layer 형태로 관리하는 것입니다.

![Surface & Layer Flow](https://imgur.com/scKZpNX.jpg)

다음은 VSync Processing에 대해 이야기 해보겠습니다.

## VSync Processing

### VSync 소개

**VSync**는 Vertical Synchronization(수직 동기화)의 약자입니다. Android 4.1 Project Butter에서 사용하게 된 기술입니다.

![Dwawing with VSync](https://imgur.com/ha7Eriy.jpg)

**VSync**는 안드로이드 기반 디바이스에서 화면의 출력되는 정보가 변경이 되었을 때 이를 동기화하고, 지속적으로 갱신해주는 기능입니다. 쉽게 말씀드리면, GPU의 Rendering Rate, 즉 fps와 Device Presenting Rate, 즉 hz사이의 간극이 있을 때 이를 조정해줍니다.

VSync는 주기적으로 신호를 발생시켜 화면을 갱신시키는 함수를 주기적으로 호출해줍니다.일반적으로 하드웨어의 경우 아직까지는 60Hz이고, 최근에 고사양 디바이스의 경우 120Hz, 더 나아가서는 LPTO 기술을 사용하여 1 ~ 120Hz로 발생주기를 조절하기도 합니다.

예를들어, GPU는 100fps(초 당 100개의 프레임을 그림), Device Screen은 60hz(초 당 60개의 화면을 그림)라면, GPU에서 그리는 것을 그대로 Device Screen에 뿌리게 되면 계속 Delay가 생길 수 있습니다.

이 때, VSync가 GPU 프레임과 Device 화면간의 Time Gap을 계산하여 필요한 Frame만 그려주어 최적화를 해줍니다.
VSync에 대한 Detail한 원리에 대해서 잘 소개된 영상은 [Android Performance Patterns: Understanding VSYNC](https://www.youtube.com/watch?v=1iaHxmfZGGc) 영상을 참고하시기 바랍니다.

###  Choreographer

 **Choreographer**에서는 SurfaceFlinger에서 송신되는 Vsync Event를 요청/수신하여 다음 작업들을 처리합니다.

- **Input Event Handling**
- **Self-Invalidation**
- **Animation**

### ViewRootImpl

**ViewRootImpl**은 DecorView와 **Choreographer**를 연결해주는 일종의 핸들러 역할을 하는 객체로써, DecorView와 Choreographer 사이에서 둘의 요청을 처리하고 중계합니다.

## Drawing Process(Framework Side)

따라서, 아래와 같은 Flow를 보여줍니다.

![Drawing Process](https://imgur.com/epHCCys.jpg)

실제 그리기는 현재 foreground에 표시되고 있는 어플리케이션에서 그릴 것이 생김으로써 시작합니다. 무언가 그릴 것이 생긴다면, ViewRootImpl 객체 내에 있는 `scheduleTraversal()` 메서드가 호출됩니다.

`scheduleTraversal()` 메서드 내부에서는 **Choreographer** 객체에게 다음 vsync를 예약해달라는 요청을 보내게 됩니다. 이 작업이 위 그림에서 Invalidate와 Vsync scheduling에 해당합니다.

Vsync 예약 요청을 받은 Choreographer는 SurfaceFlinger로 하여금 다음 Vsync 도착 시 알려달라고 요청합니다.

다음 Vsync signal이 도착하면 Choreographer는 ViewRootImpl의 `performTraversal()` 메서드를 호출한다. performTraversal 메서드 내부에서는 다시 그려야 될 부분을 `measure()`하고, layout을 재 구성한 다음, `performDraw()` 메서드를 호출하여 그리기를 수행합니다.

ViewRootImpl 객체에서는 performDraw 메서드에 의해 호출되는 draw 메서드가 최종적으로 DecorView로 하여금 할당받은 Surface에 그리게 됩니다.

이때, Rendering 방법에서 **HardwareRenderer**를 이용하는 경우와 **SoftwareReneder**를 이용하는 경우 두가지로 나뉘게 됩니다.

**SofrwareRenderer**의 경우 3.0 이전 버전의 경우 CPU를 통해 그려주지만, 3.0 이후 버전부터는 하드웨어 가속을 사용합니다.



# GPU Rendering

하드웨어를 기반으로 렌더링 하는 것을 설명하기 전, 소프트웨어로 렌더링 되는 방식에 대해 간단히 설명하겠습니다.

## Software Rendering Model

Software Rendering Model은 다음 두 단계로 그립니다.

1. 계층 구조 무효화
2. 계층 구조 그리기

## 하드웨어 가속

Android 3.0(API 레벨 11)부터 Android 2D 렌더링 파이프라인 하드웨어에서는 가속화를 지원하므로, `View`의 캔버스에서 실행되는 모든 그리기 작업에서 GPU를 사용합니다. 하드웨어 가속을 사용하려면 필요한 리소스가 늘어나므로 앱에서 더 많은 RAM을 사용합니다.

Android 시스템에서는 여전히 `invalidate()` 및 `draw()`를 사용하여 화면을 업데이트하고 보기를 렌더링하지만, 실제 그리기는 다르게 처리합니다. 

Android 시스템에서는 즉시 그리기 명령을 실행하지 않고, 보기 계층 구조의 그리기 코드 출력을 포함하는 표시 목록에 기록합니다.

또한 Android 시스템에서 `invalidate()` 호출을 통해 더티로 표시된 보기의 표시 목록을 기록하고 업데이트만 하면 되도록 최적화됩니다.

무효화되지 않은 보기는 단순히 이전에 기록된 표시 목록을 다시 실행하여 다시 그릴 수 있습니다. 새 그리기 모델에는 다음 세 단계가 포함됩니다.

1. 계층 구조 무효화
2. 표시 목록 기록 및 업데이트
3. 표시 목록 그리기



지금까지는 View가 렌더링 될 때 두가지 방식으로 어떻게 동작하는지 개념 설명 및 원리 설명을 드렸습니다. 그러면, 하드웨어 가속을 사용하는 GPU 렌더링 체계의 경우 어떤 문제가 있고, 어떻게 프로파일링 하는지 보도록 하겠습니다.



# Android GPU Rendering Profiling

## Android - Overdraw란?

시각적으로 UI는 동일하지만, 실제로 화면의 보이는 컴포넌트의 구성방식은 정말 다양한 형태로 구성이 가능합니다.

이때, 같은 프레임 내 두번이상 같은 픽셀을 그릴 때 Overdraw가 발생합니다.

Overdraw 빈도 수가 높아지는 경우 추가 GPU 작업으로인해 성능저하가 발생할 수 있습니다. 따라서, 해당 문제를 해결하기 위해 GPU 오버드로 시각화 도구를 사용하여 어떤 뷰에서 Overdraw가 많이 발생하는지 알 수 있습니다.

**여는방법**

1. 기기에서 **설정**으로 이동하여 **개발자 옵션**을 탭합니다.
2. **하드웨어 가속 렌더링** 섹션으로 스크롤하여 **GPU 오버드로 디버그**를 선택합니다.
3. **GPU 오버드로 디버그** 대화상자에서 **오버드로 영역 표시**를 선택합니다.

![Android Overdraw](https://imgur.com/lQs5GtK.jpg)

그렇다면 위와같이 색상에 따라 오버드로 빈도수를 시각적으로 표현 가능합니다.

## How to avoid Overdraw

오버드로를 제거하는 방법은 다음과 같습니다.

- 레이아웃에서 불필요한 배경삭제
- 뷰 계층 구조 평면화 => 최대한 관계성 설정하여 뷰 계층을 줄이기(like ConstraintLayout)
- 투명도 존재유무에 따라 Rendering Cost가 상승 => [Hidden Cost of Transparency](https://www.youtube.com/watch?v=wIy8g8yNhNk&index=46&list=PLWz5rJ2EKKc9CBxr3BVjPTPoDPLdPIFCE) 참고



이를 측정하기 위해 개발자 옵션에서는 HWUI 프로파일링 옵션을 사용할 수 있습니다.

## GPU Rendering Speed Profiling

60fps(GPU 연산으로 초 당 60프레임으로 화면을 뿌리는 것)일 때 렌더링에 얼마나 성능을 잡아먹는지 확인하기 위해 개발자 옵션을 통해 다음 도구를 사용설정 할 수 있습니다.

Android 3.0(Honeycomb) 버전 이상은 HWUI 프로파일링 옵션을 켜서 사용이 가능합니다.



**여는방법**

1. 기기에서 **설정**으로 이동하여 **개발자 옵션**을 탭합니다.
2. **모니터링** 섹션에서 **프로필 GPU 렌더링**을 선택합니다.
3. '프로필 GPU 렌더링' 대화상자에서 **화면에 막대로 표시**를 선택하여 기기의 화면에 그래프를 오버레이합니다.
4. 프로파일링하려는 앱을 엽니다.



**빠른 접근 방법**

1. 기기에서 **설정**으로 이동하여 **개발자 옵션**을 탭합니다.
2. 빠른 설정 개발자 타일을 엽니다.
3. 프로필 HWUI 렌더링 옵션을 On/Off 합니다.
4. Status Bar를 퀵 설정 타일에서 관리 가능합니다.



설정을 활성화 하고 확인해보면, 위와 같이 히스토그램으로 표시되는 것을 볼 수 있습니다. 여기에서 색상이 뜻하는 의미를 각자 살펴보자면 다음과 같습니다.

![HWUI Profiling](https://imgur.com/sxNtxZq.jpg)

- 표시되는 각 애플리케이션에 관해 그래프가 표시됩니다.
- 가로축의 각 세로 막대는 프레임을 나타내며 각 세로 막대의 높이는 프레임을 렌더링하는 데 걸린 시간(밀리초)을 나타냅니다.
- 녹색 가로선은 16밀리초를 나타냅니다. 60fps를 달성하려면 각 프레임의 세로 막대가 이 선 아래에 머물러야 합니다. 막대가 이 선을 넘어서면 애니메이션이 일시중지될 수 있습니다.
- 16밀리초 임계값을 초과하는 프레임은 막대를 더 넓고 덜 투명하게 만들어 강조표시됩니다.
- 각 막대에는 렌더링 파이프라인의 단계로 매핑되는 색상으로 표시된 구성요소가 있습니다. 구성요소의 수는 기기의 API 수준에 따라 다릅니다.

![Android HWUI Profinling Spec](https://imgur.com/pR4QUY1.jpg)



지금까지 안드로이드 뷰 렌더링 과정을 이해하고, 이 과정에서 성능 프로파일링을 어떻게 할 수 있는지 알아보았습니다.

뷰 렌더링 과정은 일반적인 앱 개발과정에서 딥하게 연구할필요는 없지만, 알아두면 더 성능좋은 앱을 만들 수 있도록 돕는 매우 중요한 지식이기도합니다. 여러분들에게 조금이나마 도움이 되길 바라면 글 마치도록 하겠습니다. 감사합니다.