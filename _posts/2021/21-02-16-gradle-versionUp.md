---
title: gradle 버전코드, 버전이름 자동 증가 코드
categories: 
    - android
    - gradle
tags:
    - android
    - gradle
    - groovy
---
안드로이드 업데이트를 할 때마다 versionCode(or versionName)을 변경해 준다.

전에는 별생각없이 수동으로 올려주었는데 자동화 배포를 접하고서 이것도 자동화시키고 싶어졌다.

이전 글의 fastlane 설정파일(Fastfile)에서 그렇게 하는 코드를 보고 그것을 사용하려고 여러번 시도를 해봤든데 실패했다.

그래서 gradle에 custom task를 만들어 적용해봤다.

단순히 파일 IO 하는 코드인데 단지 gradle task 용 groovy인 것밖에 없다. 처음해보는 거였지만 큰 어려움이 없었다.

<br>


build.gradle(.app)
```gradle
versionCode 32
versionName "0.0.3"
```

build.gradle에서 각자 사용하는 형태에 맞게 versioning등을 변경해주면 됩니다.


{% gist 4f8f8549aace5217d1ed44ae7b31596e %}

사용법은 

```
// versionCode만 1 증가
gradlew versionUp

//  versionCode, "0.0.0" -> "1.0.0"
gradlew versionUp --versioning="major"

// versionCode, "0.0.0" -> "0.1.0"
gradlew versionUp --versioning="minor"

// versionCode, "0.0.0" -> "0.0.1"
gradlew versionUp --versioning="patch"

```


major, patch등을 복합적으로 증가시키는 법은 어렵지 않으니 응용해보세요.

<br>









    

