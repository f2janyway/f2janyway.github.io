---
layout: posts
# published: false
title: "안드로이드 Hilt 개념 설명 및 사용법[2]"
comments: true
# pagination:
#     enabled: true
categories:
    - android
tags:
    - android
    - hilt
date: 2020-12-14T01:15:00Z
---

[이전 글]({%  post_url 20-12-12-hilt-codelab%}) 에서 Hilt의 기본에 대해서 알아봤다.

간략히 정리를 해보면


- Container : 안드로이드 컴포넌트(클래스)들의 생명 주기를 고려한 hilt-component로 지정을 할 수 가 있다. 이 안에서 각 의존성들이 생성되고 필요한 곳에 주입을 시켜준다.
- @AndroidEntryPoint : hilt가 의존성 주입을 어디(어느 클래스인지를)에 해야하는지 먼저 알아야 하는데 이 @AndroidEntryPoint가 그 역할을 해준다. dagger에는 없던 어노테이션인데 이는 안드로이드 컴포넌트의 생명주기를 고려해 의존성 주입을 하기 위해 hilt에서 생긴 것이다.
  - 안드로이드 컴포넌트가 아닌 클래스에 주입할 때는 필요없다.
- @Inject : field-injection, constructor-injection 두가지가 있다.
  - 이 어노터에션이 붙은 변수(필드) 또는 생성자 파라미터에 의존성을 주입한다.
- 모듈(@Module) : constructor-injection 할 수 없는 경우에 사용(추상 클래수,인터페이스, 외부 라이브러리)
  - @Binds : 리턴 타입이 인터페이스(추상 클래스)일 경우
    - 모듈 클래스가 abstract일 때(즉 주입 메서드도 abstract)
    - 파라미터가 한개만 가능
  - @Provide : 의존성이 외부라이브러리 클래스일 경우
    - 모듈 클래스가 object(static)일 때
    - 여러 파라미터 가능
  - @Binds나 @Provides에서 Hilt가 필요한 건 리턴 타입과 파라미터들 뿐이다.(이름은 그냥 적당히 지으면 된다. 보통 fun provide~ , fun bind~)
- @InstallIn(~) : 모듈을 사용할 때 어느 안드로이드 컴포넌트를 사용할 건지 지정     



### Qualifier
:이름 그대로 `한정자`or`지정자`다<br>
이 어노테이션(콸리퐈이어)이 필요한 때는 모듈 사용시 똑같은 타입의 의존성들이 있을때다.<br>
예를 들면 Car를 상속해 각각 기름 종류에 따라 의존성이 필요할 경우를 보면
```kotlin
enum class OilType{
    GASOLINE,DIESEL,LPG
}
interface Car{
    fun getOilType():OilType
    ...
}

@InstallIn(ActivityComponent::class)
@Module
abstract class CarModule{
    
    @Binds
    fun bindGasolineCar(impl:GasolineCar):Car
    @Binds
    fun bindDieselCar(impl:DieselCar):Car
    ...
}
```
만약 어디선가 Car을 요청할때 hilt는 두 바인드 메서드에서 어느 종류의 Car을 리턴해야 하는지 모른다.<br>
이를 보완하기 위해서 @Qualifier가 등장한다.
```kotlin
@Qualifier
annotation class GasolineCar

@Qualifier
annotation class DieselCar
```
이렇게 어노테이션 클래스를 만들고 
```kotlin
@GasolineCar
@Binds
fun bindGasolineCar(impl:GasolineCar):Car

@DieselCar
@Binds
fun bindDieselCar(impl:DieselCar):Car
```
이렇게 모듈에서 지정을 하고
```kotlin
@GasolineCar
@Inject lateinit var gasolineCar:Car

@DieselCar
@Inject lateinit var dieselCar:Car
```
이렇게 사용하면 된다.
<br><br>
그리고 @Provides를 사용할때도 똑같다. 리턴 타입은 동일하나 파라미터가 다른 경우가 있겠다.
<br> ex)

```kotlin
@InstallIn(ApplicationComponent::class)
@Module
abstract class NetworkModule{
    
  @AuthInterceptorOkHttpClient
  @Provides
  fun provideAuthInterceptorOkHttpClient(
    authInterceptor: AuthInterceptor // <<
  ): OkHttpClient {
      return OkHttpClient.Builder()
               .addInterceptor(authInterceptor) // <<
               .build()
  }

  @OtherInterceptorOkHttpClient
  @Provides
  fun provideOtherInterceptorOkHttpClient(
    otherInterceptor: OtherInterceptor // <<
  ): OkHttpClient {
      return OkHttpClient.Builder()
               .addInterceptor(otherInterceptor) // <<
               .build()
  }
}
```
Predefined qualifiers(미리 정의된 한정자)
- @ActivityContext
- @ApplicationContext

이 두가지를 hilt에서 제공해준다. 정말 편하다.

___

<br>

### @EntryPoint
:@AndroidEntryPoint를 사용하지 못하는 경우 사용됨
이 어노테이션은 hilt에서 지원하지 않는 의존성 주입에 사용된다.<br>
Hilt는 안드로이드 생명주기에 특화된(보일러플레이트코드를 줄여주는) DI라고 생각해도 되겠다. 그러니 안드로이드 생명주기와 상관없는 단순 feature module같은 경우는 hilt가 아닌 dagger을 이용해야한다. <br>
(일반 클래스에서 @Inject를 그냥 사용할 수 없다.이런 경우 @EntryPoint가 필요하다.)
<br>

이 Car함수에서 local db를 사용한다고 가정하면 요론 식으로 사용할 수 있겠다.
```kotlin
class GasolineCar(private val context:Context):Car{
    override fun oilType() : OilType = OilType.GASOLINE

    //필요한 스코프를 생각해서 InstallIn을 정한다.
    @InstallIn(ApplicationComponent::class)
    @EntryPoint // 이 녀석으로 hilt가 entryPoint임을 알 수 있게 된다.
    interface DbEntryPoint{
        fun database():AppDatabase
    }

    private fun getDb(context: Context):AppDatabase{
        //EntryPointAccessors:static accessor
        //다음의 파라미터들은 hilt-component가 알 수 있는 값들이어야 한다.
        //context는 디폴트로 가지고 있고, DbEntryPoint는 위에서 지정해 주었다.
        return EntryPointAccessors.fromApplication(
            context,
            DbEntryPoint::class.java
            ).database()
    }

    //그리고 사용한다.                                    
    fun getDestination() = getDb(context).mapDao.destination 
}
```
EntryPointAccessors의 메서드
- fromApplication(context,entryPoint:Class<T!>)
- fromActivity(context,entryPoint:Class<T!>)
- fromFragment(context,entryPoint:Class<T!>)
- fromView(context,entryPoint:Class<T!>)

이렇게 나와있다. 해당 @InstallIn(인터페이스 위의)에 맞춰주면 된다.

<br>

___

<br>

Hilt를 폭 넓게 사용하려면 dagger도 알아야 한다.

dagger의 component, subComponent, Builder, Factory 등등에 대한 좋은 정보는

여기서 공부하면 좋다. 영어긴 한데 독일분인지 발음이 딱딱 떨어져 좀 듣기 수월하기도하고 코드만 대충 봐도 이해가 될 듯 싶다. >> [*유튜브 강의*](https://www.youtube.com/watch?v=ZZ_qek0hGkM&list=PLrnPJCHvNZuA2ioi4soDZKz8euUQnJW65){:target="_blank"}

<br>

## `Hilt를 익혀봤으니 이제는 써보자!!!`

<br>

____

<br>

오류 있을시 알려주시면 감사하겠습니다. 


<br>

android-doc : https://developer.android.com/training/dependency-injection <br>
android-hilt-codelab : https://codelabs.developers.google.com/codelabs/android-hilt#0



