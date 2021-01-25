---

# published: false
title: "[안드로이드] ViewModel,LiveData 테스트하기"
comments: true
# pagination:
#     enabled: true
categories:
    - android
    - test
tags:
    - android
date: 2020-12-08T01:15:00Z
---

### Before

AndroidX Test
- 안드로이드 테스트를 위한 라이브러리
- local or instrumented 모두 가능
  - local에서 사용시 시뮬레이션 환경을 제공(Robolectric의 도움 받음)
  - instrumented에서 사용시 실제 환경 값(context etc)들을 이용
- local에서 사용사 Robolectric library 필요
  
LiveData 테스트를 위해
1. InstantTaskExecutorRule
   - Junit Rule
   - 모든 AAC 관련 background-job들을 한 Thread에서 실행하게 하고 결과는 동기적으로 나온다.
2. liveData.observe()를 이용
<br>

>testImplementation "androidx.arch.core:core-testing:$archTestingVersion"


이번 테스트는 local-test 다.

UI-test가 아니다.

android doc을 여기저기 보다보면 isolate(tion)이라는 단어 많이 나온다. 그리고 내용의 맨 밑에는 test 예시에 대해 나온것들이 많이 있다.

테스트하기 쉬운 코드를 짜는것이 isolated(고립된)한 코드라는 말이라고 이해한다.

(디커플과 같은 의미겠다.)

이는 익히 알려져 있듯이 유지보수에 유리하고 읽기쉬운 코드가 될 확률이 높다.
(사실 본인은 아직 의식적으로 클린코드로 나름대로 작성하려 해도 본래의 습관이 나오는 상태다...훈련이 덜 된거다.)


ViewModel도 테스트하기 좋게 구성을 해야한다.

___

<br>

### Setting

ViewModel test시 안드로이드 컴포넌트(Context 등)가 필요할 경우 아래의 의존성들을 추가해주자.
    
    //build.gradle(.)
    ext{
        ...
        androidXTestCoreVersion = '1.2.0'
        androidXTestExtKotlinRunnerVersion = '1.1.2'
        robolectricVersion = '4.3.1'
    } 
최신버전에 맞게 설정한다.

    //build.gradle(.app)
    android{
        // Always show the result of every unit test when running via command line, even if it passes.
        testOptions.unitTests {
            includeAndroidResources = true
            //keep your unit tests running as you add idling resource code to your application code.
            returnDefaultValues = true
        }
    }
    dependencies {
        ...
        // AndroidX Test - JVM testing
        testImplementation "androidx.test.ext:junit-ktx:$androidXTestExtKotlinRunnerVersion"
        testImplementation "androidx.test:core-ktx:$androidXTestCoreVersion"
        testImplementation "org.robolectric:robolectric:$robolectricVersion"

        testImplementation "org.jetbrains.kotlinx:kotlinx-coroutines-test:1.4.2"
    }

<br><br>
ViewModel에서는 주로 repository 와 연결된 일을 주로 처리시켜놨다.
그래서 ViewModel을 테스트한다는 뜻은 repository와 그 안의 room을 테스트하는 모양이 된다.

각 기능별 테스트가 필요하겠으나 이 예제에서는 ViewModel에서 전부(repository,room)를 테스트한다.

아래의 @RunWith은 말 그대로 AndroidJUnit4를 runner로 쓰겠다는 어노테이션이다.
(Junit이 가지고 있는 built-in runner가 아닌)

### Test
```kotlin
//@Config targetSdk가 29 이상일 경우는 Robolectric에서 에러나는것을 방지
@Config(sdk = [Build.VERSION_CODES.O_MR1])
@ExperimentalCoroutinesApi //coroutine 관련 api사용시 필요
@RunWith(AndroidJUnit4::class)
class MainViewModelTest {
    //아래는 다 테스트를 위한 준비다.

    private lateinit var localDataSource: DataSource
    private lateinit var db: TimerDatabase
    private lateinit var repository: Repository

    // Set the main coroutines dispatcher for unit testing.
    @ExperimentalCoroutinesApi
    @get:Rule
    var mainCoroutineRule = MainCoroutineRule()

    // Executes each task synchronously using Architecture Components.
    @get:Rule
    var instantExecutorRule = InstantTaskExecutorRule()

    @Before
    fun createDB() {
        val con = ApplicationProvider.getApplicationContext<Context>()
        //테스트를 위해 inMemoryDatabaseBuilder와  .allowMainThreadQueries()를 사용한다.
        db = Room.inMemoryDatabaseBuilder(con, TimerDatabase::class.java)
            .allowMainThreadQueries().build()
    }

    //DB를 닫아준다.
    @After
    fun closeDb() = db.close()

    @Before
    fun createRepository() {
        //Fake~ 클래스 타입이 인터페이스이고 Fake~는 구현체다
        localDataSource = FakeDataSource(dao = db.timerDao())
        repository = FakeRepository(localDataSource)
    }
    
}
```
참고
 - InstantTaskExecutorRule
   - InstantTaskExecutorRule is a JUnit Rule. When you use it with the @get:Rule annotation, it causes some code in the InstantTaskExecutorRule class to be run before and after the tests 
   - This rule runs all Architecture Components-related background jobs in the same thread so that the test results happen synchronously, and in a repeatable order. When you write tests that include testing LiveData, use this rule!
 
   - testImplementation "androidx.arch.core:core-testing:$archTestingVersion"



여기까지는 @Before 즉 테스트 전에 실행되는 코드들이다.

테스트는 이제 쭉 필요한 만큼 사용하면 된다.

```kotlin
//runblockingTest라는 테스트용이 있으나 버그가 좀 있다. 그래서 그냥 
//coroutine을 사용하므로 runblocking을 사용한다. delay()도 잘 작동한다.
@Test
fun observeTimersTest() = runblocking{
    val tiemr1 = Tiemr(id = 0, totalSecond = 10)
    val tiemr2 = Tiemr(id = 1, totalSecond = 20)
    val tiemr3 = Tiemr(id = 2, totalSecond = 30)

    repository.addTimers(timer1, timer2, timer3)

    //여기서 getOrAwaitValue()이란 확장함수가 나오는데 
    //liveData 의 시간차를 두고 observe하게 해준다. 
    //구글 codelab이나 medium에서 그냥 사용하라고 하는데 
    //구현코드를 보면 대충은 감이 오지 않을까 싶다. 
    //아래 참고
    val timerList = repository.observeTimers().getOrAwaitValue().data
    assertThat(3,`is`(timerList.count()))
    assertThat(10,`is`(timerList[0].totalSecond))

    repository.deleteTimerById(0)
    val twoTimerList = repository.observeTimers().getOrAwaitValue().data
    assertThat(2,`is`(twoTimerList.count()))
    //...이런식으로 계속 조작해가며 다른 테스틀르 만들어주면 되겠다.

}
```

[android developer medium 참고 링크](https://medium.com/androiddevelopers/unit-testing-livedata-and-other-common-observability-problems-bb477262eb04){:target="_blank"} <br>
[codelab 참고](https://developer.android.com/codelabs/advanced-android-kotlin-training-testing-basics/#8){:target="_blank"}


```kotlin
@VisibleForTesting(otherwise = VisibleForTesting.NONE)
fun <T> LiveData<T>.getOrAwaitValue(
    time: Long = 2,
    timeUnit: TimeUnit = TimeUnit.SECONDS,
    afterObserve: () -> Unit = {}
): T {
    var data: T? = null
    val latch = CountDownLatch(1)
    val observer = object : Observer<T> {
        override fun onChanged(o: T?) {
            data = o
            latch.countDown()
            this@getOrAwaitValue.removeObserver(this)
        }
    }
    this.observeForever(observer)

    try {
        afterObserve.invoke()

        // Don't wait indefinitely if the LiveData is not set.
        if (!latch.await(time, timeUnit)) {
            throw TimeoutException("LiveData value was never set.")
        }

    } finally {
        this.removeObserver(observer)
    }

    @Suppress("UNCHECKED_CAST")
    return data as T
}
```

아래에는 위의 테스트 이해를 위한 구현체들이 있다.


ViewModel은 이렇게 repository를 의존하고
```kotlin
MainViewModel(private val repository:Repository):ViewModel(){}
```
Repository는 dataSource를 의존하고
```kotlin
class TimerRepository(private val localDataSource:TimerLocalDataSoure):Repository{}

//테스트하기 쉽게 Repository interface를 만들어 추상화 해준다.
interface Repository {}

```

DataSource는 dao를 의존한다.
```kotlin
class TimerLocalDataSource (
    private val timerDao: TimerDao
):DataSource

//Repository와 같이 DataSource interface로 빼준다.
interface DataSource {}
```


interface 로 추상화를 하는 방식이 좀 귀찮다고 느낄수도 있다. 인터페이스 변경이 있을 때마다 계속 변경을 해주어야 하기 때문인데 나중에 메서드가 늘어나거나 또는 다른 인터페이스(의존성)를 추가할 경우 관리가 수월해지고 역시 테스트도 편리해지는 장점이 더 클것이다. 그리고 테스트에서는 FakeTimerRepository를 사용한다. 그래서 interface가 있으면 편리하다. 

그리고 dagger을 사용하면 더 편하겠다.(이 예제에서는 다루지 않는다.)


FakeRepository는 위의 TimerRepository와 같은 구현체라 보면 되겠다.
Fake를 일부만 테스트할 수 있으니 유용하다. 뭐 전부를 테스트하려면 
TimerRepository로 테스트해도 되겠다.
```kotlin
class FakeRepository(val localSource: DataSource):Repository {}
```
<br>
이 한 경우( observeTimers() )를 테스트하는 것을 보겠다. 그리고 나중에 저 List<Timer> 를 Wrapper 클래스로 감싸주는데 그 부분은 생략한다. 네트워크(retrofit) response를 감싸주는 것과 같은 방식이니 뭐 구글링하면 많이 나온다.

```kotlin
@Dao
interface TimerDao {
    @Query("SELECT * FROM timers")
    fun observeTimers(): LiveData<List<Timer>>
    ...
}    
```
<br>
그리고 DataSource에 메서드를 추가하고 TimerLocalDataSource에 구현하고 repository도 똑같이 구현하주고 하면 이런 모양이 나온다.

```kotlin
interface DataSource {
    fun observeTimers(): LiveData<List<Timer>>
    //...
}    
// TimerLocalDataSource 
override fun observeTimers(): LiveData<List<Timer>> {
    //여기서 Wrapper로 감싸준다. 예를 들면 
    //위의 리턴 타입이  LiveData<Result<List<Timer>>>
    //아래의 리턴 값은  timerDao.observeTimers().map{ Success(it) }
    //필요에 따라 조건을 추가해서 요런식으로 해주면 되겠다.
    return timerDao.observeTimers()
}

interface Repository {
    fun observeTimers():LiveData<List<Timer>>
    //...
}
//TimerRepository
override fun observeTimers(): LiveData<List<Timer>> {
    return timerLocalDataSource.observeTimers()
}

//MainViewModel
private lateinit var _timers: LiveData<List<Timer>>
val timers: LiveData<List<Timer>>
        get() = _timers

init{
    viewModelScope.launch(Dispatchers.IO) {
        _timers = repository.observeTimers()
            .distinctUntilChanged()
            // Wrapper 사용한 경우
            // .switchMap { filterTimer(it) }
    }
}

```

<br>

____


안드로이드 테스트는 쉽지가 않다...<br>
쉽지가 않어.<br>
...<br>
그냥 훈련이 안되서 그런건가?<br>
모르겠다. 어쨌든 습관들이기를 계속해야겠다.<br>


참고
- [코드랩1](https://developer.android.com/codelabs/advanced-android-kotlin-training-testing-basics#7){:target="_blank"}
- [코드랩3](https://developer.android.com/codelabs/advanced-android-kotlin-training-testing-survey#2){:target="_blank"}
- [안드로이드 medium](https://medium.com/androiddevelopers/unit-testing-livedata-and-other-common-observability-problems-bb477262eb04){:target="_blank"}

