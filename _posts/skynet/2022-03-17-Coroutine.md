---
layout: post
title: Coroutine
date: 2022-03-17T13:53:54.026Z
category: Skynet
lastmod: 2022-03-22T15:20:39.593Z
---

由于 skynet 框架的消息处理使用了 coroutine ,所以不可以将 lua 原本的 coroutine api 直接和 skynet 服务混用.否则,skynet 的阻塞 API （见 LuaAPI）将调用 coroutine.yield 而使得用户写的 coroutine.resume 有不可预期的返回值,并打乱 skynet 框架本身的处理流程.
Because the Skynet framework uses coroutine to process messages, so it's not allowed to mix the use of the original coroutine API of Lua with Skynet services. Otherwise, the blocking API of Skynet (see LuaAPI) will call coroutine.yield and it will cause an unknown return from user-defined coroutine.resume and it will break the processing flow of the Skynet framework.
skynet 프레임워크의 메시지처리는 coroutine 을 사용하기 때문에 lua 원래의 api 를 skynet 서비스에 혼용할수없다.  그렇지 않으면 skynet 의 blocking API 는 coroutine.yield 를 호출하게되고 이것은 유저정의 coroutine.resume 으로 부터 알수없는 결과를 리턴하게되고 스카이넷 프레임워크의 처리흐름을 깨지게한다.
通常,你可以使用 skynet.fork ,skynet.wait,skynet.wakeup 在 skynet 服务中创建用户级线程.
In general, you can use skynet.fork ,skynet.wait,skynet.wakeup to create a user-level thread in the Skynet service.
일반적으로 skynet.fork, skynet.wait, skynet.wakeup 을 사용하여 스카이넷서비스에서 유저레벨의 스레드를 만들수있다.
如果你有其它原因想使用 coroutine ,那么可以使用 skynet.coroutine 模块.该模块的 API 含义和 Lua 原生的 coroutine 基本一致,所以一般可以这样使用:
If you have other reasons for using coroutine, try skynet.coroutine module. It contains API that has equal functionality to the original Lua API, and call it like this:
localcoroutine =require"skynet.coroutine"

만약 다른이유로 coroutine 을 사용하고 싶으면 skynet.coroutine 모듈을 사용할수있다. 이 모듈의 API 는 lua 원래의 coroutine과 기본적으로 동일하고 그래서 일반적으로 이렇게 사용할수있다
```lua
local coroutine = require "skynet.coroutine"
```

该模块增加了一个 API:  
In this module, a new API gets added:  
이모듈에는 새로운 API 가 추가되었다.
`skynet.coroutine.thread(co)`, 它返回两个值,第一个是该 co 是由哪个 skynet thread 间接调用的.如果 co 就是一个 skynet thread ,那么这个值和 coroutine.running() 一致,且第二个返回值为 true ,否则第二个返回值为 false .这第二个返回值可以用于判断一个 co 是否是由 skynet.coroutine.create 或 skynet.coroutine.wrap 创建出来的 coroutine.
  
skynet.coroutine.thread(co) returns two values, the first one is the thread of indirect caller that invokes this co. If co is a Skynet thread, it'll return the same value as coroutine.running() and the second value will be true, otherwise it'll return false. the second return value can be used to check if co is coming from skynet.coroutine.create or skynet.coroutine.wrap.  

Skynet.coroutine.thread(co) 는 두개의 값을 리턴한다. 첫번쨰는 이 co 를 호출한 간접호출자의 스레드. 만약 co 가 skynet 스레드라면 이것은 coroutine.running() 과 같은 값을 리턴할것이다 

这里的 co 的默认值为 coroutine.running().
By default co is coroutine.running().
# 限制 Limitation 제한
如果你没有调用 skynet.coroutine.resume 启动一个 skynet coroutine 而调用了 skynet.coroutine.yield 的话,会返回错误.  
If you didn't call skynet.coroutine.resume to start a Skynet coroutine and you called skynet.coroutine.yield directly, it'll return an error.  
만약 skynet.coroutine 을 시작하기위해 skynet.coroutine.resume 을 호출하지 않고 skynet.coroutine.yield 를 직접호출했다면 에러를 반환할것이다.  

你可以在不同的 skynet 线程（由 skynet.fork 创建,或由一条新的外部消息创建出的处理流程）中 resume 同一个 skynet coroutine .但如果该 coroutine 是由 skynet 框架（通常是调用了 skynet 的阻塞 API）而不是 skynet.coroutine.yield 挂起的话,会被视为 normal 状态,resume 出错.  
You can resume the same Skynet coroutine in different Skynet processes (thread created by skynet.fork or processing flow created by an external message). If this coroutine is suspended by the Skynet framework (usually by calling blocking API of Skynet) and not by skynet.coroutine.yield, it's treated as a normal state, resume in error.  
서로다른 skynet 프로세스들에서 같은 skynet.coroutine(skynet.fork 를 통해 생성됨, 아니면 외부메시지처리를 위해 생성된 처리 프로세스) 에서 동일한 skynet.coroutine 을 resume 할수있다. 
만약 이 coroutine 이 skynet framework(보통 블럭API호출) 에의해 중지되었고 skynet.coroutine.yield 에 의해 중지된것이아니라면 이것은 정상상태로 간주되고, resume 은 에러가 발생한다.  


注:对于挂起在 skynet 框架下的 coroutine ,skynet.coroutine.status 会返回 "blocked" .  

Note: for suspended coroutine in Skynet framework, skynet.coroutine.status will return "blocked".  

주:  skynet 프레임워크하에 멈춰버린 coroutine은 skynet.coroutine.status 는 blocked 를 리턴한다


출처: <https://github.com/cloudwu/skynet/wiki/Coroutine> 
