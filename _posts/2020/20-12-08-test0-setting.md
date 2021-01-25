---

published: false
title: "[안드로이드 테스트(1)] 테스트 명명법 및 코드순서"
comments: true
# pagination:
#     enabled: true
categories:
    - android
    - test
tags:
    - android
date: 2020-12-08T00:15:00Z
---

### 안드로이드 테스트 간략 설명
 Local unit tests
   - module-name/src/test/java/ 에 위치
   - JVM에서 실행됨
   - 안드로이드 프레임워크에 대한 의존성 필요없음(순수 Java or Kotlin 코드 테스트)

 Instrumented tests
   -  module-name/src/androidTest/java/ 에 위치
   -  실제 하드웨어(스마트폰) 또는 에뮬레이터에서 실행
   -  통합 or UI 테스트용(안드로이드 프레임워크 의존)
  
<br>

[원문 (from codelabs)](https://developer.android.com/codelabs/advanced-android-kotlin-training-testing-basics#5)


<br>

___
<br>

### 테스트 함수 이름 짓는법(여러 방법 중 하나)

    `SUBJECT_ACTION_EXPECTED` or `ACTION_EXPECTED`
1. 테스트하고자 하는 메서드나 클래스이름 (SUBJECT)
2. 테스트하고자 하는 액션 (ACTION)
3. 기댓값 검토 (EXPECTED)

codelabs을 보면 무조건 이런 컨벤션을 따르지는 않는다.

상황에 따라 유동적으로 하면 되겠다.

```kotlin
fun add(a:Int?,b:Int?):Int {
    return a?:0 + b?:0
}
```
위 메서드를 테스트하고자 한다면 

'생각보다 쉽지 않군...'

        @Test 
    ex) add_inputNull_SumWithNullToZero(){...}

        @Test
        addNewTask_setsNewTaskEvent(){...}


해당 상황에 맞게 머리를 좀 싸매야 한다.

<br>

___

<br>

### 테스트 코드 짜는 법(여러 방법 중 하나)
1. Given: 준비 세팅 (테스트 상황이 될 수 있게 상태를 맞춤)
   - 보통 더미데이터(가짜데이터)를(들을) 준비
   - UI테스트일 경우 해당 프레그먼트나 액티비티를 실행
2. When:  테스트하고자 하는 상황
   - 보통 유닛테스트일 경우 메서드를 실행
   - 보통 UI테스트일 경우 화면 반응(전환,터치,스크롤 등)실행
3. Then:  기댓값 검토  
<br>
<br>
```kotlin
//unit-test
@Test
fun getActiveAndComletedStats_noActive_returnsZeroHundred() {
    //Given
    val tasks  = listOf<Task>(Task("MyTask","desc...",true))
    //When
    val rs = getActiveAndCompletedStats(tasks)
    //Then
    assertThat(rs.activeTasksPercent,`is`(0f))
    assertThat(rs.completedTasksPercent,`is`(100f))
}

//UI-test by espresso
//Given
@get:Rule var activityScenarioRule = activityScenarioRule<MainActivity>()

@Test
fun changeText_sameActivity() {

  // Type text and then press the button. << When
  onView(withId(R.id.editTextUserInput))
          .perform(typeText(STRING_TO_BE_TYPED), closeSoftKeyboard())
  onView(withId(R.id.changeTextBt)).perform(click())

  // Check that the text was changed. << Then
  onView(withId(R.id.textToBeChanged)).check(matches(withText(STRING_TO_BE_TYPED)))
}

```

