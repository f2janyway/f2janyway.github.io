---
title: 'Retrofit2 suspend 유닛 테스트 [2]'
categories:
    - android
tags:
    - android
    - test
    - retrofit2
---

[이전 글]({% post_url 21-01-26-test-retrofit2_call %})에서는 기본적인 Call\<T>로 이루어진 request를 테스트하는 법을 봤습니다.

이번에는 suspend를 이용해 response를 받아오는 테스트입니다. 

- 기본 세팅은 전 글에서와 같으니 참고하시길 바랍니다.
- 간략히 구현 코드를 보겠습니다.

[MainViewModel]({% post_url 21-01-26-test-retrofit2_call %}#mainviewmodel)
```kotlin
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
```

- suspend를 사용하면 Call을 하용하지 않고 DTO(User)를 바로 받아올 수 있습니다.

[Service]({% post_url 21-01-26-test-retrofit2_call %}#service)
```kotlin
@GET("retrofit_test/user{id}")
suspend fun getUserByIdSuspend(@Path("id") id:Int):User?
```

- 우선 mockWerSer 테스트를 하겠습니다.
- 이 전의 테스트와 차이점은 단지 runBlocking으로 coroutine을 사용하는것 뿐입니다.

```kotlin
@Test
fun `suspend getUserIdOne test`() = runBlocking {
    server.enqueue(userOneResponse)
    mainViewModel.loadUserByIdSuspended(1, testService)

    mainViewModel.user.getOrAwaitValue().let { result ->
        assertThat(result).isInstanceOf(Result.Success::class.java)

        val user = (result as Result.Success).data
        println(user)
        assertThat(user.id).isEqualTo(1)
        assertThat(user.name).isEqualTo("Moreno Figueroa")
    }
}
```
- 테스트 결과 입니다.
- User의 값을 잘 받아오고 있습니다.

![화면 캡처3](https://user-images.githubusercontent.com/55625423/105953507-9e3c5300-60b6-11eb-87a1-e074c1e46e71.png)


- 실제 서버에 요청을 해보겠습니다.
- 테스트와 다르게 gitHub의 user는 name : "@Moreno Figueroa@" 이렇게 해놓았습니다.

```kotlin
@Test
fun `suspend real getUserIdOne test`() = runBlocking {
    //위의 테스트에서 server.enqueue()가 빠졌습니다.
    //실제 서버에 도달할 baseUrl을 가진 service를 넣어줍니다. 
    mainViewModel.loadUserByIdSuspended(1, getService())

    mainViewModel.user.getOrAwaitValue().let { result ->
        assertThat(result).isInstanceOf(Result.Success::class.java)
        val user = (result as Result.Success).data
        println(user)
        assertThat(user.id).isEqualTo(1)
        assertThat(user.name).isEqualTo("@Moreno Figueroa@")
    }
}
```

- 테스트 결과입니다.
- name이 변경된것을 볼 수 가 있습니다.

![화면 캡처 5png](https://user-images.githubusercontent.com/55625423/105954264-cb3d3580-60b7-11eb-9efc-48d35478fa6c.png)

- 이번에는 에러 케이스를 테스트 해보겠습니다.
  - HttpURLConnection.HTTP_NO_CONTENT = 204


```kotlin
@Test
fun `suspend getUserIdOne fail test`() = runBlocking {
    server.enqueue(MockResponse().apply {
        setResponseCode(HttpURLConnection.HTTP_NO_CONTENT)
    })
    mainViewModel.loadUserByIdSuspended(1, testService)
    mainViewModel.user.getOrAwaitValue().let { result ->
        assertThat(result).isInstanceOf(Result.Error::class.java)

        val message = (result as Result.Error).e.message
        println(message)
    }
}

```
- 테스트 결과입니다.

![화면 캡처suspend_e](https://user-images.githubusercontent.com/55625423/105960683-ec565400-60c0-11eb-8326-7ba143b4ceb3.png)

- suspend를 사용 할 경우 try/catch를 잘 해주어야 합니다.

___ 

이상 suspend를 이용한 request 테스트를 보았습니다.

사실 이전 글에서 테스트들과 크게 다를게 없습니다. Coroutine만 사용하고 그리고 try/catch를 사용한다 뿐입니다.

만약 try/catch를 사용하지 않고 retrofit에서 suspend를 사용하는 방법이 궁금하시다면 전에 [작성한 이 글]({% post_url 20-09-24-retrofit-suspend-callback %})을 보시면 도움이 될 것 같습니다.

___


    “Tests are stories we tell the next generation of programmers on a project.”
    
     — Roy Osherove