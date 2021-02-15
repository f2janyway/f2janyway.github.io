---
title: 안드로이드 자동화 배포 경험기
categories: 
    - android
    - fastlane
    - wsl
tags:
    - android
    - fastlane
    - wsl
---

참고사항
- 윈도우에서 fastlane을 사용시 <br>
  대략적인 자동화 배포(?)의 모양을 구현하는 과정입니다.
- 윈도우 사용자 중 안드로이드로 빠르게 실행해보고 싶으신 분에게 적합할 듯 합니다.
- 상세한 내용은 없습니다.

<br>

___


<br>

개발을 하다가 플레이스토어에 수시로 올려야 하는 경우가 종종 생긴다. <br>
(특히 검토중 버그발견 시 심장이 *`쿵쾅 쿵쾅`*) <br><br>
자동화 배포를 하기 전의 과정은
1. 구글 콘솔 창을 연다.
2. 해당 앱의 프러덕션 출시를 누른다.
3. 앱 번들 파일을 드래그 앤 드롭한다.
4. 업데이트 내용을 적는다.
5. 기다린다.
6. 저장 후 출시를 누른다.
   
이런 과정을 매번 반복했다.

첫 회사 다닐 때도 자동화 배포 도구를 사용하지 않아서 매번 위와 같이 업로드를 했다.

그래서 전부터 알아는 봤지만 시도는 안해봤는데 이번에 한번 해 봤다.

찾아보던 중 fastlane이 오픈소스라 시도를 해봤다.

몇몇 블로그에 좋은 예시들이 있었는데 전부 mac os의 경우였다.

그런데 Window10 사용자라 따라하는 정도로는 불가능했다.

. <br>
.<br>
.<br>
<br>

    - 자동화 배포 삽질기          
    - 열심히 삽질을 하다보면     
    - 보석이 나올까...     
    - 피할 수 없는 길이라...      
  
### fastlane 설치
[여기 fastlane doc](https://docs.fastlane.tools/getting-started/android/setup/)에서 설명을 보고 설치한다. 
- ruby -> bundler 다운
- ./Gemfile 세팅
- sudo install fastlane


여기까지 위의 사이트에서 시키는대로 했다.

위의 과정은 전부 **wsl**에서 했다.

그리고 해당 프로젝트 루트 디렉토리(프로젝트 이름 폴더)에서 
- fastlane init

을 한다.

설치하는 중에 패키지이름과 아래 사진의 json 파일 경로를 작성해준다.

json파일은 구글 콘솔에서 얻는 건데 [이 글](https://dev-yakuza.posstree.com/ko/react-native/fastlane/)의 중간 안드로이드 파트에 상세히 나와 있다. 

(저 블로그 글을 따라 했지만 실패를 한 했다... )

그러면 아래 사진과 같이
- project
  - fastlane
    - AppFile
    - Fastfile
- Gemfile
- Gemfile.lock
  
이 생긴다. 

사진의 README.md, report.xml은 fastlane을 실행하면 생긴다.

![화면 캡처 2021-02-15 151610](https://user-images.githubusercontent.com/55625423/107911979-f58c5f80-6fa0-11eb-87cb-adf217f3406a.png){:width="400" height="400"}

여기서 [위의 블로그 글](https://dev-yakuza.posstree.com/ko/react-native/fastlane/)에 나온대로 실행을 해서 된다면 

부럽습니다...

저 코드로 했을 경우 여러모로 실패를 많이 해서 다른 온갖 방법을 시도 했다.

그래서 찾은 경우는 아래와 같다.

### fastlane supply
그냥 앱번들 파일 경로를 지정해 `fastlane supply`를 해주니

 오!! 업로드가 바로 됐다.

```sh
fastlane supply --aab app/release/app-release.aab
```
- fastlane의 여러 설정(스크린샷, 업데이트 내용 등등)이 있는데 당장 그게 중요한게 아니라 나중으로 미뤘다.

그렇다면 간단하게 `test -> 번들(.aab)파일을 만들거 위의 명령어만 연결해 실행하면 된다` 라고 생각이 들었고

 fastlane을 실행하는 wsl에서 실행하는 방법을 해보다가...

 local.properties에서 경로문제로 성공을 하지 못했다.

 wsl에서 sdk.dir=C:\\User\\~~~~\\Sdk 이 경로를 모른다고 실패를 했다.

 WSLENV로 ANDROID_SDK_ROOT로 윈도우랑 wsl이랑 연동을 시켰는데도 
 
 그보다 저 sdk.dir만 알아 차리는것 같았다.

이 local.properties는 자동 생성 파일이라 gradle의 무언가를 설정해야 하는것 같은데 방법을 모르겠다.

그래서 wsl에서 하는것을 포기하고 terminal에서 하려고 시도를 했다.

아... 그런데 또 여기선 윈도우가 fastlane을 찾지 못했다. (지금 생각해보니 여기서 WSLENV로 설정을 했으면 또 어떨지 모르겠다.)

그래서 cmd로 wsl을 실행하는 명령어가 있어서 그 방법으로 시도를 했다.

결과적으로 해당 커맨드가 자동화 배포 커맨드다.

```sh
gradlew clean test  bundleRelease && keytool -list -printcert -jarfile (경로)/app-release.aab && wsl -e fastlane supply --aab (경로)/app-release.aab
```

로직은 간단하다
1. gradlew
    - clean
    - test
    - bundleRelease
2. keytool
     - 서명 확인  
3. fastlane
    - 업로드

위의 명령어는 순차적으로 실행이 된다.

저 command에서 terminal은 && 을 알지 못했다. 

&& 는 cmd 에서 `앞의 커맨드 실행 성공 후 뒤의 것을 실행하라` 라고 한다. [스택오버플로](https://stackoverflow.com/questions/8055371/how-do-i-run-two-commands-in-one-line-in-windows-cmd)

signed check에 있어서는 여기 [안드로이드 문서](https://developer.android.com/studio/publish/app-signing)를 보고 했다. 잘 작동했다.


___


본인이 생각해도 위의 방법이 좋다고 생각이 되지 않지만

좋은 시도라고는 생각한다. 

`아 이런게 되는구나...`라는 경험

```sh
BUILD SUCCESSFUL in 3m 25s
87 actionable tasks: 85 executed, 2 up-to-date
Signer #1:

Signature:

Owner: CN=xxx
Issuer: CN=xxx
Serial number: xxx
Valid from: Fri Dec *** KST 2019 until: Tue Dec ***
Certificate fingerprints:
         MD5:  xxx
         SHA1: xxx
         SHA256: xxxxxx
Signature algorithm name: xxx
Subject Public Key Algorithm: xxx
Version: x

Extensions:

#1: ObjectId: xxx Criticality=xxx
SubjectKeyIdentifier [
KeyIdentifier [
xxxx
xxxx
]
]


[✔] 🚀
[16:03:58]: fastlane detected a Gemfile in the current directory
[16:03:58]: However, it seems like you didn't use `bundle exec`
[16:03:58]: To launch fastlane faster, please use
[16:03:58]:
[16:03:58]: $ bundle exec fastlane supply --aab app/build/outputs/bundle/release/app-release.aab
[16:03:58]:
[16:03:58]: Get started using a Gemfile for fastlane https://docs.fastlane.tools/getting-started/ios/setup/#use-a-gemfile
/var/lib/gems/2.7.0/gems/fastlane-2.174.0/fastlane_core/lib/fastlane_core/ui/***
parameters is deprecated; maybe ** should be added to the call
/var/lib/gems/2.7.0/gems/fastlane-2.174.0/fastlane_core/lib/fastlane_core/ui/***

+---------------------------------+----------------------------------------------------------+
|                                 Summary for supply 2.174.0                                 |
+---------------------------------+----------------------------------------------------------+
| aab                             | app/build/outputs/bundle/release/app-release.aab         |
| package_name                    | com.box.firecast                                         |
| release_status                  | completed                                                |
| track                           | production                                               |
| json_key                        | /***                                                     |
|                                 | ******json                                               |
| skip_upload_apk                 | false                                                    |
| skip_upload_aab                 | false                                                    |
| skip_upload_metadata            | false                                                    |
| skip_upload_changelogs          | false                                                    |
| skip_upload_images              | false                                                    |
| skip_upload_screenshots         | false                                                    |
| validate_only                   | false                                                    |
| check_superseded_tracks         | false                                                    |
| timeout                         | 300                                                      |
| deactivate_on_promote           | true                                                     |
| ack_bundle_installation_warning | false                                                    |
+---------------------------------+----------------------------------------------------------+

[16:04:02]: Preparing aab at path 'app/build/outputs/bundle/release/app-release.aab' for upload...

[!] Google Api Error: forbidden: APK specifies a version code that has already been used. - APK specifies a version code that has already been used.

```

이런 식으로 실행이 된다.

다만 버전 코드만 동일해 실패했을 뿐이다. `예시`라서요.


이 과정 중에 wsl(linux), window, gradle등 평소에 크게 관심을 갖지 않는것에 

많은 관심을 가졌더니 

`흥미로운게 참 많다.`

`알아야 할 게 많다.`

`갈 길이 멀다.`

라는걸 느낀다.


___

참고

fastlane doc : [https://docs.fastlane.tools/getting-started/android/setup/](https://docs.fastlane.tools/getting-started/android/setup/)

android doc : [https://developer.android.com/studio/publish/app-signing](https://developer.android.com/studio/publish/app-signing)

dev-yakuza 분의 블로그 : [https://dev-yakuza.posstree.com/ko/react-native/fastlane/](https://dev-yakuza.posstree.com/ko/react-native/fastlane/)


기타 자잘하게 참고한 부분이 너무 많습니다.












