---
layout: post
title: Coroutine
date: 2022-03-17T13:53:54.026Z
category: Skynet
lastmod: 2022-03-22T15:20:39.593Z
---

skynet 프레임워크의 메시지처리는 coroutine 을 사용하기 때문에, 오리지날 lua의 coroutine api 를 skynet 서비스에 혼용할수없다, 그렇지 않으면 skynet 의 blocking API (보라 [LuaAPI](2022-03-17-LuaAPI.markdown)) 는 coroutine.yield 를 호출하게되고 이것은 유저가 정의한 coroutine.resume 은 예상할 수 없는 결과를 리턴하게되고, 스카이넷 프레임워크의 처리흐름을 깨지게한다.

통상 skynet.fork, skynet.wait, skynet.wakeup 을 사용하여 스카이넷 서비스에서 유저레벨의 스레드를 만들수있다.  

만약 다른이유로 coroutine 을 사용하고 싶으면 skynet.coroutine 모듈을 사용할수있다. 이 모듈의 API 는 lua 원래의 coroutine과 기본적으로 동일하고, 일반적으로 이렇게 사용할수있다.  

```lua
local coroutine = require "skynet.coroutine"
```

이모듈에는 새로운 API 가 추가되었다:

> `skynet.coroutine.thread(co)`, 它返回两个值,第一个是该 co 是由哪个 skynet thread 间接调用的.如果 co 就是一个 skynet thread ,那么这个值和 coroutine.running() 一致,且第二个返回值为 true ,否则第二个返回值为 false .这第二个返回值可以用于判断一个 co 是否是由 skynet.coroutine.create 或 skynet.coroutine.wrap 创建出来的 coroutine.
  
> skynet.coroutine.thread(co) returns two values, the first one is the thread of indirect caller that invokes this co. If co is a Skynet thread, it'll return the same value as coroutine.running() and the second value will be true, otherwise it'll return false. the second return value can be used to check if co is coming from skynet.coroutine.create or skynet.coroutine.wrap.  

> `skynet.coroutine.thread(co)`, 두개의 값을 리턴한다. 첫번째는 이 co가 어떤 skynet thread 의 간접호출되었는지. 만약 co 가 skynet thread라면, 이것의 값은  `coroutine.running()` 과 동일하다. 그리고 두번째 반환값은 true 이다, 아니면 두번째 값이 false 이다. 이 두번째 반환값은 co 가 `skynet.coroutine.create` 나 `skynet.coroutine.wrap` 에의해 생성된 coroutine 인지 판단하는데 사용할 수 있다.

> 역: 파라메터로 넣은 co 가 일반 코루틴인지 skynet 코루틴인지 판단할수 있다는 말인거 같다. 증거를 만들어보자.

co 의 기본값은 `coroutine.running()`.
> 역: 파라메터의 기본값이 coroutine.running()이란말인가.. 루아에서 무슨뜻인지 몰라서 이해를 할수 없다. 루아에 기본파라메터 설정이있던가??

# 제한
만약 skynet.coroutine 을 시작하기위해 skynet.coroutine.resume 을 호출하지 않고 skynet.coroutine.yield 를 직접호출했다면 에러를 반환한다.  
 
> 你可以在不同的 skynet 线程（由 skynet.fork 创建,或由一条新的外部消息创建出的处理流程）中 resume 同一个 skynet coroutine .但如果该 coroutine 是由 skynet 框架（通常是调用了 skynet 的阻塞 API）而不是 skynet.coroutine.yield 挂起的话,会被视为 normal 状态,resume 出错.  
> 
> You can resume the same Skynet coroutine in different Skynet processes (thread created by skynet.fork or processing flow created by an external message). If this coroutine is suspended by the Skynet framework (usually by calling blocking API of Skynet) and not by skynet.coroutine.yield, it's treated as a normal state, resume in error.  
> 
> 서로 다른 skynet 스레드에서 (skynet.fork 로 생성되거나 아니면 외부메시지에 의해 생성된 외부프로세스)에서 동일한 skynet coroutine을 resume 할수 있다. 하지만 만약 이 coroutine이 skynet.coroutine.yield 로 정지된 것이 아니고 skynet 프레임워크(통상 skynet의 블록 API 호출)로 정지된것이라면, 정상 상태로 취급하며, resume 은 오류가 발생한다. 
> 
> 역: ??? 번역은 제대로된거 같은데 너무어렵다 증거를 만들어보자 ???
> >skynet framework 로 정지한 coroutine 과 skynet.coroutine.yield 로 정지한것에 동작에 차이가 있는지 확인
> 
 

skynet.coroutine.yield 로 정지된 것이 아니라면
주: skynet 정지한 skynet framework 의 coroutine은, skynet.coroutine.status 의 반환값은  blocked 이다.


출처: <https://github.com/cloudwu/skynet/wiki/Coroutine> 
