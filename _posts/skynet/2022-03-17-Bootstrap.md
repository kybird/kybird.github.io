---
layout: post
title: Bootstrap
date: 2022-03-17T13:53:54.026Z
category: Skynet
lastmod: 2022-03-22T14:24:51.213Z
---



skynet 由一个或多个进程构成，每个进程被称为一个 skynet 节点。本文描述了 skynet 节点的启动流程。

Skynet is consist of one or multiple processes, each process is called a Skynet node. This section is about the starting flow of the Skynet nodes.
Skynet 은 한 개나 다수의 프로세스로 이루어진다, 각 프로세스는 skynet 노드라 불린다. 이장에서는 skynet 노드들의 시작과정에대해 이야기한다.

skynet 节点通过运行 skynet 主程序启动，必须在启动命令行传入一个 [Config](2022-03-17-Config.md) 文件名作为启动参数。skynet 会读取这个 config 文件获得启动需要的参数。

Skynet node is invoked by Skynet's main process, it has to be started by a [Config](2022-03-17-Config.md) file as running parameters. Skynet read this config file to get the running parameters.

Skynet 노드는 Skynet의 메인프로세스에의해 호출된다. 노드는 [Config](2022-03-17-Config.md) 파일들로 실행파라메터들이 설정되어야한다 skynet 은 이 config 파일을 읽어서 실행파라메터들을 가져온다.

第一个启动的服务是 logger ，它负责记录之后的服务中的 log 输出。logger 是一个简单的 C 服务，`skynet_error` 这个 C API 会把字符串发送给它。在 config 文件中，logger 配置项可以配置 log 输出的文件名，默认是 nil ，表示输出到标准输出。

The first started service is logger, it's handling the output of services log. logger is a simple C service and a C API called `skynet_error` sends strings to it. In the config file, logger config entry can be set to the log file name, and it defaults to nil, which means standard output.

첫번째시작된서비스는 logger, 서비스의 로그를 관리한다. Logger 는 간단한 C서비스이고 `skynet_error` 라는 C API 로 로거에 문자열을 전송한다. Config 파일에서 logger config 엔트리는 log 파일이름으로 설정될수있고 기본은 nil 이고 이것은 표준 output 이다.


bootstrap 这个配置项关系着 skynet 运行的第二个服务。通常通过这个服务把整个系统启动起来。默认的 bootstrap 配置项为 "snlua bootstrap" ，这意味着，skynet 会启动 snlua 这个服务，并将 bootstrap 作为参数传给它。snlua 是 lua 沙盒服务，bootstrap 会根据配置的 luaservice 匹配到最终的 lua 脚本。如果按默认配置，这个脚本应该是 service/bootstrap.lua 。

bootstrap config is about the second started service of Skynet. The whole system is started by this service. The default config entry of bootstrap is "snlua bootstrap", which means Skynet will run a service called snlua and pass bootstrap as the parameter. snlua is the sandbox of Lua, bootstrap will match the final Lua script based on the luaservice config. By default, the script config entry should be service/bootstrap.lua.

Bootstrap 설정은 skynet 의 두번째 시작되는 서비스에 대한것이다. 전체시스템은 이서비스에의해 시작된다. Bootstrap 의 기본설정엔트리는 'snlua bootstrap' 이다. 이말은 Skynet 이 snlua 라는 서비스를 실행할것이고bootstrap 을 파라메터로 넘길것이라는것이다. Snlua 는 lua 의 샌드박스이다. Bootstrap 은 luaservice config 에 기반하여 마지막 lua 스크립트와 매치할것이다. 기본으로, 스크립트 설정 엔트리는 service/bootstrap.lua 이다.

如无必要，你不需要更改 bootstrap 配置项，让默认的 bootstrap 脚本工作。目前的 bootstrap 脚本如下：

You don't need to change the bootstrap config entry to make the default bootstrap script work unless it's necessary. The bootstrap script is as follows:

만약당신이 bootstrap 설정 엔트리를 바꿀필요가 없다면 현재 bootstrap 의 스크립트는 아래와같다.
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
这段脚本通常会根据 standalone 配置项判断你启动的是一个 master 节点还是 slave 节点。如果是 master 节点还会进一步的通过 harbor 是否配置为 0 来判断你是否启动的是一个单节点 skynet 网络。

This script will check if you want to start a master node or slave node based on standalone config entry. If it's a master node, it'll make a decision on whether to start a single node Skynet network or not by checking if harbor config entry is 0.

이스크립트는 standalone 설정에 기반하여 마스터노드를 시작할것인지 슬레이브노드를 시작할것인지 체크한다 만약 마스터노드라면 싱글노드 스카이넷을 실행여부를 harbor 설정의 엔트리가 0인지 체크하여 결정한다.

单节点模式下，是不需要通过内置的 harbor 机制做节点间通讯的。但为了兼容（因为你还是有可能注册全局名字），需要启动一个叫做 cdummy 的服务，它负责拦截对外广播的全局名字变更。

When it's single-node mode, it doesn't need harbor strategy for node communication. But to be compatible (because you may register global names), you may need to run a service called cdummy, which is used for blocking the broadcast of the changes of global names.

싱글노드일경우, 노드통신에 항구전략이 필요하지 않다. 하지만 호환성을위해 (전역이름을 설정할것이기에) 글로벌이름이 변경의 broadcast 를 블럭하는 cdummy 라 불리는 서비스를 실행할필요가있을것이다

如果是多节点模式，对于 master 节点，需要启动 cmaster 服务作节点调度用。此外，每个节点（包括 master 节点自己）都需要启动 cslave 服务，用于节点间的消息转发，以及同步全局名字。

In multiple-node mode, the master node starts cmaster service as coordinator of nodes. In addition, each node including master itself needs to run cslave service, which is used for message forwarding between nodes and synchronization of global names.

만약 다중노드모드라면, 마스터노드는 cmaster 라는 서비스를 노드들의 관리자로 시작한다. 더하여 마스터를 포함하여 각노드들은 cslave 서비스를 실행해야한다.  이서비스는 노드들간의 전역이름을 동기화하고 노드들간의 메시지를 전달하는데 사용된다.

接下来在 master 节点上，还需要启动 [DataCenter](2022-03-17-DataCenter.md) 服务。

Next, the master node also needs to run DataCenter service.

다음은 마스터노드는 또한 [DataCenter](2022-03-17-DataCenter.md) 서비스를 실행해야한다

然后，启动用于 UniqueService 管理的 service_mgr 。

Next, service_mgr will be start as service_mgr to manage UniqueService .
그리고  [UniqueService](2022-03-17-UniqueService.md) 를 관리하기위한 server_mgr 을 시작한다.

最后，它从 config 中读取 start 这个配置项，作为用户定义的服务启动入口脚本运行。成功后，把自己退出。

Last, it'll read start config entry as the starting point of user-defined services script. It'll quit once it's done.
최후로 config 에서 유저가 정의한 서비스스크립트의 시작점을 읽어서 시작하고 완료되면 끝낸다.

这个 start 配置项，才是用户定义的启动脚本，默认值为 "main" 。如果你只是试玩一下 skynet ，可能有多份不同的启动脚本，那么建议你多写几份 config 文件，在里面配置不同的 start 项。examples 目录下有很多这样的例子。

The start config entry is the user-defined starting script, the default value is "main". If you want to play with Skynet, you can have multiple starting scripts, it's recommended that you have multiple config files with different start entry. Examples can be found in the examples directory.

시작 설정 엔트리는 유저가 정의한 스크립트의 시작점으로 기본값은 main 이다. 만약 당신이 skynet 으로 놀고싶으면 당신은 다수의 시작점을 가질수있다.  여러다른 시작점을위해 다수의 config 파일을 가지기를 추천한다. 예제디렉토리에서 예제를 찾을수 있다.
