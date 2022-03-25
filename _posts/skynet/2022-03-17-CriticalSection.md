---
layout: post
title: CriticalSection
date: 2022-03-17T13:53:54.026Z
category: Skynet
lastmod: 2022-03-17T15:03:37.125Z
---


skynet 서비스에서 메시지를 처리중, 만약 블록 API 를 호출하면, 정지된다. 정지한 과정중, 이 서비스는 여전히 다른메시지에 응답할 수 있다. 이때 시간 순서 문제가 발생할수 있으며, 처리하는데 매우 주의할 필요가 있다.  

다시말해, 일단 메시지처리과정에 외부 요청이 있을경우, 먼저 도착한 메시지가 이후에 도달한 메시지보다 먼저 처리될거라는 보장이 없다. 각각의 블록 API호출후, 서비스의 내부상태는 호출되기전과 동일하지 않을수 있다. (다른메시지처리중 상태가 변경될수 있음으로)  

Skynet.queue 모듈은 이러한 동시성이 가져오는 복잡성을 피하는데 도움을 준다.  

```lua
localqueue = require"skynet.queue"
```

이렇게 얻어낸 queue는 함수이다. 매번 호출할때 한개의 새로운 크리티컬 색션을 얻어낸다. 크리티컬섹션은 한 단락의 코드가 동시 실행되지 않도록 보장한다.  

```lua
local cs = queue()  -- cs 是一个执行队列 (cs is a execute queue)

local CMD = {}

function CMD.foobar()
  cs(func1)  -- push func1 into critical section
end

function CMD.foo()
  cs(func2)  -- push func2 into critical section
end
```

foobar 와 foo 두종류의 메시지를 지원하는 메시지 분발기를 구현한다고 가정해보자. 만약 skynet.queue 로 생성한 queue인 cs 를 사용한다면. 위에 처리 과정중에서, func1 과 func2 두개의 함수는, 모두 실행과정중 서로를 끼어들수 없다.  

만약 서비스가 다수의 foobar 나 foo 메시지를 받게 되면, 설령 func1 과 func2 내부에 skynet.call 과같은 종류의 블록 API 호출이 있더라도, 절대적으로 한개메시지를 처리완료후 다음메시지를 처리하게된다. 새로운 처리 과정은 cs 큐의 꼬리에 넣어지게 되고, 앞쪽의 과정이 실행완료 된후 시작되게 된다.  


주: func1 함수내부에서 다시 cs 를 호출하는것은 합법이다. 즉

```lua 
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
```

만약 이렇게 구현한다면, 매번 foobar 메시지를 받은후, 프로그램 과정은 step1, step2, step3, step4, step5 의 순서로 실행하게 되고, dead lock 이 발생하지 않는다.  

이과정에서, 만약 foobar 메시지의 처리 과정이 블럭되면, 설령 새로운 foobar 메시지가 도착하더라도, 새로운 메시지는 step1 을 바로 실행하게된다(cs 보호가 없기 때문), 그리고나서 이전의 step4가 완료된후 (step5 는 cs 보호중이아니다), 새로운 step2 를 시작하게 된다.  
