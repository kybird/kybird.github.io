---
layout: post
title: Bootstrap
date: 2022-03-17T13:53:54.026Z
category: skynet
lastmod: 2022-03-24T16:14:47.965Z
---

skynet 은 하나나 다수의 프로세스로 이루어진다, 각 프로세스는 skynet 노드라 불린다. 이장에서는 skynet 노드의 시작 과정에 대해 이야기한다.  
<BR>
skynet 노드는 skynet의 메인 프로세스에 의해 기동 된다. 기동 시 명령행에 [Config](2022-03-17-Config.md)  설정파일을 파라메터로 설정 해야 한다. skynet 은 이 config 파일을 읽어서 기동에 필요한 파라메터들을 가져온다.  
<BR>
첫번째 시작되는 서비스는 logger이며, 서비스의 로그를 관리하며 이후의 로그를 출력한다. logger 는 간단한  C서비스이고, skynet_error 라는 C API 로 logger에 문자열을 전송한다. config 파일에서 logger 항목에 log 파일명을 지정할 수 있으며, 기본값은 nil이며 이는 stdout 을 의미한다.  
<BR>
bootstrap 설정항목은 skynet 의 두번째 시작되는 서비스에 대한 것이다. 전체시스템은 이 서비스에 의해 시작된다. bootstrap 의 기본값은 'snlua bootstrap' 이다.  이는 skynet이 snlua라는 서비스를 실행하며 bootstrap 을 파라메터로 준다는 뜻이다.  snlua 는 lua의 샌드박스 서비스이며, bootstrap 은 luaservice 설정값에 의거하여  해당이름의 스크립트를칮아낼것이다.  기본설정에 의하면, 이 스크립트의 위치는 service/bootstrap.lua 이다.  
<BR>
필요 없다면,  bootstrap 설정 값을 변경할 필요가 없으며, 기본값인 bootstrap을 동작하게 한다, 현재 bootstrap 의 스크립트는 아래와같다.  
<BR>
```lua
local skynet = require "skynet"
local harbor = require "skynet.harbor"

skynet.start(function()
	local standalone = skynet.getenv "standalone"

	local launcher = assert(skynet.launch("snlua","launcher"))
	skynet.name(".launcher", launcher)

	local harbor_id = tonumber(skynet.getenv "harbor")
	if harbor_id == 0 then
		assert(standalone ==  nil)
		standalone = true
		skynet.setenv("standalone", "true")

		local ok, slave = pcall(skynet.newservice, "cdummy")
		if not ok then
			skynet.abort()
		end
		skynet.name(".slave", slave)

	else
		if standalone then
			if not pcall(skynet.newservice,"cmaster") then
				skynet.abort()
			end
		end

		local ok, slave = pcall(skynet.newservice, "cslave")
		if not ok then
			skynet.abort()
		end
		skynet.name(".slave", slave)
	end

	if standalone then
		local datacenter = skynet.newservice "datacenterd"
		skynet.name("DATACENTER", datacenter)
	end
	skynet.newservice "service_mgr"
	pcall(skynet.newservice,skynet.getenv "start" or "main")
	skynet.exit()
end)
```

이 스크립트는 standalone 설정에 의거하여 마스터 노드를 기동할 것인지 슬레이브 노드를 시작할것인지 판단한다. 만약 마스터 노드라면 싱글 노드 스카이넷을 실행여부를 harbor 설정값이 0인지 체크하여 결정한다.  <BR>

싱글 노드일 경우, 노드간 통신에 harbor 메커니즘이 필요하지 않다. 하지만 호환성을 위해 (전역이름을 등록할수도 있기때문에), cdummy 라 불리는 서비스를 기동할 필요가 있으며, 이는 외부로broadcasting 하는 전역이름의 변경을 블록 한다.  ?? 무슨뜻일까 cdummy 가 싱글노드일경우 전역 브로드캐스팅을 막는다고 어떻게 왜???  <BR>

만약 다중 노드 모드라면, 마스터 노드에서, cmaster 서비스를 노드들의 관리자로 기동해야 한다. 더하여 모든 노드 (마스터를 포함하여)에 cslave 서비스를 기동한다.  이서비스는 노드들 사이의 전역이름을 동기화하고 노드들 사이의 메시지를 전달하는데 사용된다.  <BR>

계속해서 마스터 노드에는, 또한[DataCenter](2022-03-17-DataCenter.md) 서비스를 기동해야 한다  <BR>

계속해서 [UniqueService](2022-03-17-UniqueService.md) 를 관리 하기위한 server_mgr을 기동한다.  <BR>

최후로, config 에 읽어 들인 start 항목, 유저가 정의한 서비스 기동의 시작점 스크립트를 운행한다, 성공 후, 스스로 종료한다.  <BR>

이 start 설정항목은, 유저가 정의한 기동 스크립트이며, 기본값은 main 이다. 만약 당신이 skynet을 그저 테스트 하려고 한다면, 여러 개의 시작 스크립트를 가지고 있을 것이다,  그렇다면 여러 개의 config 함수를 만들어, 각설정파일에 서로 다른 start 항목을 정의하는 것을 추천한다, examples 디렉토리에는 아주 많은 이러한 예제가 있다.  <BR>

