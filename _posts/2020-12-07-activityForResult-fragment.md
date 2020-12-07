---
layout: posts
title: "fragment의 startActivityForResult() 고찰"
comments: true
# pagination:
#     enabled: true
categories:
    - android
tags:
    - android
date: 2020-12-07T09:15:00Z
---

activity 에서 startActivityForResult를 할 경우는 직관적으로 해당 activity의 onActivityResult()를 바로 진입함을 알 수있다.

**그런데** activity 안의 fragment 에서 startActivityForResult를 실행할 경우 onActivityResult()를 해당 activity 에서 처리를 해줘야 할까? 아니면 fragment에서 처리를 해줘야 할까?

    androidx.fragment.app.FragmentActivity 의 startActivityForResult << not this case
    androidx.fragment.app.Fragment의 startActivityForResult << this case


정석적으로는 fragment에서 받아주는게 맞다. 그러나 activity에서도 처리를 할 수 있긴하다.

### test requestCode는 111

logcat을 보면   

                                                                    requestCode/resultCode/data.getIntExtra()
    D/TestActivity:     >>  onActivityResult:  ####  up super       65647 -1 100  
    D/BlankFragment0:   >>  onActivityResult:  ####  up super       111 -1 100
    D/BlankFragment0:   >>  onActivityResult:  ####  bottom super   111 -1 100 
    D/TestActivity:     >>  onActivityResult:  ####  bottom super   65647 -1 100 

즉 fragment 에서 startActivityForResult()을 호출할 경우 이 순서로 실행되고 requsetCode는 해당 fragment에서만 같다. 그러나 resultCode나 data는 동일하게 이동하는것을 볼 수 있다. 

좋은 방법은 아니지만 특별한 경우(어쩔수 없는 경우)에는 activity의 onActivityResult에서 처리를 해주어도 될것 같다.

```kotlin
(in activity)
override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?){
        //1 (순서)
        super.onActivityResult(requestCode, resultCode, data)--->@
        //4
}

(in fragment)
@--->override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?){
        //2
        super.onActivityResult(requestCode, resultCode, data)
        //3
}
//activity의 super.onActivityResult(requestCode, resultCode, data) 을 호출하면 
//아래 코드를 지나고 여기서 (*) onActivityResult를 호출함을 볼 수 있다.
Fragment targetFragment = mFragments.findFragmentByWho(who);
if (targetFragment == null) {
    Log.w(TAG, "Activity result no fragment exists for who: " + who);
} else {
    //(*)
    targetFragment.onActivityResult(requestCode & 0xffff, resultCode, data);
}
return;
```


여기서 activity의 requestCode는 *65647*로 나오는데 이는 (Fragment)startActivityForResult()가 맨처음 호출되고 바로 host인 그 내부에서 바로 
```kotlin
mHost.onStartActivityFromFragment(this /*fragment*/, intent, requestCode, options);
```

를 호출한다. 여기서 또

```kotlin
ActivityCompat.startActivityForResult(
                    this, 
                    intent, 
                    ((requestIndex + 1) << 16) + (requestCode & 0xffff), 
                    options);
```


을 호출하는데 여기서 위의 (activity의 onActivityResult)1번이 호출되고 위의 shift연산이 **65647**을 생성하게 된다. 그리고 (FragmentActivity) 내부의 onActivityResult()에서 이 **65647**를 다시 **(requestCode & 0xffff)** 으로 원래 requestCode로 변경해 놓고 fragment의 onActivityResult에서 제대로 된 requestCode를 넘겨준다.



shift 연산이 친숙하진 않는데 위의 계산대로 해보면 이렇게 나오는걸 보면 대충 이해할 수 있을듯 하다. 
```kotlin
println(Integer.toBinaryString(65647))      10000000001101111
println(Integer.toBinaryString(111))                  1101111
println(Integer.toBinaryString(0xffff))      1111111111111111
```



전부를 디버깅 ((F7)step into로 )하는것은 좀 아니지만 끊어가면서 한번은 해볼 가치가 있을 듯 하다.



<br><br>
디버깅하면서 느낀점

 - cpu의 속도를 새삼 느끼고
 - android sdk 개발자들의 레벨을 어느정도 느끼고
 - 가야할 길이 멀다는것을 느낌