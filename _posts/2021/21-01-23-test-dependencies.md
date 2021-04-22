---

title: "[안드로이드] 테스트 dependencies 리스트"
comments: true
# pagination:
#     enabled: true
categories:
    - android
    - test
tags:
    - android
date: 2021-01-23T00:15:00Z
---


- [AndroidX test setUp (Instrument test)](https://developer.android.com/training/testing/set-up-project#gradle-dependencies) 

- [viewModel, repository, dataSource, fragment testing dependencies codelab](https://developer.android.com/codelabs/advanced-android-kotlin-training-testing-test-doubles?hl=ko#7)

### test(local)
- testImplementation "androidx.test:core-ktx:$core_ktx_version"

- testImplementation "androidx.test.ext:junit-ktx:$junit_ktx_version"
    - AndroidJUnit4
- testImplementation "androidx.arch.core:core-testing:$arch_core_testing_version"
- testImplementation "androidx.room:room-testing:$room_version"
- testImplementation "org.hamcrest:hamcrest-all:$hamcrest_version"
- testImplementation "org.robolectric:robolectric:$robolectric_version"
- testImplementation "org.mockito:mockito-core:$mockito_core_version"
- testImplementation "org.mockito:mockito-inline:$mockito_inline_version"
- testImplementation "org.jetbrains.kotlinx:kotlinx-coroutines-test:$coroutine_version"
- testImplementation "com.google.truth:truth:$truth_version"
  
### AndroidTest(UI)
- androidTestImplementation "androidx.test.ext:junit:$core_ktx_version"
- androidTestImplementation "androidx.test.espresso:espresso-core:$espresso_version"
- androidTestImplementation "androidx.test.espresso:espresso-contrib:$espresso_version"
- androidTestImplementation "androidx.test.ext:junit:$junit_ktx_version"
- androidTestImplementation "androidx.arch.core:core-testing:$arch_core_testing_version"
- androidTestImplementation "org.jetbrains.kotlinx:kotlinx-coroutines-test:$coroutine_version"


// Core library
- androidTestImplementation "androidx.test:core:$androidx_text_core_version"

// AndroidJUnitRunner and JUnit Rules
- androidTestImplementation "androidx.test:runner:1.1.0"
- androidTestImplementation "androidx.test:rules:1.1.0"

// Assertions
- androidTestImplementation "androidx.test.ext:junit:$ext_truth_version"
- androidTestImplementation "androidx.test.ext:truth:$ext_truth_version"
- androidTestImplementation "com.google.truth:truth:$truth_version"

// Espresso dependencies
- androidTestImplementation "androidx.test.espresso:espresso-core:$espresso_version"
- androidTestImplementation "androidx.test.espresso:espresso-contrib:$espresso_version"
- androidTestImplementation "androidx.test.espresso:espresso-intents:$espresso_version"
- androidTestImplementation "androidx.test.espresso:espresso-accessibility:$espresso_version"
- androidTestImplementation "androidx.test.espresso:espresso-web:$espresso_version"
- androidTestImplementation "androidx.test.espresso.idling:idling-concurrent:$espresso_version"

// The following Espresso dependency can be either "implementation"
// or "androidTestImplementation", depending on whether you want the
// dependency to appear on your APK"s compile classpath or the test APK
// classpath.
- androidTestImplementation "androidx.test.espresso:espresso-idling-resource:$espresso_version"

<br>

___

<br>
<br>


