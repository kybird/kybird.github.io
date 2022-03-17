---
layout: post
title: CriticalSection
date: 2022-03-17T13:53:54.026Z
category: Skynet
lastmod: 2022-03-17T15:03:37.125Z
---
同一个 skynet 服务中的一条消息处理中，如果调用了一个阻塞 API ，那么它会被挂起。挂起过程中，这个服务可以响应其它消息。这很可能造成时序问题，要非常小心处理。
When processing a message in the same Skynet service, if the user calls a blocking API, it will be suspended. During the time of suspension, this service can still respond to other services, but it's very likely to have time sequencing problems and it should be handled very carefully.
스카이넷서비스가 메시지를 처리할때 유저가 블럭킹API 를 호출할경우 해당시간동안 프로세스는 정지 됩니다. 하지만 이서비스는 여전히 다른서비스에 응답할수있습니다. 이때 시간 순서 문제가 발생할 가능성이 매우높기때문에 이를 처리하는데 매우 주의해야합니다.

换句话说，一旦你的消息处理过程有外部请求，那么先到的消息未必比后到的消息先处理完。且每个阻塞调用之后，服务的内部状态都未必和调用前的一致（因为别的消息处理过程可能改变状态）。
In other words, once your message processing contains an external request, it's not guaranteed the former messages will be processed earlier than the latter. And after each blocking call, the internal state of service before the call might not be the same as after (because it might be changed during the message processing).
다시말해서, 메시지가 외부 요청을 포함할경우 이전매시지가 이후메시지보다 먼저 처리될거라는 보장이 없다. 그리고 각각의 블럭킹콜 이후 호출되기전의 서비스의 내부상태는 이전과 동일하지 않을수 있다 ( 왜냐하면 이는 메시지 처리과정에 서 변경될수있기 때문에)
skynet.queue 模块可以帮助你回避这些伪并发引起的复杂性。
skynet.queue module helps you avoid the complexity caused by fake concurrency.
Skynet.queue 모듈은 이러한 동시성이 가져오는 복잡성을 피하는데 도움을 준다.

localqueue =require"skynet.queue"
这样获得的 queue 是一个函数，每次调用它都可以得到一个新的临界区。临界区可以保护一段代码不被同时运行。
The queue returned is a function, each time you'll get a new critical section and it can be used to protect code block from executing concurrently.
큐의 반환값은 함수이다. 매번 당신은 새로운 크리티컬섹션을 얻을것이고 동시에 실행되는것을 보호해준다.
local cs = queue()  -- cs 是一个执行队列 (cs is a execute queue)

local CMD = {}

function CMD.foobar()
  cs(func1)  -- push func1 into critical section
end

function CMD.foo()
  cs(func2)  -- push func2 into critical section
end


比如你实现了这样一个消息分发器，支持 foobar 和 foo 两类消息。如果你使用 cs 这个 skynet.queue 创建出来的队列。那么在上面的处理流程中， func1 和 func2 这两个函数，都不会在执行过程中相互被打断。
Assume you implemented a message dispatcher and it supports two message types: foobar and foo. If you use queue cs created by skynet.queue, in the process flow above, func1 and func2 will not be terminated by each other.

두개의 메시지타입 footbar 와 foor 를 지원하는 dispatcher 를 구현한다고 가정한다. 만약 당신이 skynet.queue를 이용하여 만든 queuecs 를 사용한다면 위의 처리흐름에서 func1 과 func2 는 서로 중단되지 않을것이다.
如果你的服务收到多条 foobar 或 foo 消息，一定是处理完一条后，才处理下一条，即使 func1 或 func2 中有 skynet.call 这类的阻塞调用。一旦它们被挂起，新的消息到来后，新的处理流程会被排到 cs 队列尾，等待前面的流程执行完毕才会开始。
If your service receives multiple messages of foobar or foo, it will be processed one after another in order, even if there is a blocking call like skynet.call in func1 or func2. Once it's suspended and new messages arrive, a new process flow will be put at the end of queue cs, waiting for the previous flow to finish its work.
만약 서비스가 foorbar나 foo 의 메시지를 여러 개를 받게 된다면. 한개씩 한개씩 순서에 따라 처리할것이다. 저기 func1 이나 func2 에 skynet.call 과같은 블럭킹 호출이 있다해도, 일단 정지하고 새로운 메시지가 도착하고 새로운 프로세스 흐름은 queue cs 의 끝에 넣어지게 된다. 이전흐름이 끝날때까지 기달려야한다

注：在 func1 函数内部再调用 cs 是合法的。即：
주: func1 함수내부에서 다시 cs 를 호출하는것은 합법이다. 아래처럼?
Note: it's legal to call cs in function func1, that is:
local function func2()
  -- step 3
end

local function func1()
  -- step 2
  cs(func2)
  -- step 4
end

function CMD.foobar()
  -- step 1
  cs(func1)  -- push func1 into critical section
  -- step 5
end


如果你这样写，每次收到 foobar 消息后，程序流程会按 step 1, step 2, step 3, step 4, step 5 这样执行，而不会死锁。
If you implement it like this, every time foobar receives messages, process flow executes it in order of step1, step 2, step 3, step 4, step 5 without deadlock.
만약이런식으로 구현한다면, 매번 footbar 는 메시지들을 받게 된다. 처리흐름은 step1,step2, step3,step4, step5로 데드락없이 진행된다.
在这个过程中，如果 foobar 消息的处理流程被挂起，即使新的 foobar 消息到来，那么，新的消息会立刻执行 step 1 （因为没有被 cs 保护），然后等前一次的 step 4 结束后（step 5 不在 cs 保护中），开始新的 step 2 。
In this process, if any message process flow gets suspended, even if a new foobar message arrives, the new message will execute step1 immediately (because it's not protected by cs), then wait for the step4 from previous flow (step5 is not protected by cs), then start a new step2

이 처리에서, 만약 어떤메시지 처리 흐름이 멈추었다면 새로운 foobar 메시지가 도착하더라도 새로운 메시지는  step1 을 즉시 실행한다 ( 왜냐하면 이것은 cs 에의해 보호되지 않기때문) 그리고 step4 를 이전의 흐름에서 기달린다. Step5 또한 cs 에 보호되지않는다. 그래고 새로운 step2 를 시작한다.
