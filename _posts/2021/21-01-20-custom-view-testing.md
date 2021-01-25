---

# published: false
title: "[안드로이드] CustomView 테스트하기(private 접근)"
comments: true
# pagination:
#     enabled: true
categories:
    - android
    - test
tags:
    - android
date: 2021-01-20T01:15:00Z
---

<br>


[개인 프로젝트(타이머)](https://play.google.com/store/apps/details?id=com.box.firecast&hl=ko&gl=US){:target="_blank"}를 하면서 간단한 circleProgress가 필요했다. 이럴 용도로 [초라한 라이브러리](https://github.com/f2janyway/custom_view){:target="_blank"} 를 만들었지만 변경할게 많아 그냥 파일 복붙만 해서 사용했다.

![멀티타이머_progress](https://user-images.githubusercontent.com/55625423/105281830-55852580-5bf0-11eb-8994-2d537727f804.gif){: width="400" height="400"}

<br>

여기서 좀 까다로왔던 부분은 각도 싱크를 맞추는 거였다.

타이머를 시작:끝 = 0.0 : 1.0 => 12시 방향에서 시계방향으로 진행을 해야 했다.

우선 테스트 코드를 순서대로 보면

private function을 접근하기 위해 리플렉션을 사용
```kotlin
@Test
fun percentAngleSweepSyncTest(){
    val circle = mock(CircleProgress::class.java)
    
    val percentToAngle = circle::class.java
        .getDeclaredMethod("percentToAngle",Float::class.java)
    percentToAngle.isAccessible = true

    //위와 같은 방식을 그냥 Util로 만든 메서드 (바로 아래 구현 코드 있음)
    val angleToSweepAngle = getPrivateMethod(
        containerClass = circle::class.java,
        methodName = "angleToSweepAngle",
        paramsTypes = arrayOf(Float::class.java)
    )
    ...
}
```


```kotlin
fun <T> getPrivateMethod(
        containerClass: Class<out T>,
        methodName: String,
        vararg paramsTypes: Class<*>
    ): Method {
        val method = containerClass.getDeclaredMethod(methodName, *paramsTypes)
        method.isAccessible = true
        return method
    }
```
여기 float(percent)와 그로 인한 angle 그리고 필요한 sweepAngle값이 있다. <br>
원을 8등분한 각도를 테스트한다.<br>
클린 코드에는 테스트가 짧아야 좋다고 했다. 보기 쉽게. <br>
이건 많이 길다...그래도 이런 케이스는 이게 더 낳지 않나 싶다. 

```kotlin
    //...test 계속
    
    var angle2 = percentToAngle.invoke(circle,0.125f) as Float
    assertThat(45,`is`((angle2).toInt()))
    var sweepAngle = angleToSweepAngle.invoke(circle,angle2) as Float
    assertThat(45,`is`(sweepAngle.toInt()))

    angle2 = percentToAngle.invoke(circle,0.25f) as Float
    assertThat(0,`is`((angle2).toInt()))
    sweepAngle = angleToSweepAngle.invoke(circle,angle2) as Float
    assertThat(90,`is`(sweepAngle.toInt()))

    angle2 = percentToAngle.invoke(circle,0.375f) as Float
    assertThat(-45,`is`((angle2).toInt()))
    sweepAngle = angleToSweepAngle.invoke(circle,angle2) as Float
    assertThat(135,`is`(sweepAngle.toInt()))

    angle2 = percentToAngle.invoke(circle,0.5f) as Float
    assertThat(-90,`is`((angle2).toInt()))
    sweepAngle = angleToSweepAngle.invoke(circle,angle2) as Float
    assertThat(180,`is`(sweepAngle.toInt()))

    angle2 = percentToAngle.invoke(circle,0.625f) as Float
    assertThat(-135,`is`((angle2).toInt()))
    sweepAngle = angleToSweepAngle.invoke(circle,angle2) as Float
    assertThat(225,`is`(sweepAngle.toInt()))

    angle2 = percentToAngle.invoke(circle,0.75f) as Float
    assertThat(180,`is`((angle2).toInt()))
    sweepAngle = angleToSweepAngle.invoke(circle,angle2) as Float
    assertThat(270,`is`(sweepAngle.toInt()))

    angle2 = percentToAngle.invoke(circle,0.875f) as Float
    assertThat(135,`is`((angle2).toInt()))
    sweepAngle = angleToSweepAngle.invoke(circle,angle2) as Float
    assertThat(315,`is`(sweepAngle.toInt()))

    angle2 = percentToAngle.invoke(circle,1f) as Float
    assertThat(90,`is`((angle2).toInt()))
    sweepAngle = angleToSweepAngle.invoke(circle,angle2) as Float
    assertThat(0,`is`(sweepAngle.toInt()))

    
```

위의 테스트 코드를 통과해야한다!!

<hr>
이제 구현 코드로 순차적으로 들어가보자.

<br>

먼저 그림(view에서) 그려주는 코드를 보면 Path().addArc() 이다.

```kotlin
public void addArc(float left,
                   float top,
                   float right,
                   float bottom,
                   float startAngle,
                   float sweepAngle)
```
여기서 최종 테스트해야 하는 값이 `sweepAngle`이다. startAngle은 12시 방향인 270f로 두고 각각 방향 매개변수에는 left:9시, top:12시, right:3시, bottom:6시 방향의 좌표점을 매칭시켜주면 된다.

위의 테스트로도 알 수 있겠지만 자세하게 설명되있진 않으니 간략히
로직만 보면 이런 식이다.
```kotlin
val circleProgress = findViewbyId<circleProgress>(R.~)
val timePercent = passedTime.toFloat / totalTime
circleProgress.setPercent(timePercent)
// => invalidate()
```
setPercent(), percentToAngle(), sweepAngle 순서대로 보면

### setPercent()
```kotlin
fun setPercent(percentFloat: Float) {
    mPaint.apply {
        color = progressColor
        style = Paint.Style.FILL
    }
    angle = percentToAngle(percentFloat)

    initCenterPointIfNull()

    invalidate()
}
```
### percentToAngle()
```kotlin
private fun percentToAngle(percentFloat: Float): Float {
    return if (percentFloat in 0.0 .. 1.0) {
        angle = when {
            percentFloat >= 0.0 && percentFloat < 0.25 ->
                360 * (0.25 - percentFloat)
            percentFloat >= 0.25 && percentFloat < 0.5 ->
                360 * (0.249 - percentFloat)
            percentFloat >= 0.5 && percentFloat < 0.75 ->
                360 * (0.25 - percentFloat)
            percentFloat in 0.75..1.00 ->
                360 * (1.25 - percentFloat)
            else -> 360f
        }
        angle
    } else{
        360f
    }
}

```
### sweepAngle
```kotlin
private val sweepAngle: Float
    get() = when {
        90 >= angle && angle > 0 -> {
            90 - angle
        }
        angle > 90 && angle <= 180 -> {
            450 - angle
        }
        angle < 0 && angle >= -180 -> {
            90 + (-angle)
        }
        else -> {
            360
        }
    }
````

이렇게하면 위의 테스트를 통과하게 된다.

<hr>

테스트하기 전에 customview가 그리는 원리를 몇번 찾아 보긴 했는데 그걸 토대로 구현이 잘 안됐다. 특히 90도 180도 270도 0,360도 이부분들에서 좀 애를 먹었다. 그 후에 테스트 코드에 맞추어 구현이 완성됐다.





