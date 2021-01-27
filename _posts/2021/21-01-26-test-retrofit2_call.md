---
title: 'Retrofit2 유닛 테스트 [1]'
categories:
    - android
tags:
    - android
    - test
    - retrofit2
---

<br>

Retrofit2를 테스트 하는 방법을 공유합니다. 

기본적인 Retrofit2 사용법은 알고 있다는 전제하에 쓴 글입니다.


<br>

## Retrofit2 Unit Test

### Test Case
1. Call\<T>을 이용할 경우 테스트
2. Suspend을 이용할 경우 테스트([다음 글]({% post_url 21-01-27-test-retrofit2_suspend %}))


이 글에서는 1번만 다룹니다. 그리고 
- json 파일로 response를 만들어서 하는 테스트
- 실제 서버에서 response를 불러오는 테스트
  
위와 같은 경우를 진행합니다.


- 테스트에 필요한 의존성을 추가합니다.

build.gradle(:app)
``` 
testImplementation 'androidx.test.ext:junit:1.1.2'
testImplementation 'androidx.test.ext:truth:1.3.0'
testImplementation 'com.squareup.okhttp3:mockwebserver:4.7.2'

//androidx.arch.core.executor.testing.InstantTaskExecutorRule 사용을 위함
testImplementation 'androidx.arch.core:core-testing:2.1.0'
testImplementation 'org.robolectric:robolectric:4.3.1'
testImplementation 'org.jetbrains.kotlinx:kotlinx-coroutines-test:1.4.2'

implementation 'com.squareup.retrofit2:retrofit:2.9.0'
implementation 'com.squareup.retrofit2:converter-gson:2.9.0'
implementation 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.2.0'
```

- robolectric 실행 중 targetSdk관련 에러가 나오면 [여기를 보세요(살짝 윗부분부터)]({% post_url 21-01-25-test-robolectric %}/#resourcesrobolectricproperties)

- 테스트에 필요한 응답값들을 .json 형태로(파일 형식이 중요하지 않지만) 준비합니다.
  - [여기서 json 파일을 생성했습니다.](https://www.json-generator.com/){:target="_blank"}
-  위에서 준비한 파일(들)을 app/src/test/resources 디렉토리를 만들고 거기에 넣어줍니다. 

이 글에서 사용하는 응답값은

resources/success.json
```json
{
  "total_count": 3,
  "users": [
    {
      "id": 1,
      "name": "Moreno Figueroa",
      "url": "https://raw.githubusercontent.com/f2janyway/rest_test/main/mp4/home_rain.mp4"
    },
    {
      "id": 2,
      "name": "Earlene Harding",
      "url": "https://raw.githubusercontent.com/f2janyway/rest_test/main/mp4/home_rain.mp4"
    },
    {
      "id": 3,
      "name": "Christa Cote",
      "url": "https://raw.githubusercontent.com/f2janyway/rest_test/main/mp4/road_rain.mp4"
    }
  ]
}
```
userId1.json
```json
{
  "id": 1,
  "name": "Moreno Figueroa",
  "url": "https://raw.githubusercontent.com/f2janyway/rest_test/main/mp4/home_rain.mp4"
}
```
userId2.json
```json
{
  "id": 2,
  "name": "Earlene Harding",
  "url": "https://raw.githubusercontent.com/f2janyway/rest_test/main/mp4/home_rain.mp4"
}
```

- 기타 잘못된 응답의 json파일을 만들어 테스트를 해보면 좋습니다.
- 해당 data class들을 만들어줍니다.
- 이 예제는 '프로퍼티명 == json의 키값' 이어서 문제가 없지만 ``같지 않을 경우`` ``@field:SerializedName("total_count") val totalCount:Int``  이런 식으로 필드를 설정합니다.
  
```kotlin
data class Group(
    val total_count: Int,
    val users: List<User>
)

data class User(
    val id: Int,
    val name: String,
    val url: String
)
```

- 본격적으로 테스트를 작성합니다.

```kotlin
@RunWith(AndroidJUnit4::class)//robolectric을 사용
class GetGroupTest {
    //liveData를 테스트하기 위해 필요합니다.
    //이걸 사용하지 않을 경우
    //_data.value = result 를 사용하지 못하고
    //_data.postValue(result)를 사용해야 합니다. 
    @Rule
    @JvmField
    val instantExecutorRule = InstantTaskExecutorRule()

    private lateinit var server: MockWebServer

    private lateinit var mainViewModel : MainViewModel

    //ApplicationProvider.getApplicationContext()는 
    //robolectric에서 제공해줍니다. 
    private val context = ApplicationProvider.getApplicationContext<Context>()

    private lateinit var mockUrl :HttpUrl

    private lateinit var testService:Service
    //필요시 기타 setHeader() 등 여러 설정을 할 수 있습니다.
    val successResponse by lazy {
        MockResponse().apply {
            //HttpURLConnection.HTTP_OK 
            //이 부분에서 여러개 테스트케이스를
            //만들어서 해보세요
            //이 글에선 HTTP_BAD_REQUEST,HTTP_CLIENT_TIMEOUT
            //두 가지만 테스트합니다.
            setResponseCode(HttpURLConnection.HTTP_OK)
            
            //readSuccessJson()구현은 아래에 나옵니다.
            val jsonText = readSuccessJson(Context)
            setBody(jsonText)
        }
    }

    @Before
    fun setUp() {
        mainViewModel = MainViewModel()

        server = MockWebServer()
        server.start()
        mockUrl = server.url("/")

        //중요!! baseUrl은
        //server.url()의 반환값인 HttpUrl을 이용해야 합니다.
        //getRetrofitBuilder(), Service 구현은 아래 있습니다.
        testService = getRetrofitBuilder()
            .baseUrl(mockUrl)
            .build()
            .create(Service::class.java)
    }
    //...
}   
```

테스트에 사용하는 service 와 viewModel 그리고 Utils(readSuccessJson()) 구현은 아래와 같습니다.

### Service
```kotlin
interface Service {
    @GET("retrofit_test/group")
    fun getGroup(): Call<Group>

    @GET("retrofit_test/user{id}")
    fun getUserById(@Path("id") id:Int):Call<User>

    //다음 글에서 사용합니다.
    @GET("retrofit_test/user{id}")
    suspend fun getUserByIdSuspend(@Path("id") id:Int):User?

    companion object {
        //실제 개인 서버는 없기에 github를 이용합니다. 
        const val BASE_URL = "https://raw.githubusercontent.com/f2janyway/testRestApi/master/"

        //테스트를 위해 service와 builder을 나눠줍니다.
        //테스트에서는 다른 baseUrl을 사용합니다.
        private val retrofitBuilder:Retrofit.Builder = Retrofit
            .Builder()
            .addConverterFactory(
                GsonConverterFactory.create()
            )

        private val service:Service = retrofitBuilder
            .baseUrl(BASE_URL)
            .build()
            .create(Service::class.java)

        fun getService() = service
        fun getRetrofitBuilder() = retrofitBuilder


        //이 enqueue를 편하게 하기위해 제네릭 함수를 만들어 줍니다.
        internal fun <T> callBack(
            call: Call<T>,
            liveData: MutableLiveData<Result<T>>
        ) {
            call.enqueue(object : Callback<T> {
                override fun onResponse(call: Call<T>, response: Response<T>) {
                    println("onResponse ${call.request()}")
                    if (response.isSuccessful) {
                        if (response.body() != null)
                            liveData.value = (Result.Success(response.body()!!))
                        else
                            liveData.value = (Result.Empty)
                    }else{
                        liveData.value = (
                            Result.Error(
                                Exception("response is not successful, status: ${response.code()}")
                            )
                        )
                    }
                }

                override fun onFailure(call: Call<T>, t: Throwable) {
                    println("onFailure")
                    liveData.value = (Result.Error(Exception("network error : onFailure")))
                }
            })
        }
    }
}
```
### MainViewModel

```kotlin
class MainViewModel : ViewModel() {
    private val _group = MutableLiveData<Result<Group>>()

    val group: LiveData<Result<Group>>
        get() = _group


    private val _user = MutableLiveData<Result<User>>()
    val user: LiveData<Result<User>>
        get() = _user

    fun loadGroup(call:Call<Group> = getService().getGroup()) {
        callBack(call,_group)
    }
    fun loadUserById(id:Int, call: Call<User> = getService().getUserById(id)){
        callBack(call,_user)
    }
    //다음 글에서 사용합니다.
    fun loadUserByIdSuspended(id: Int, service: Service) = viewModelScope.launch {
        try {
            val rs = service.getUserByIdSuspend(id)

            if (rs == null) _user.value = Result.Empty
            else _user.value = Result.Success(rs)

        } catch (e: HttpException) {
            _user.value = Result.Error(e)
        } catch (e: Exception) {
            _user.value = Result.Error(e)
        }
    }
}
```
### Utils
```kotlin
fun readSuccessJson(context:Context):String{
    return fileReader(context,"success.json")
}
fun readEmptyJson(context: Context):String{
    return fileReader(context,"fail.json")
}
fun readUndefinedJson(context: Context):String{
    return fileReader(context,"unDefined.json")
}
fun readUser1(context: Context):String{
    return fileReader(context,"userId1.json")
}
fun readUser2(context: Context):String{
    return fileReader(context,"userId2.json")
}
//좀 밑에서 쓰입니다.
//GetUserByIdTest
fun getUserIdOneResponseWithSituation(httpStatus:Int,context: Context):MockResponse{
    return MockResponse().apply {
        setResponseCode(httpStatus)
        val jsonText = readUser1(context)
        setBody(jsonText)
    }
}
fun getUserIdTwoResponseWithSituation(httpStatus:Int,context: Context):MockResponse{
    return MockResponse().apply {
        setResponseCode(httpStatus)
        val jsonText = readUser2(context)
        setBody(jsonText)
    }
}
private fun fileReader(context: Context,fileName:String):String{
    // context.classLoader.getResource()로
    // json 위치를 참조합니다. (test/resources)
    val file = File(context.classLoader.getResource(fileName).file)
    return file.bufferedReader().use {
        val str = it.readText()
        it.close()
        str
    }
}
```
- 사전 준비가 많았습니다.
- 진짜 본격적으로 테스트 코드를 작성합니다.

```kotlin
@Test
fun `getGroup success test`(){
    //mockServer에 응답을 미리 넣어줍니다.
    server.enqueue(successResponse)
    //setUp()에서 준비한 testService로 getGroup()을 호출해줍니다.
    mainViewModel.loadGroup(testService.getGroup())
    //getOrAwaitValue()는 구글개발자가 liveData테스트 할 때 사용하라고 사용합니다.
    //코드를 보시면 liveData의 value를 관찰하여 한번만 사용할 수 있게 되어 있습니다.
    //아래 구현 링크있습니다.
    mainViewModel.group.getOrAwaitValue().let { result->
        assertThat(result).isInstanceOf(Result.Success::class.java)
        val group = (result as Result.Success).data
        assertThat(group).isInstanceOf(Group::class.java)
        assertThat(group.total_count).isEqualTo(3)
    }
}
```
[getOrAwaitValue()](https://gist.github.com/JoseAlcerreca/35828c25fca123c8a115d6251cf3f45b#file-livedatatestutil-kt){:target="_blank"}
- 테스트 결과는 아래와 같습니다.
- request를 보면 `url=http://127.0.0.1:59542/retrofit_test/group` 이렇게 mockServer가 만든 url을 볼수 있고  경로도 @Get getGroup()과 일치함을 볼 수 있습니다.

![화면 캡처 2021-01-27 143127](https://user-images.githubusercontent.com/55625423/105947418-68926c80-60ac-11eb-905a-6cc6c988d346.png){: width="1000" height="400"}



- 이번에는 연속해서 두 개의 호출을 하는 테스트를 보겠습니다.

```kotlin
@Test
fun `group success and empty test`() {
    // 위와 같은 방식입니다.
    server.enqueue(successResponse)

    mainViewModel.loadGroup(testService.getGroup())

    mainViewModel.group.getOrAwaitValue{}.let { result ->
        assertThat(result).isInstanceOf(Result.Success::class.java)

        val group = (result as Result.Success).data
        assertThat(group).isInstanceOf(Group::class.java)

        group.let {
            assertThat(it.total_count).isEqualTo(3)
            assertThat(it.users[0].id).isEqualTo(1)
            println("end1")
        }
    }

    //두번째 response를 넣어줍니다.
    //사실 맨 위에서
    // server.enqueue(successResponse)
    // server.enqueue(emptyResponse)
    //이렇게 해주어도 상관없습니다.
    //단지 넣은 순서대로 loadGroup()이 호출됩니다.
    //FIFO
    //emptyResponse의 body는 {} 이렇습니다. 
    server.enqueue(emptyResponse)
    mainViewModel.loadGroup(testService.getGroup())
    //sleep을 안시켜주면 liveData.value를 얻기전에 테스트가 끝나게 됩니다.
    Thread.sleep(1000)
    mainViewModel.group.getOrAwaitValue().let { result ->
        assertThat(result).isInstanceOf(Result.Success::class.java)

        val group = (result as Result.Success).data
        assertThat(group).isInstanceOf(Group::class.java)

        group.let {
            assertThat(it.total_count).isEqualTo(0)
            assertThat(it.users).isNull()
            println("end2")
        }
    }
}

```

- 아래와 같이 정상적으로 테스트가 성공했습니다.

![화면 캡처](https://user-images.githubusercontent.com/55625423/105948208-e86d0680-60ad-11eb-8921-76263c70c69e.png)

- 이번에는 실제 서버에 있는값을 테스트해보습니다.
  - 테스트에 사용한 코드 그대로 사용함을 보이기 위한 예시입니다.
- 위와 같은 방식의 설정으로 다른 테스트 클래스를 만들었습니다.
- GetUserByIdTest 에서 테스트를 하겠습니다.
  
```kotlin
@RunWith(AndroidJUnit4::class)
class GetUserByIdTest {
    //... 위와 동일

    private val userOneResponse by lazy {
        getUserIdOneResponseWithSituation(
            HttpURLConnection.HTTP_OK,
            context
        )
    }
    private val userOneResponseTimeout by lazy {
        getUserIdOneResponseWithSituation(
            HttpURLConnection.HTTP_CLIENT_TIMEOUT,
            context
        )
    }
    private val userOneResponseBadRequest by lazy {
        getUserIdOneResponseWithSituation(
            HttpURLConnection.HTTP_BAD_REQUEST,
            context
        )
    }
    private val userTwoResponse by lazy {
        getUserIdTwoResponseWithSituation(
            HttpURLConnection.HTTP_OK,
            context
        )
    }
    //...setUp()도 위와 동일


}    
```

- 우선 간단히 mockServer 테스트를 해봅니다.

```kotlin
@Test
fun `getUserIdOne and getUserIdTwo success test`() {
    //위와 똑같은 방식입니다.
    //다만 loadUserById(1, testService.getUserById(1)) 메서드를 바꿔 User를 가져옵니다.
    server.enqueue(userOneResponse)
    mainViewModel.loadUserById(1, testService.getUserById(1))

    mainViewModel.user.getOrAwaitValue {}.let { result ->
        assertThat(result).isInstanceOf(Result.Success::class.java)

        val userIdOne = (result as Result.Success).data
        println(userIdOne)
        assertThat(userIdOne).isInstanceOf(User::class.java)

        userIdOne.let {
            assertThat(it.id).isEqualTo(1)
            assertThat(it.name).isEqualTo("Moreno Figueroa")
        }
    }

    server.enqueue(userTwoResponse)
    mainViewModel.loadUserById(2, testService.getUserById(2))
    Thread.sleep(1000)
    mainViewModel.user.getOrAwaitValue {}.let { result ->
        assertThat(result).isInstanceOf(Result.Success::class.java)

        val userIdTwo = (result as Result.Success).data
        println(userIdTwo)
        assertThat(userIdTwo).isInstanceOf(User::class.java)

        userIdTwo.let {
            assertThat(it.id).isEqualTo(2)
            assertThat(it.name).isEqualTo("Earlene Harding")
        }
    }
}
```
- 테스트 결과 입니다.
- 실제서버 테스트(아래)와 비교하기 위한 사진입니다.
- 경로가 user1과 user2로 잘 나눠짐을 볼 수 있고
- User도 잘 받아옴을 볼 수 있습니다.


![화면 캡처 1](https://user-images.githubusercontent.com/55625423/105949448-0d627900-60b0-11eb-899f-573d6ff3ad69.png)

- 실제 서버를 테스트합니다.
  
```kotlin
@Test
fun `real network getUserIdOne and getUserIdTwo test`() {
    //server.enqueue() 할 필요가 없습니다.
    //viewModel 함수 그대로 사용합니다.
    //두번째 파라미터는 미리 선언되어서
    //기본 세팅 되어 있는 service를 사용합니다.
    //call: Call<User> = getService().getUserById(id)
    mainViewModel.loadUserById(1)
    mainViewModel.user.getOrAwaitValue().let {
        assertThat(it).isInstanceOf(Result.Success::class.java)
        val userIdOne = (it as Result.Success).data
        println(userIdOne)
        assertThat(userIdOne.id).isEqualTo(1)
    }

    mainViewModel.loadUserById(2)
    Thread.sleep(1000)
    mainViewModel.user.getOrAwaitValue().let { result ->
        assertThat(result).isInstanceOf(Result.Success::class.java)
        val userIdTwo = (result as Result.Success).data
        println(userIdTwo)
        assertThat(userIdTwo.id).isEqualTo(2)
    }
}

```
- 테스트 결과 입니다.
- 위의 mockServer테스트와 결과가 같고 url은 실제 url임을 볼 수 있습니다.
  
![화면 캡처2](https://user-images.githubusercontent.com/55625423/105949954-e5274a00-60b0-11eb-908e-9c2001fdd8ee.png)


- 이번에는 HttpURLConnection의 status를 변경해 예외 경우를 보겠습니다.
- TimeOut 에러를 넣어 보겠습니다.
  - HttpURLConnection.HTTP_CLIENT_TIMEOUT = 408 

```kotlin
@Test
fun `getUserIdOne http_client_timeout_test`(){

    server.enqueue(userOneResponseTimeout)
    mainViewModel.loadUserById(1,testService.getUserById(1))
    Thread.sleep(10000)
    mainViewModel.user.getOrAwaitValue ().let { result ->
        assertThat(result).isInstanceOf(Result.Error::class.java)
        val message = (result as Result.Error).e.message
        println(message)
        //isEqualTo의 문구는 위의 service 구현 부분 아래에 있습니다.
        assertThat(message).isEqualTo("network error : onFailure")
    }
}
```

- 테스트 결과입니다. onFailure()을 호출 했습니다.

![화면 캡처 timeout](https://user-images.githubusercontent.com/55625423/105958846-82d54600-60be-11eb-89b7-ae2649508628.png)

- 이번에는 HttpURLConnection.HTTP_BAD_REQUEST = 400 에러를 테스트 하겠습니다.

```kotlin
@Test
fun `getUserIdOne http_bad_req test`(){
    server.enqueue(userOneResponseBadRequest)
    mainViewModel.loadUserById(1,testService.getUserById(1))
    mainViewModel.user.getOrAwaitValue ().let { result ->
        assertThat(result).isInstanceOf(Result.Error::class.java)
        val message = (result as Result.Error).e.message
        println(message)
        assertThat(message).isEqualTo("response is not successful, status: ${HttpURLConnection.HTTP_BAD_REQUEST}")
    }
}
```
- 아래와 같이 onResponse()를 호출했습니다.
- 그러나 response.isSuccessful 은 false이고
- response는 null 입니다.

![화면 캡처 eror](https://user-images.githubusercontent.com/55625423/105959457-3fc7a280-60bf-11eb-904c-5c416ecfe2a4.png)


여러가지의 케이스를 테스트 해보시기 바랍니다.

____

이렇게 Retrofit2의 유닛 테스트를 할 수 있습니다.

이런 식으로 테스트 코드를 작성하면 testable(테스트 하기 쉬운)한 코드가 자연스레 만들어지는데

이런 경험을 해보면 테스트의 매력에 빠지게 됩니다.

다음 글은 위에서 나온 suspend function을 이용한 request를 테스트하는 방법을 보겠습니다.

___

    “Nothing makes a system more flexible than a suite of tests."
    -Uncle Bob Martin

____

참고
- [https://github.com/square/okhttp/tree/master/mockwebserver](https://github.com/square/okhttp/tree/master/mockwebserver){:target="_blank"}









