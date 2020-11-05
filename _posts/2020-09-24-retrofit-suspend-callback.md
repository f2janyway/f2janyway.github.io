---
layout: posts
title: "Retrofit2에서 suspend func Call을 CallAdapter을 이용해 응답을 처리하는 방법"
comments: true
categories:
    - android
    - kotlin
tags:
    - android
    - kotlin
    - retrofit2
# pagination:
#     enabled: true
date: 2020-09-24T12:34:00Z
---

[proandroiddev(medium) 자세한 설명은 여기서](https://proandroiddev.com/create-retrofit-calladapter-for-coroutines-to-handle-response-as-states-c102440de37a){:target="_blank"}

이 글에서 NetworkResponse~.kt 파일들의 코드를 거의 저 위의 블로그에서 복붙했다고 볼 수 있다.

돌아가는 것을 보기 위해 github-api를 이용하고 아주 간략하게 data class를 만들어 적용해 봤다.

```kotlin
data class Git(
    val name:String,
    val full_name:String,
    val owner: Owner
)
data class Owner(
      val login:String? = null,
      val url:String? = null
)
```

아직도 NetworkResponse~.kt들을 전부 이해하진 못했다. 그러나 실습 해보면서 제대로 작동하는 코드를 만들어가는 경험은 유익하다.

<br>

### 1. Service 인터페이스 

그냥 suspend 만 붙이는 경우

```kotlin
 @GET("users/{user}/repos")
    suspend fun getUser(@Path("user") user:String): Call<List<Git>>
```

이렇게 할 경우 서버와 통신중 에러가 나면 Exception이 난다. 그래서 try-catch로 응답에 대응해야 한다.
<br>
<br>
Call -> NetworkResponse 변경

```kotlin
interface Service{
    @GET("users/{user}/repos")
    suspend fun getUser(@Path("user") user:String): NetworkResponse<List<Git>, Error>

    companion object{
        val service = Retrofit.Builder().baseUrl("https://api.github.com/")
            .addConverterFactory(GsonConverterFactory.create())
            .addCallAdapterFactory(NetworkResponseAdapterFactory())
            .build()
    }
}
```

### 2. NetworkResponse sealed 클래스 

```kotlin
sealed class NetworkResponse <out T:Any,out U:Any>{
    /**
     * Success response with body
     * */
    data class Success<T:Any>(val body:T): NetworkResponse<T, Nothing>()
    /**
     *  Failure response with body
     */
    data class ApiError<U:Any>(val body :U, val code:Int): NetworkResponse<Nothing, U>()
    /**
     * Network Error
     * */
    data class NetworkError(val error:IOException): NetworkResponse<Nothing, Nothing>()
    /**
     * For example, json parsing error
     * */
    data class UnknownError(val error: Throwable?): NetworkResponse<Nothing, Nothing>()
}
```

<details>
<summary>sealed class 간략 설명</summary>

- sealed modifier 을 이용해 클래스의 계층을 제한할 때 쓰인다.
- enum과 유사하다.
- 여러 객체를 가질수 있다.(enum은 object;static 객체 하나만 존재)
- 상태값(value)을 넣고 사용할 수 있다.
- when 사용시 편하다.


```kotlin

enum class Animal{
    CAT,DOG,BIRD
}   

```
위의 enum class와 아래의 sealed class는 같은 기능을 한다.

sealed class에서 전부 object만 이용할 거면 enum을 쓰는게 kotlin개발자들의 의도이지 않을까 싶다.
```kotlin
sealed class Animal
object CAT:Animal()
object DOG:Animal()
object BIRD:Animal()
```
그런데 위의 방식과 아래의 방식에서 object들은 다른 object이다.
```kotlin
sealed class Animal{
    object CAT
    object DOG
    object BIRD
}
```

kotlin doc example
```kotlin
sealed class Expr
data class Const(val number: Double) : Expr()
data class Sum(val e1: Expr, val e2: Expr) : Expr()
object NotANumber : Expr()
```

```kotlin
fun eval(expr: Expr): Double = when(expr) {
    is Const -> expr.number
    is Sum -> eval(expr.e1) + eval(expr.e2)
    NotANumber -> Double.NaN
    // the `else` clause is not required because we've covered all the cases
}
```
</details>

<details>
`
    <summary>in(반공변성), out(공변성) 간략 설명</summary>
- in, out 은 제네릭을 사용할 때 쓰인다.
- \<in T> 와 \<out T>는 반대 기능을 한다고 생각하자.(당연하지만)
- \<in T> 은 T 안(하위계층)의 class들만(T포함) 가질 수 있다.
- \<out T> 은 T 밖(상위계층)의 class들만(T포함) 가질 수 있다.

```kotlin
class Home<in T>
open class Parent
class Child():Parent()

fun main(){
    val parentHome: Home<Parent> = Home<Child>() << compile error
    val childHome:Home<Child> = Home<Parent>() << OK
}
```

```kotlin
class Home<out T>
open class Parent
class Child():Parent()

fun main(){
    val parentHome: Home<Parent> = Home<Child>() << OK
    val childHome:Home<Child> = Home<Parent>() << compile error
}
```

</details>


<br>

### 3. NetworkResponseCall 클래스

```kotlin
internal class NetworkResponseCall<S:Any,E:Any>(
    private val delegate: Call<S>,
    private val errorConverter: Converter<ResponseBody,E>
) :Call<NetworkResponse<S, E>>{
    override fun clone(): Call<NetworkResponse<S, E>> = 
        NetworkResponseCall(delegate.clone(), errorConverter)

    override fun execute(): Response<NetworkResponse<S, E>> = 
        throw UnsupportedOperationException("NetworkResponseCall doesn't support execute")

    override fun enqueue(callback: Callback<NetworkResponse<S, E>>) {
        return delegate.enqueue(object : Callback<S>{
            override fun onResponse(call: Call<S>, response: Response<S>) {
                val body = response.body()
                val code = response.code()
                val error = response.errorBody()
                if(response.isSuccessful){
                    if(body != null){
                        callback.onResponse(this@NetworkResponseCall,Response.success(
                            NetworkResponse.Success(body)
                        ))
                    }else{
                        // response is successful but the body is null
                        callback.onResponse(this@NetworkResponseCall,Response.success(
                            NetworkResponse.UnknownError(null)
                        ))
                    }
                }else{
                    val errorBody = when{
                        error == null -> null
                        error.contentLength() == 0L -> null
                        else ->try{
                            errorConverter.convert(error)
                        }catch (e:Exception) {
                            null
                        }
                    }
                    if(errorBody != null){
                        callback.onResponse(this@NetworkResponseCall, Response.success(
                            NetworkResponse.ApiError(errorBody, code)
                        ))
                    }else{
                        callback.onResponse(this@NetworkResponseCall, Response.success(
                            NetworkResponse.UnknownError(null)
                        ))
                    }
                }
            }

            override fun onFailure(call: Call<S>, t: Throwable) {
                val networkResponse = when(t){
                    is IOException -> NetworkResponse.NetworkError(t)
                    else-> NetworkResponse.UnknownError(t)
                }
            }

        })
    }

    override fun isExecuted(): Boolean =delegate.isExecuted

    override fun cancel(): Unit =delegate.cancel()

    override fun isCanceled(): Boolean =delegate.isCanceled

    override fun request(): Request =delegate.request()

    override fun timeout(): Timeout =delegate.timeout()

}
```

### 4. NetworkResponseAdapter 클래스

```kotlin
class NetworkResponseAdapter <S:Any,E:Any>(
    private val successType:Type,
    private val errorBodyConverter:Converter<ResponseBody,E>
):CallAdapter<S, Call<NetworkResponse<S, E>>>{
    override fun responseType(): Type = successType

    override fun adapt(call: Call<S>): Call<NetworkResponse<S, E>> = 
        NetworkResponseCall(call,errorBodyConverter)
}
```

### 5. NetworkResponseAdapterFactory 클래스

```kotlin
class NetworkResponseAdapterFactory : CallAdapter.Factory() {

    override fun get(
        returnType: Type,
        annotations: Array<Annotation>,
        retrofit: Retrofit
    ): CallAdapter<*, *>? {

        // suspend functions wrap the response type in `Call`
        if (Call::class.java != getRawType(returnType)) {
            return null
        }

        // check first that the return type is `ParameterizedType`
        check(returnType is ParameterizedType) {
            "return type must be parameterized as Call<NetworkResponse<<Foo>> or Call<NetworkResponse<out Foo>>"
        }

        // get the response type inside the `Call` type
        val responseType = getParameterUpperBound(0, returnType)
        // if the response type is not ApiResponse then we can't handle this type, so we return null
        if (getRawType(responseType) != NetworkResponse::class.java) {
            return null
        }

        // the response type is ApiResponse and should be parameterized
        check(responseType is ParameterizedType) { "Response must be parameterized as NetworkResponse<Foo> or NetworkResponse<out Foo>" }

        val successBodyType = getParameterUpperBound(0, responseType)
        val errorBodyType = getParameterUpperBound(1, responseType)

        val errorBodyConverter =
            retrofit.nextResponseBodyConverter<Any>(null, errorBodyType, annotations)

        return NetworkResponseAdapter<Any, Any>(successBodyType, errorBodyConverter)
    }
}
```