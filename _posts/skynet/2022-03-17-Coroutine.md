---
layout: post
title: Coroutine
date: 2022-03-17T13:53:54.026Z
category: Skynet
lastmod: 2022-05-23T06:38:16.790Z
---

skynet 프레임워크의 메시지처리는 coroutine 을 사용하기 때문에, 오리지날 lua의 coroutine api 를 skynet 서비스에 혼용할수없다, 그렇지 않으면 skynet 의 blocking API (보라 [LuaAPI](2022-03-17-LuaAPI.markdown)) 는 coroutine.yield 를 호출하게되고 이것은 유저가 정의한 coroutine.resume 은 예상할 수 없는 결과를 리턴하게되고, 스카이넷 프레임워크의 처리흐름을 깨지게한다.

통상 skynet.fork, skynet.wait, skynet.wakeup 을 사용하여 스카이넷 서비스에서 유저레벨의 스레드를 만들수있다.  

만약 다른이유로 coroutine 을 사용하고 싶으면 skynet.coroutine 모듈을 사용할수있다. 이 모듈의 API 는 lua 원래의 coroutine과 기본적으로 동일하고, 일반적으로 이렇게 사용할수있다.  

```lua
local coroutine = require "skynet.coroutine"
```

이모듈에는 새로운 API 가 추가되었다:

`skynet.coroutine.thread(co)`, 두개의 값을 리턴한다. 첫번째 내용은 co가 어떤 skynet thread 에 의해 간접호출되었는지 알려준다. 만약 co 가 skynet thread라면, 그 값은  `coroutine.running()` 과 동일하다. 그리고 두번째 반환값은 true거나 false 이다. 이 두번째 반환값은 co 가 `skynet.coroutine.create` , `skynet.coroutine.wrap` 둘중 어느것에 의해 생성된 coroutine 인지 판단하는데 사용할 수 있다.

# 제한
만약 skynet.coroutine 을 시작하기위해 skynet.coroutine.resume 을 호출하지 않고 skynet.coroutine.yield 를 직접호출했다면 에러를 반환한다.  
 
 서로 다른 skynet 스레드(skynet.fork 로 생성되거나, 외부메시지에 의해 처리과정중 생성된)에서 동일한 skynet coroutine을 resume 할수 있다. 하지만 coroutine이 skynet.coroutine.yield 가아닌 skynet 프레임워크 (skynet의 블럭 API호출)에 의해 블럭된것이라면,  normal 상태로 간주되고, result 은 오류가 발생한다. 

skynet.coroutine.yield 로 정지된 것이 아니라면
주: skynet 정지한 skynet framework 의 coroutine은, skynet.coroutine.status 의 반환값은  blocked 이다.


출처: <https://github.com/cloudwu/skynet/wiki/Coroutine> 
