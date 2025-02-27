---
layout: post
title: TimeoutCall
date: 2020-01-02 19:20:23 +0900
category: Skynet
lastmod: 2022-03-17T14:26:22.068Z
---

skynet 은 call 의 행위에 대해, 오로지 상대방이 버그가 없고 반환값이 없다면, framework 는 무한블럭이 되지 않는것을 보장한다(리턴or에러발생). 그래서 타임아웃 메커니즘을 제공하지 않는다. 이는 코드를 작성할때 심리적 부담을 줄일수 있다.  특히 동일한 프로세스내에서, 호출 대기시간초과 검사는 bug 를 덮어버린다 (호출된 서비스에 skynet.ret의 누락 아니면 설계오류, 어떤서비스의 처리 능력의 과부하). 보통의 프로그램에서, 라이브러리 함수 호출에 대기시간초과를 설정하지 않는것이 이치이다. 하지만 어떤특수한 상황에서, 만약 꼭 skynet.call 에 대기시간초과를 설정하고자 한다면, 스스로 간단한 wrapper 를 만들수 있다.

```lua
local function timeout_call(ti, ...)
	local co = coroutine.running()
	local ret

	skynet.fork(function(...)
		ret = table.pack(pcall(skynet.call, ...))
		if co then
			skynet.wakeup(co)
		end
	end, ...)

	skynet.sleep(ti)
	co = nil	-- prevent wakeup after call
	if ret then
		if ret[1] then
			return table.unpack(ret, 1, ret.n)
		else
			error(ret[2])
		end
	else
		-- timeout
		return false
	end
end
```

다른방법:

https://github.com/cloudwu/skynet/blob/master/test/testselect.lua#L72-L79

출처: <https://github.com/cloudwu/skynet/wiki/TimeOutCall> 


