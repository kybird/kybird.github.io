---
layout: post
title: TinyService
date: 2020-01-02 19:20:23 +0900
category: Skynet
lastmod: 2022-03-17T14:26:38.614Z
---

이는 진일보한 지침이다, skynet 경험이 많지 않은 사람에게 추천하지 않는다.
만약 간단한 서비스의 성능에 대한 특별한 요구가 있다면,  C 코드로 서비스를 작성할 수 있다. 아니면 skynet의 기존 lua 프레임워크를 우회하여 이 API 를 호출 한다.
예를 들어, 아래와 같은 코드로  examples 아래의 simpledb.lua의 서비스와 등가의 구현을 할수 있다.



```lua
local skynet = require "skynet"
local c = require "skynet.core"

local db = {}

local odispatch = skynet.dispatch_message

local unpack = c.unpack
local pack = c.pack
local send = c.send

function skynet.dispatch_message(prototype, msg, sz, session, source)
	if prototype == 10 then	-- 10 是 lua 消息，这里直接对这类消息进行解析。
		local command, key, value = unpack(msg, sz)
		if command == "GET" then
			send(source, 1, session, pack(db[key])) -- 1 是回应消息，此处相当于 skynet.ret 。
		elseif command == "SET" then
			local v = db[key]
			db[key] = value
			send(source, 1, session, pack(v))
		end
	else
		return odispatch(prototype, msg, sz, session, source)
	end
End


skynet.start(function() end)
```


이 코드조각은 원래의 skynet.dispatch_message 를 수정하였으며, 서비스가 받은 메시지를 직접 처리한다. 그리고 직접 응답한다. 왜냐하면 메시지처리 루프에서 외부로 요청(skynet.call) 하지 않으며, 그러니 원래의 복잡한 프레임워크를 사용할 필요가 없다.




출처: <https://github.com/cloudwu/skynet/wiki/TinyService> 




