---
# published: false

title: "[안드로이드] Robolectric 실전 테스트"
comments: true
# pagination:
#     enabled: true
categories:
    - android
tags:
    - android
    - test
date: 2021-1-25T01:15:00Z
---

Robolectric은 이 글의 제목에도 있듯이 여러 AndroidX Framework를 local에서(device or emulator 없이) 테스트할 수 있는 도구입니다.

이 테스트들은 개인 앱에서 robolectric을 사용한 케이스입니다.
<br>
    
    기본적인 설정(build.gradle(.app))은 인터넷에 많이 나와 있으니 SKIP >

<br>

Espresso는 유저 interaction에 특화된 라이브러리고 Robolectric은 그보다는 뷰의 데이터나 위치들 (또는 내부 로직) 이런 정보들을 테스트하는데 적합한 라이브러리가 되겠습니다.

assert는 Truth라이브러리를 사용했습니다.

테스트 화면은 아래와 같습니다.


![Screenshot_1611568870](https://user-images.githubusercontent.com/55625423/105690823-cb1a2a00-5f3f-11eb-9cac-fb51e4f1bb98.png){: width="400" height="400"}


![Screenshot_1611568878](https://user-images.githubusercontent.com/55625423/105690826-cc4b5700-5f3f-11eb-8ef3-a2fe2ef58a20.png){: width="400" height="400"}

테스트 케이스
1. 현재 엑티비티의 recyclerView 아이템 갯수
2. 각 그림의 위의 recyclerView의 처음, 마지막 text
3. 각 그림의 우측의 LinearLayout의 처음, 마지막 text




혹시 targetSdk >= 29 일 경우 에러가 나오는데 테스트 클래스 이름 위에 @Config(sdk = [28]) 같이 설정을 하면 별 지장 없이 테스트 가능합니다.

여러 클래스 테스트를 할 경우 매번 이 @Config를 설정하는게 귀찮은 작업이 되는데 그럴 경우 app/src/test/resources 로 디렉토리를 만들어 resources안에 아래와 같이 간단한 파일을 만들면 모든 (local)테스트에서 @Config 사용 없이 Robolectric 사용이 가능합니다.

resources/robolectric.properties

    sdk=28



또 테스트 마다 나오는 아래와 같은 경고는 그냥 무시하셔도 됩니다.

<span style="color:red">
    [Robolectric] WARN: Android SDK 29 requires Java 9 (have Java 8). Tests won't be run on SDK 29 unless explicitly requested.
</span>

___

우선 위의 테스트에 앞서 해당 화면(activity or fragment)에 필요한 세팅을 해줍니다.
```kotlin
@RunWith(AndroidJUnit4::class)
class DicActivityTest {

    //테스트에 필요한 값들을 지정해 줍니다.
    val fileName = "창세기_1_dic.txt"

    val intent = Intent(
            //테스트용 Context입니다.
            ApplicationProvider.getApplicationContext(),
            DicActivity::class.java
    ).apply {
        putExtra(JEOL, "1")
    }

    //여러 테스트에 사용하기 위해 프로퍼티로 생성합니다.
    //구현 코드에서 보이듯 launchActivity는 ActivityScenario를 반환합니다.
    //public inline fun <reified A : Activity> launchActivity(
    //                                      intent: Intent?,
    //                                      activityOptions: Bundle?
    //                                  ): ActivityScenario<A>
    val scenario by lazy { launchActivity<DicActivity>(intent) }

    //위의 scenario는 rule을 통해서도 얻을 수 있습니다.
    //단 @get:Rule 은 @Before 전에 실행되니 setUp()의 설정은 적용되지 않습니다.
    //@get:Rule val rule = ActivityScenarioRule<DicActivity>(intent)

    @Before
    fun setUp() {
        //adapter을 사용할 경우 아래 코드가 없으면 
        //테스트 실행중 onCreateViewHolder() 에서 에러가 납니다.
        //viewHolder가 다 그려지기 전에 실행 하려해서 에러가 나옵니다.
        Robolectric.getForegroundThreadScheduler().pause()
        // Robolectric.getBackgroundThreadScheduler().pause()
        //... 기타 필요 세팅
    }
    //...

}
```


### RecyclerView 아이템 갯수 테스트
```kotlin
@Test
fun genOneJangOneJeolDictionaryLenTest() {
    scenario.onActivity { dic ->
        val recyclerview = dic.findViewById<RecyclerView>(R.id.dic_recycler)
        val itemCount = recyclerview.adapter!!.itemCount
        assertThat(itemCount).apply {
            isEqualTo(6)
        }
    }
}
```

### RecyclerView의 처음,마지막 text 테스트
```kotlin
@Test
fun genOneJangOneJeolFirstChineseTextTest() {
    scenario.onActivity { dic ->
        val recyclerview = dic.findViewById<RecyclerView>(R.id.dic_recyclerView)
        val firstChild = recyclerview.getChildAt(0)
        val firstVH = recyclerview.findContainingViewHolder(firstChild)
        val textView = firstVH!!.itemView.findViewById<TextView>(R.id.words)
        val text = textView.text
        val firstLineText = text.split("\n")[0]
        assertThat(firstLineText).apply {
            println(firstLineText)
            assertThat(firstLineText).isEqualTo("太클 태")
        }
    }
}

@Test
fun genOneJangOneJeolLastChineseTextTest() {
    scenario.onActivity { dic ->
        val recyclerview = dic.findViewById<RecyclerView>(R.id.dic_recycler)
        val lastIndex = recyclerview.adapter!!.itemCount - 1

        
        recyclerview.scrollToPosition(lastIndex)
        //위의 scrollToPosition() 작업을 하고 기다려주는 역할을 합니다.
        //View가 다시 그려지기 때문입니다.
        shadowOf(Looper.getMainLooper()).idle()

        val firstChild = recyclerview.getChildAt(recyclerview.childCount - 1)
        val firstVH = recyclerview.findContainingViewHolder(firstChild)
        val textView = firstVH!!.itemView.findViewById<TextView>(R.id.words)
        val text = textView.text
        val firstLineText = text.split("\n")[0]
        assertThat(firstLineText).apply {
            println(firstLineText)
            assertThat(firstLineText).isEqualTo("造지을 조, 이를 조")
        }
    }
}
```

### 우측의 LinearLayout의 처음, 마지막 text
```kotlin
@Test
fun scrollLinearLayoutTextTest() {
    scenario.onActivity { dic ->
        val scrollLinearLayout = 
                dic.findViewById<LinearLayout>(R.id.dic_fast_scroll)
        val childViewCount = scrollLinearLayout.childCount
        assertThat(childViewCount).isEqualTo(6)

        val firstTextView = scrollLinearLayout.getChildAt(0) as TextView
        val lastTextView = scrollLinearLayout.getChildAt(childViewCount - 1) as TextView

        assertThat(firstTextView.text).isEqualTo("太")
        assertThat(lastTextView.text).isEqualTo("造")
    }
}
```


이런 식으로 테스트를 해주면 추후에 코드 수정이나 업데이트시 유용한 안전막이 됩니다.

<br>

____

<br>

이 정도만 봐도 Robolectric 을 대충 어떻게 사용할 수 있는지 알 수 있을 것입니다.

이렇게 테스트하다보면 동작 반응에 대해서도 테스트를 하고 싶은 욕구가 나옵니다.

그러나 그런 테스트는 Espresso가 적절하겠습니다.


















