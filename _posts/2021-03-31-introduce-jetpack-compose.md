---
layout: post
title: "Introduce Android Jetpack Compose 🚀"
description: 
image: 'https://imgur.com/9eCJddH.jpg'
category: 'programming'
date: 2021-03-31 13:00:00
tags:
- android
- jetpack
- compose
- declarative ui
- kotlin
introduction: Anroid Jetpack에서 제공하는 라이브러리 - Compose를 소개합니다.
twitter_text: Anroid Jetpack에서 제공하는 라이브러리 - Compose를 소개합니다.

---

## Jetpack Compose란 무엇인가?

Compose는 Native UI를 코드레벨로 구현할 수 있는 최신 툴킷이다. Compose를 이용하면 아래와 같은 장점이 있다.

![image](https://drive.google.com/uc?export=view&id=1vXOGJD7inmGfDYiPu6wMVrpu5xFnov38)

> **출처** [Android Developers#Jetpack Compse](https://developer.android.com/jetpack/compose)

- ### 코드 감소 `Less Code`

  - 적은 수의 코드로 더 많은 작업을 하고 전체 버그 클래스를 방지할 수 있으므로 코드가 간단하며 유지 관리하기 쉽습니다.

- ### 직관적 `Intuitive`

  - UI만 설명하면 나머지는 Compose에서 처리합니다. 앱 상태가 변경되면 UI가 자동으로 업데이트됩니다.

- ### 빠른 개발 과정  `Accelerate Development`

  - 기존의 모든 코드와 호환되므로 언제 어디서든 원하는 대로 사용할 수 있습니다. 실시간 미리보기 및 완전한 Android 스튜디오 지원으로 빠르게 반복할 수 있습니다.

- ### 강력한 성능 `Powerful`

  - Android 플랫폼 API에 직접 액세스하고 머티리얼 디자인, 어두운 테마, 애니메이션 등을 기본적으로 지원하는 멋진 앱을 만들 수 있습니다.



## Jetpack Compose을 사용해야하는 개인적인 이유

지금까지 나온 안드로이드의 네이티브 뷰 컴포넌트 요소는 컴포넌트를 한번 Window에 추가한 이후 인스턴스에서 제공하는 속성값을 제어하는 형태로 구성되어 왔다. 그러다보니, 기존에 있던 뷰 상태를 체크하여 기존 뷰를 변경하는 UI 로직이 필연적이었고, 이런 부분에서 개발자가 집중해야할 본질적인 부분인 비즈니스로직에 대해 신경을 분산시키는 원인이 되기도했다.

이런문제를 깔끔하게 해결해줄 수 있는 Toolkit인 Compose가 리액티브 프로그래밍 모델을 기반으로 선언형 UI Builder 패턴과 함께 우리앞에 선보이게 되었는데, 코드만 보면 화면의 상태`Model` 에 따라 UI의 상태를 결정짓기 떄문에, 더 이상 UI 상태를 보고 UI로직을 구성할 필요가 없다라는 장점이 있다.

또한, Gof의 State 패턴과 같이 화면의 상태를 단일로 표현할 수 있는 수단이 있다면, 화면을 제어할 때 더할나위없이 쉬운방법으로 개발을 경험할 수 있다는 것 또한 장점이다.

일단 코드랩의 코드를 보고 한번 어떤 느낌인지 보도록하자. [Compose CodeLab](https://developer.android.com/codelabs/jetpack-compose-basics?authuser=1#1)

## Codelab 하나씩 정리해보기 - Jetpack Compose basics

위 코드에서 간단하게 OverView를 본 것에 더해, 사용법을 익혀보자. 우리는 너무 많은것을 익혀보기엔 시간이 부족하므로, 간단하게 1 ~ 5까지 함께 보도록해보자.

**새 프로젝트 생성 - Empty Compose Activity 선택**

### ![프로젝트 세팅](https://imgur.com/le8L09z.jpg)

선택 이후 Next를 클릭하고, Compose를 구현할 수 있는 최소 API 레벨인 21을 선택해야한다.

프로젝트를 생성하면 아래와 같이 app/build.gradle에 의존성 설정 및 추가가 되어 있는것을 알 수 있다.

```groovy
android {
    ...
    kotlinOptions {
        jvmTarget = '1.8'
        useIR = true
    }
    buildFeatures {
        compose true
    }
    composeOptions {
        kotlinCompilerExtensionVersion compose_version
    }
}

dependencies {
    ...
    implementation "androidx.compose.ui:ui:$compose_version"
    implementation "androidx.activity:activity-compose:1.3.0-alpha03"
    implementation "androidx.compose.material:material:$compose_version"
    implementation "androidx.compose.ui:ui-tooling:$compose_version"
    ...
}
```

이제 MainActivity.kt를 보자.

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            BasicsCodelabTheme {
                // A surface container using the 'background' color from the theme
                Surface(color = MaterialTheme.colors.background) {
                    Greeting("Android")
                }
            }
        }
    }
}

@Composable
fun Greeting(name: String) {
    Text(text = "Hello $name!")
}

@Preview(showBackground = true)
@Composable
fun DefaultPreview() {
    BasicsCodelabTheme {
        Greeting("Android")
    }
}
```



그러면 아까 보았던 코드를 보았을 때, 총 세가지의 요소를 가지는 것으로 정리해볼 수 있겠다.

- 위젯을 포함하는 Composable 함수
- Preview를 하기 위한 Preview Composable 함수
- setContent 람다 표현식으로 실제 화면에 노출하는 코드

일반적으로 우리가 아는 Activity의 라이프사이클 콜백 `onCreate()`에서   `setContentView(Int)` 함수를 호출하던것이 `setContent()` 함수로 바뀐것이 가장 큰 특징으로 보여진다.

그리고 하단에 `@Composable` 이라는 어노테이션이 보이고, `@Preview` 어노테이션이 보이는데, 이 두가지를 통해 우리는 유추해볼 수 있는 단어의 의미가 있다.

- Composable : 무언가 조합이 가능한 녀석이다. 컴포넌트를 생성하여 조합이 가능할 것으로 보인다.
- Preview : 미리보기를 할 수 있는 녀석이다. @Composable이 아래에 붙은것을 보니 컴포넌트에 대한 미리보기가 가능한 형태로 제공할 것으로 보인다.

자, 그러면 하나하나씩 코드를 보자.

### Composable Function

Composable Function은 어노테이션을 이용한 기술이다. 함수위에 `@Composable` 어노테이션을 붙이게 되면 함수 안 다른 함수를 호출할 수 있게된다. 아래 코드를 보자.

```kotlin
@Composable
fun Greeting(name: String) {
    Text(text = "Hello $name!")
}
```

단순하게 내부에는 Text라는 함수가 존재하는데, 이를 통해 UI계층 별 요구하는 컴포넌트를 생성해준다. 기본적으로 보이는 text 파라미터는 내부 속성에서 받는 일부 중 하나이다.

### TextView 만들기

위 코드를 실행시켜보면 당연하게도 Hello로 시작하는 TextView가 화면에 그려질것을 암시한다.

```kotlin
setContent {
  BasicsCodelabTheme {
    // A surface container using the 'background' color from the theme
    Surface(color = MaterialTheme.colors.background) {
      Greeting("Android")
    }
  }
}
```

![output](https://imgur.com/aO6Jlsg.jpg)

### @Preview

말 그대로 어노테이션을 이용하여 IDE에서 Preview를하기 위한 용도이다. 아래 코드와 같이 @Preview 어노테이션을 추가하면 다음 결과를 볼 수 있다.

```kotlin
@Preview("Greeting Preview")
@Composable
fun GreetingPreview() {
    BasicsCodelabTheme {
        Surface(color = MaterialTheme.colors.background) {
            Greeting("Android")
        }
    }
}
```

![Greeting Preview](https://imgur.com/WprDTs1.jpg)

### 화면을 구성하는데 필수적인 요소들

```kotlin
setContent {
  BasicsCodelabTheme {
    // A surface container using the 'background' color from the theme
    Surface(color = MaterialTheme.colors.background) {
      Greeting("Android")
    }
  }
}
```

기존에 onCreate시점에 화면을 그려주기 위한 필수적인 요소를 정리해보자면

- **setContent** : Activity에서 setContentView함수를 사용하는 것과 동일한 동작을 하는 확장함수이다. 다만, setContent의 경우 (@Composable) -> Unit 타입의 컴포즈 UI를 구현해주어야한다.

- **XXXTheme** : Theme정보를 의미한다. 해당 프로젝트에서는 Theme.kt에 여러 테마에 필요한 정보를 정리하고, 컴포즈 UI 구현을 위한 코드를 작성해두었다.

```kotlin
private val DarkColorPalette = darkColors(
    primary = purple200,
    primaryVariant = purple700,
    secondary = teal200
)

private val LightColorPalette = lightColors(
    primary = purple500,
    primaryVariant = purple700,
    secondary = teal200

    /* Other default colors to override
    background = Color.White,
    surface = Color.White,
    onPrimary = Color.White,
    onSecondary = Color.Black,
    onBackground = Color.Black,
    onSurface = Color.Black,
    */
)

@Composable
fun BasicsCodelabTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colors = if (darkTheme) {
        DarkColorPalette
    } else {
        LightColorPalette
    }

    MaterialTheme(
        colors = colors,
        typography = typography,
        shapes = shapes,
        content = content
    )
}
```

- **Surface** : Greeting을 감싸는 뷰에 해당한다. 여기서는 크기를 정하지 않고, background 색상을 정의하고 있다. 역시 람다 표현식이다. 색상에 대한 Paramter로 `color` 라는 값을 사용하여 부여가 가능하다. 내부코드를 보면

```kotlin
@Composable
fun Surface(
    modifier: Modifier = Modifier,
    shape: Shape = RectangleShape,
    color: Color = MaterialTheme.colors.surface,
    contentColor: Color = contentColorFor(color),
    border: BorderStroke? = null,
    elevation: Dp = 0.dp,
    content: @Composable () -> Unit
) {
    val elevationPx = with(LocalDensity.current) { elevation.toPx() }
    val elevationOverlay = LocalElevationOverlay.current
    val absoluteElevation = LocalAbsoluteElevation.current + elevation
    val backgroundColor = if (color == MaterialTheme.colors.surface && elevationOverlay != null) {
        elevationOverlay.apply(color, absoluteElevation)
    } else {
        color
    }
    CompositionLocalProvider(
        LocalContentColor provides contentColor,
        LocalAbsoluteElevation provides absoluteElevation
    ) {
        Box(
            modifier.graphicsLayer(shadowElevation = elevationPx, shape = shape)
                .then(if (border != null) Modifier.border(border, shape) else Modifier)
                .background(
                    color = backgroundColor,
                    shape = shape
                )
                .clip(shape),
            propagateMinConstraints = true
        ) {
            content()
        }
    }
}
```

## 선언형 UI

노란색 배경을 입혀 기존 TextView에 추가해보았다. 또한, Greeting에는 Modifier라는 것을 이용하여 Padding을 추가했다. 아래와 같은 결과가 나오게 되었다.

```kotlin
BasicsCodelabTheme {
  // A surface container using the 'background' color from the theme
  Surface(color = Color.Yellow) {
    Greeting("Android")
  }
}
```

```kotlin
@Composable
fun Greeting(name: String) {
    var isSelected by remember { mutableStateOf(false) }
    val backgroundColor by animateColorAsState(if (isSelected) Color.Red else Color.Transparent)

    Text(
        text = "Hello $name!",
        modifier = Modifier
            .padding(24.dp)
            .background(color = backgroundColor)
            .clickable(onClick = { isSelected = !isSelected })
    )
}
```

![결과](https://imgur.com/qgQ6oY4.jpg)

선언형 UI의 장점은 말 그대로 내가 UI를 정의한대로 시각적으로 표현이 가능하다는 장점이 있다. 기존에는 속성을 매번 On/Off와 같은 옵션을 통해 변경하는 것이 다반사였지만, 이제는 매번 속성에 변경이 생길때마다 새로 그려주게 되는것이다.

### Compose reusability

컴포즈의 장점 중 하나는 재사용성이 뛰어난것인데, XML에서 우리가 include 태그를 통해 여러곳에서 갖다쓸 수 있던것처럼, 함수를 통해 여러곳에서 정의하여 사용이 가능하다.

참고해야할 점은 컴포즈 컴포넌트 확장 시 @Composable 어노테이션을 붙여 확장이 필요하다.

### Container 작성

MyApp이라는 이름으로 컴포즈 컴포넌트를 구횬하여 여러곳에서 공통으로 사용할 수 있는 Composable을 구현하였다. 내부적으로 Container내 내가 원하는 컴포넌트를 넣어주려면 아래와 같이 인자로 `@Composable () -> Unit` 타입을 넘겨받아 처리해주면 된다.

```kotlin
@Composable
fun MyApp(content: @Composable () -> Unit) {
    BasicsCodelabTheme {
        Surface(color = Color.Yellow) {
            content()
        }
    }
}
```

위 함수를 통해 이제는 어디서든 반복해서 사용할 수 있는 Container를 구현하게 되어 아래와 같이 코드를 활용할 수 있게되었다.

```kotlin
class MainActivity : AppCompatActivity() {
  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContent {
      MyApp {
        Greeting("Android")
      }
    }
    ...
  }
```



### Calling Composable functions multiple times using Layouts

지금까지는 하나의 컴포넌트만을 갖고 사용했지만, 여러개의 컴포넌트를 넣는것도 가능하다.

```kotlin
@Composable
fun MyScreenContent() {
    Column {
        Greeting("Android")
        Divider(color = Color.Black)
        Greeting("there")
    }
}
```

 `Column`과 위에서부터 사용하던 `Greeting` 함수를 사용하고, 라인을 그어주기 위한 `Divider`를 추가한 결과물은 다음과 같다.

![multiple component](https://imgur.com/VLTxB8C.jpg)

위 컴포넌트 중 못보던 컴포저블이 있는데, 아래와 같이 설명이 가능하다.

- Column : 항목을 순서대로 배치하기 위해 사용한다.
- Divider : 선 긋기 가능한 Compose 함수이다.

이를 리스트 형태로도 구현이 가능하다.

```kotlin
@Composable
fun MyColumnScreen(names: List<String> = listOf("Line One", "Line Two")) {
    Column {
        names.forEach {
            Greeting(name = it)
            Divider(color = Color.Black)
        }
    }
}
```

### 상태값 관리

위 컴포넌트에 버튼을 클릭했을 때 클릭한 카운트를 집계하는 간단한 컴포넌트를 만들어보았다.

```kotlin
@Composable
fun MyColumnScreen(names: List<String> = listOf("Line One", "Line Two")) {
    val counterState = remember { mutableStateOf(0) } // 

    Column {
        names.forEach {
            Greeting(name = it)
            Divider(color = Color.Black)
        }
        Counter(
            count = counterState.value,
            updateCount = { newCount ->
                counterState.value = newCount
            }
        )
    }


}
```

rememer라는 함수를 사용하여 기존에 존재하는 컴포넌트의 상태값을 기억하게 하는 함수가 있다. 내부를 한번보면

```kotlin
/**
 * Remember the value produced by [calculation]. [calculation] will only be evaluated during the composition.
 * Recomposition will always return the value produced by composition.
 */
@OptIn(ComposeCompilerApi::class)
@Composable
inline fun <T> remember(calculation: @DisallowComposableCalls () -> T): T =
    currentComposer.cache(false, calculation)
```

매번마다 Recomposition(재조합)하게되는 경우 컴포넌트에 값을 다시 제공하는 것을 알 수 있다. @Composable 어노테이션에 들어간 함수는 매번 해당 상태를 구독하고, 상태가 변경될때마다 알림을 받아 기존 화면을 갱신해준다.

그리고, 아래 Counter를 보면 Button을 이용하여 이벤트를 받아 처리하도록 했다.

```kotlin
@Composable
fun Counter(count: Int, updateCount: (Int) -> Unit) {
    Button(
        onClick = { updateCount(count + 1) },
        colors = ButtonDefaults.buttonColors(
            backgroundColor = if (count > 5) Color.Green else Color.White
        )
    ) {
        Text("I've been clicked $count times")
    }
}
```

`updateCount(Int)` 함수릉 통해 매번 값을 업데이트 해주는데, 이를 통해 counterState에 값을 넣어주면서 해당 컴포넌트가 매번 변경이 되는것이다.

따라서 결과를 보면, 다음과 같다. Count가 5가 넘어가면 초록색으로 바뀐다.

![count](https://imgur.com/Ju6BSg2.gif)

그 외에도 여러형태의 모양을 구성할수 있도록 옵션이 제공되어 있다. 자세한 정보는 나중에 [Codelabs](https://developer.android.com/codelabs/jetpack-compose-basics)에 더 나와 있으니 보도록하고, 이번에 setContent에 대한 동작원리를 함께 고민해보자.

### Activity에서 작성

Compose를 안드로이드 앱에서 사용하려면 Activity, Fragment와 같은곳에서 contentView로 뿌려줘야한다. 기존에 우리가 사용하던 함수를 보자.

```java
/**
* Set the activity content from a layout resource.  The resource will be
* inflated, adding all top-level views to the activity.
*
* @param layoutResID Resource ID to be inflated.
*
* @see #setContentView(android.view.View)
* @see #setContentView(android.view.View, android.view.ViewGroup.LayoutParams)
*/
public void setContentView(@LayoutRes int layoutResID) {
  getWindow().setContentView(layoutResID);
  initWindowDecorActionBar();
}
```

UI 컴포넌트에서 화면을 붙일 수 있는 Window라는 녀석에서 Layout Resource Id를 통해 기존에 등록되어있던 Layout XML 파일을 로드하여 인플레이터에서 파싱하고, 이를통해 레이아웃 계층에 있는 뷰객체를 생성하여 순차적으로 ViewGroup, View를 만들어 넣어주게 된다.

`PhoneWindow`를 보면 자세하게 알 수 있는데, Window를 구현한 setContentView에서 처음에 생성되는 최상위 레이아웃 그 위에 따로 없다면 `installDecor()` 함수를 통해 mContentParent(레이아웃 리소스가 붙게될 ViewGroup)를 생성하고, 하위에 넣어주게 된다.

그러면 기존 방식은 이정도로 설명을하고, 이번엔 Compose에서 `setContent()` 라는 함수를 어떻게 사용하는지 보자.

```kotlin
/**
 * Composes the given composable into the given activity. The [content] will become the root view
 * of the given activity.
 *
 * This is roughly equivalent to calling [ComponentActivity.setContentView] with a [ComposeView]
 * i.e.:
 *
 * ```
 * setContentView(
 *   ComposeView(this).apply {
 *     setContent {
 *       MyComposableContent()
 *     }
 *   }
 * )
 * ```
 *
 * @param parent The parent composition reference to coordinate scheduling of composition updates
 * @param content A `@Composable` function declaring the UI contents
 */
public fun ComponentActivity.setContent(
    parent: CompositionContext? = null,
    content: @Composable () -> Unit
) {
    val existingComposeView = window.decorView
        .findViewById<ViewGroup>(android.R.id.content)
        .getChildAt(0) as? ComposeView

    if (existingComposeView != null) with(existingComposeView) {
        setParentCompositionContext(parent)
        setContent(content)
    } else ComposeView(this).apply {
        // Set content and parent **before** setContentView
        // to have ComposeView create the composition on attach
        setParentCompositionContext(parent)
        setContent(content)
        // Set the view tree owners before setting the content view so that the inflation process
        // and attach listeners will see them already present
        setOwners()
        setContentView(this, DefaultActivityContentLayoutParams)
    }
}
```

이녀석도 마찬가지로 `window.decorView.findViewById<ViewGroup>(android.R.id.content)`  함수를 호출하여 decorView를 가져온다. 만약 compose를 통해 만들어진 최상위 레이아웃이 존재하면, 기존에 inflator에서 ViewGroup, View를 생성해서 넣어주던것 처럼 `setContent()` => window가 Activity/Fragment에 붙으면 `createComposition()`를 호출하여 검증 후 `ensureCompsositionCreated()` 함수를 호출한다. 현재는 내부적으로 `ViewGroup.setContent()` 를 사용하고 있는데, 곧 교체 될 예정이라고 한다. 이코드도 보면 기존에 있는 ViewGroup에 확장함수로 구현한 녀석인데, 쉽게 말해 ViewGroup에 하위 View, ViewGroup에 Composable로 구현된 함수로 컴포넌트를 넣어줄 때 AndroidComposeView라는 객체를 꺼내오거나 없다면 새로 생성하여 넣어준다.

```kotlin
/**
 * Composes the given composable into the given view.
 *
 * The new composition can be logically "linked" to an existing one, by providing a
 * [parent]. This will ensure that invalidations and CompositionLocals will flow through
 * the two compositions as if they were not separate.
 *
 * Note that this [ViewGroup] should have an unique id for the saved instance state mechanism to
 * be able to save and restore the values used within the composition. See [View.setId].
 *
 * @param parent The [Recomposer] or parent composition reference.
 * @param content Composable that will be the content of the view.
 */
internal fun ViewGroup.setContent(
    parent: CompositionContext,
    content: @Composable () -> Unit
): Composition {
    GlobalSnapshotManager.ensureStarted()
    val composeView =
        if (childCount > 0) {
            getChildAt(0) as? AndroidComposeView
        } else {
            removeAllViews(); null
        } ?: AndroidComposeView(context).also { addView(it.view, DefaultLayoutParams) }
    return doSetContent(composeView, parent, content)
}
```



다시 돌아와서, ComposeView의 `setContent()` 이라는 녀석을 보자.

```kotlin
/**
 * A [android.view.View] that can host Jetpack Compose UI content.
 * Use [setContent] to supply the content composable function for the view.
 *
 * This [android.view.View] requires that the window it is attached to contains a
 * [ViewTreeLifecycleOwner]. This [androidx.lifecycle.LifecycleOwner] is used to
 * [dispose][androidx.compose.runtime.Composition.dispose] of the underlying composition
 * when the host [Lifecycle] is destroyed, permitting the view to be attached and
 * detached repeatedly while preserving the composition. Call [disposeComposition]
 * to dispose of the underlying composition earlier, or if the view is never initially
 * attached to a window. (The requirement to dispose of the composition explicitly
 * in the event that the view is never (re)attached is temporary.)
 */
class ComposeView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : AbstractComposeView(context, attrs, defStyleAttr) {

    private val content = mutableStateOf<(@Composable () -> Unit)?>(null)

    @Suppress("RedundantVisibilityModifier")
    protected override var shouldCreateCompositionOnAttachedToWindow: Boolean = false
        private set

    @Composable
    override fun Content() {
        content.value?.invoke()
    }

    /**
     * Set the Jetpack Compose UI content for this view.
     * Initial composition will occur when the view becomes attached to a window or when
     * [createComposition] is called, whichever comes first.
     */
    fun setContent(content: @Composable () -> Unit) {
        shouldCreateCompositionOnAttachedToWindow = true
        this.content.value = content
        if (isAttachedToWindow) {
            createComposition()
        }
    }
}
```

위에 뭐라뭐라 써져 있는데, 결론적으로 `AbstractComposeView` 라는 녀석은 ViewGroup을 상속받은 녀석이며, 모든 composable의 상태가 변화 되었을 때 이를 감지하는 중요한 녀석이다.

`setContent()`라는 함수는 위에서 설명했으니 넘어가고, 이번에는 `Content`라는 녀석을 보자. 이녀석은 추상 메소드로, `createComposition()` 이라는 함수가 호출 되었을 때, 가장 먼저 불리는 함수이다. 아까 언급되었던 `ensureCompsositionCreated()` 함수에서 tree계층의 ComposeView가 다 붙었다면, 이후에 즉시 Content함수가 호출이된다.

```kotlin
@Suppress("DEPRECATION") // Still using ViewGroup.setContent for now
    private fun ensureCompositionCreated() {
        if (composition == null) {
            try {
                creatingComposition = true
                composition = setContent(
                    parentContext ?: findViewTreeCompositionContext() ?: windowRecomposer
                ) {
                    Content() // 이곳에서 뷰가 다 window에 붙게되면 콜백을 호출한다.
                }
            } finally {
                creatingComposition = false
            }
        }
    }
```

그러면 아래 `ComposeView`의 오버라이딩 된 Content가 호출되면서, 기존에 생성된 View에 UI속성과 같은 Content가 붙게된다.

```kotlin
/**
* The Jetpack Compose UI content for this view.
* Subclasses must implement this method to provide content. Initial composition will
* occur when the view becomes attached to a window or when [createComposition] is called,
* whichever comes first.
*/
@Composable
abstract fun Content()
```

Content는 설명에서 보는것과 같이 `createComposition()` 함수 호출 후 View가 Window에 붙은 이후 즉시 호출된다.

최종적으로 `ComponentActivity.setContent(CompositionContext?, @Composable () -> Unit)` 함수에서 구현된 ComposeView 인스턴스를 ContentLayout을 widht/height를 wrapContent크기로 정하여 ContentView를 Set해주게 된다.

```kotlin
/**
 * Composes the given composable into the given activity. The [content] will become the root view
 * of the given activity.
 *
 * This is roughly equivalent to calling [ComponentActivity.setContentView] with a [ComposeView]
 * i.e.:
 *
 * ```
 * setContentView(
 *   ComposeView(this).apply {
 *     setContent {
 *       MyComposableContent()
 *     }
 *   }
 * )
 * ```
 *
 * @param parent The parent composition reference to coordinate scheduling of composition updates
 * @param content A `@Composable` function declaring the UI contents
 */
public fun ComponentActivity.setContent(
    parent: CompositionContext? = null,
    content: @Composable () -> Unit
) {
  	...
		else ComposeView(this).apply {
        // Set content and parent **before** setContentView
        // to have ComposeView create the composition on attach
        setParentCompositionContext(parent)
        setContent(content)
        // Set the view tree owners before setting the content view so that the inflation process
        // and attach listeners will see them already present
        setOwners()
        setContentView(this, DefaultActivityContentLayoutParams)
    }
}
```



## 그래서, Jetpack Compose는 쓸만한가?

쓸만하다고 생각한다. 안드로이드에서 전혀 생각지도 못한 선언형 프로그래밍 방식을 도입할 수 있던 것이 신선하다고 생각하고, 선언형 프로그래밍도 결국 유행이될 것이기때문에, 기존에 선언형 프로그래밍을 쉽게 접했던 개발자들 ex) React, Flutter 등등을 접한 개발자는 더 쉽게 안드로이드 개발을 도전해보지 않을까 싶다. 오히려 반대로 기존에 XML로 레이아웃 코드를 짜던 개발자들은 생소한 방식이라 적응하는데 시간이 꽤나 걸릴 것이다.

다만, 아직도 alpha버전이라 바뀔 수 있는 것이 너무나도 많은상황이다. 앞으로 좋은 예시들이 나오고, 좀 더 기존 안드로이드 개발자들이 익숙해 할만한 컴포넌트로 제공이 되면, 좀 더 많은 개발자들이 선언형 프로그래밍으로 넘어갈 수 있지 않을까 싶다.

지금 당장은.. 본인의 경우 웹 프로그래밍도 경험을 해봤던터라 익숙한 코드패턴이긴하지만, 당장 기존에 있던 코드와 공존하면서 쓰기에는 여간 불편할것 같기도하다. 애초에 Flutter의 경우 선언형 프로그래밍 방식으로 렌더링을 지원하기 때문에 처음부터 개발을 선언형으로 하게되니, 굳이 이럴바에는 Flutter로 서비스를 따로 구현하는 게 낫지 않을까라는 생각이 들었다.

그럼에도 불구하고, 이러한 새로운 시도는 안드로이드 진영에서 침체되었던 분위기에 활기를 띄워주었고, 개발자의 취향에 따라 안드로이드 프로그래밍을 할 수 있도록 폭이 넓어진 부분에서는 매우 좋은것 같다.(공부할 게 또 늘었구만..)











