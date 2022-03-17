---
layout: post
title: DebugConsole
date: 2022-03-17T13:53:54.026Z
category: Skynet
lastmod: 2022-03-17T14:14:55.675Z
---

skynet 自带了一个调试控制台服务。你需要在你的启动脚本里启动它。
Skynet has a built-in debug console and you can run it from starting script.
스카이넷은 내장 디버그콘솔을 가지고있고 기동스크립트에서 실행할수 있습니다

skynet.newservice("debug_console",8000)

这里的示例是监听 8000 端口，你可以修改成别的端口。
In this example, it's listening on port 8000 and it's configurable.
이예제에서는,  8000포트를 리슨하고 이것은 설정가능하다.
出于安全考虑，调试控制台只能监听本地地址 127.0.0.1 ，所以如果需要远程使用，需要先登录到本机，然后再连接。
For safety purpose, debug console only listen on local address: 127.0.0.1, to use it remotely, please login into local machine and then connect.
보안을 위해 디버그콘솔은 오직 127.0.0.1지역주소만을 리슨한다.리모트에서 사용하려면 해당머신에 로그인한후 연결한다

可以用 telnet 或 nc 登录调试控制台。启动后会显示
You can use telnet or nc to log in and debug the console. After starting it shows:
Telnet 이나 nc 를 사용하여 로그인할수 있다. 시작후 다음이 보여진다

Welcome to skynet console
表示连接成功。
That means it connects successfully.
이게보여지면 연결이 성공한것이다.

注：由于 skynet 使用自己的 IO 库，所以很难把 libreadline 接入（不能在 readline 的 hook 中 yield）。如果你希望在控制台中使用 readline 的 history 等特性，可以自己使用 rlwrap 。
Note: Because Skynet is using its own IO lib, so it's hard to hook up libreadline (you cannot use yield in hook function of readline). If you want to use the history of readline in your debug console, try using rlwrap.
주: 스카이넷은 자신만의 IO라이브러리를 사용하기 때문에 libreadline 을통합하기가 어렵다( readline 의 hook 함수안에서 yield를 사용할수 없다). 만약 debug console 에서 readline 의 히스토리를 사용하기를 원한다면 rlwrap 을 사용해보아라

这时，你可以输入调试指令，输入 help 可以列出目前支持的所有指令。这份文档可能落后于实际版本，所以应以 help 列出的指令为准。
Then, you can input debug commands, input help to list all supported commands. These commands are subject to change, check help for the latest.
그러면, 디버그명령을 입력하고 help 를 입력하여 모든 지원하는 명령을 리스트할수있다. 이들명령들은계속변경될수 있으니 help명령으로 확인한다
命令的一般格式是 命令 地址 ，有些命令不带地址，会针对所有的服务。当输入地址时，可以使用 :01000001 这样的格式指代一个服务地址：由冒号开头的 8 位 16 进制数字，也可以省略前面两个数字的 harbor id 以及接下来的连续 0 ，比如 :01000001 可以简写为 1 。所有活动的服务可以输入 list 列出。
A command usually has this format: command address, some of the commands have no address and are for all services, try using: 01000001 this format to specify a service address: column prefix 8 digits hex value, you can emit the first two digits and following 0s, eg: 01000001 is short for 1. All active services can be listed using the list command.
명령의 일반적 포맷은 command address 이다. 어떤명령들은 주소가 필요치않다.  모든 서비스들에대하여 주소를 입력할시 :01000001 이와같은포맷은 한 개의 서비스주소를 의미한다.  : 로시작하는 8자리의 16진수이다. 첫번째 두개의 숫자harborId 와 연속된 0은 생략할수있으며 예를들어 :01000001 은 1로 줄여쓸수있다. 모든 살아있는 서비스는 list명령으로 일람할수있다.
同时也支持使用 curl 发送 HTTP GET 请求来执行命令：
It also supports using curl to send HTTP GET requests to execute commands：
동시에 curl 을통해 http GET 방식의 명령을 사용할수도있다.
> curl --http0.9 'http://127.0.0.1:8000/list'
Welcome to skynet console
:01000004       snlua cmaster
:01000005       snlua cslave
:01000007       snlua datacenterd
:01000008       snlua service_mgr
:0100000a       snlua protoloader
:0100000b       snlua console
:0100000c       snlua debug_console 8000
:0100000d       snlua simpledb
:0100000e       snlua watchdog
:0100000f       snlua gate
<CMD OK>
> curl --http0.9 'http://127.0.0.1:8000/call/:01000008/"LIST"'
Welcome to skynet console
1       protoloader::0100000a
n       1
<CMD OK>
常用的针对所有 lua 服务的指令有：
The most used commands for Lua services are:
루아서비스를 위한 자주사용되는 명령어들은 다음과 같다
	• list 列出所有服务，以及启动服务的命令参数。
	• list lists all commands and the command params to start service. 
	• List 모든 명령어들과 서비스를 실행하기위한 명령어파라메터를 리스팅한다 //?? Help 잘못씀?
	• gc 强制让所有 lua 服务都执行一次垃圾回收，并报告回收后的内存。
	• gc force Lua service execute garbage collection and report updated memory usage.
	• Gc 강제로 모든 루아서비스가 가버지컬렉트하도록함. 동시에 갱신된 메모리사용량을 보고함
	• mem 让所有 lua 服务汇报自己占用的内存。（注：它只能获取 lua 服务的 lua vm 内存占用情况，如果需要 C 模块中内存使用报告，请参考 MemoryHook 。
	• mem make all Lua services report their memory usage. (Note: it only contains the Lua VM memory of Lua service, if c module memory usage is also needed, please refer to MemoryHook.
	• Mem 모든 루아서비스들이 자신의 메모리 사용량을 레포트하도록함 (주: 오직 루아서비스의 루아 VM 메모리만을 포함한다 만약 씨 모듈의 메모리사용량이 필요하다면 MemoryHook 을 참고)
	• stat 列出所有 lua 服务的消息队列长度，以及被挂起的请求数量，处理的消息总数。如果在 Config 里设置 profile 为 true ，还会报告服务使用的 cpu 时间。
	• stat list the message queue length of all Lua services and number of the pending requests, processed messages. If profile is set to true in Config, it'll report CPU time usage.
	• Stat 모든 로아서비스들의메시지큐의길이와 지연된 요청을 리스트한다
	• service 列出所有的唯一 lua 服务。并显示出请求还不存在的服务被挂起的请求。
	• service list all single Lua services and show pending requests that send to non-exist services.
	• Service 모든 단일 루아서비스를 리스트하고 존재하지 않은서비스에게 전송되어 지연된 요청을 보여준다.
	• netstat 列出网络连接的概况。
	• netstat list network status.
	• Netstat 네트워크 상태를 리스트한다
注意，由于这些指令是挨个向每个服务发送消息并等待回应，所以当某个 lua 服务过载时，可能需要等待很长时间才有返回。
Note, because all these commands are sent to all services and wait for a response, so if one Lua service is overloaded, it might take a while to get a response.
주, 이모든 명령들은 모두 서비스에게 전달되어 응답을 기다리기때문에 어떤 루아서비스가 로드가 오버되었다면 응답을 받을떄까지 시간이 걸릴수있다.
针对单个 lua 服务的指令有：
단일루아서비스의 명령은 다음과 같다.
	• start service_name 用 skynet.newservice 启动一个新的 lua 服务。
	• start service_name use skynet.newservice to start a new Lua service.
	• Start service_name 은 skynet_newservice 를 사용하여 새로운 lua service 를 시작한다
	• snax service_name 用 snax.newservice 启动一个新的 snax 服务。
	• start service_name use snax.newservice to start a new snax service.
	• Snax service_name 은 snax.newservice 를 이용하여 새로운 snax service 를 시작한다
	• exit address 让一个 lua 服务退出。
	• exit address quit a Lua service.
	• Exit addresss 루아서비스를 종료한다
	• kill address 强制中止一个 lua 服务。
	• kill address force quit a Lua service.
	• Kill address 루아서비스 종료를 강제한다
	• info address 让一个 lua 服务汇报自己的内部信息，参见 Profile 。
	• info address make a Lua service report its own internal info, see Profile.
	• Info address 는 루아서비스가 자신의 정보를 리포트하도록한다 Profile 참조하라
	• signal address sig 向服务发送一个信号，sig 默认为 0 。当一个服务陷入死循环时，默认信号会打断正在执行的 lua 字节码，并抛出 error 显示调用栈。这是针对 endless loop 的 log 的有效调试方法。注：这里的信号并非系统信号。
	• signal address sig sends a single to service, sig defaults to 0. when a service has an infinite loop, the default signal will terminate Lua binary code, and raise an error with the call stack. This is the best way to debug the log of endless loops. Note: the signal here is not the same thing as the system signal.
	• Signal address sig 서비스에개 한개의메시지 sig를 전송한다 sig기본값은0이다. 섭스가 무한루프에들어갈시 기본 시그날은 루아의 바이너리코드를 정지한다. 그리고 콜스택과 함께 에러를 발생한다. 이것은 endless loop 를 디버깅하는 최선의 방법이다 주: 여기의시그날은 시스템의 시그날과 같은것이 아니다.
	• task address 显示一个服务中所有被挂起的请求的调用栈。
	• task address shows the call stack of all pending requests from a specified service.
	• Task address 명시한서비스의 모든 대기중요청의 콜스택을 보여준다
	• debug address 针对一个 lua 服务启动内置的单步调试器。 http://blog.codingnow.com/2015/02/skynet_debugger.html
	• debug address enables an enbeded single debugger for a specified Lua service.
	• Debug address  명시한 루아서비스에 단일 디버그를 활성화한다
	• logon/logoff address 记录一个服务所有的输入消息到文件。需要在 Config 里配置 logpath 。
	• logon/logoff address record all input messages to a file. logpath needs to be set in Config.
	• Logon/logoff address 는 모든 입력메시지들을 파일로 기록한다 logpath 는 Config 에 설정해야한다
	• inject address script 将 script 名字对应的脚本插入到指定服务中运行（通常可用于热更新补丁）。
	• inject address script run script file on specified service. (usually, it's used for hot patches).
	• Inject address script 는 script 이름에 해당하는 스크립트를 지정한 서비스에 삽입하여실행한다 (핫패치시에 사용된다)
	• call address 调用一个服务的lua类型接口，格式为: call address "foo", arg1, ... 注意接口名和string型参数必须加引号,且以逗号隔开, address目前支持服务名方式。
	• call address call Lua interface of a service. the format is: call address "foo", arg1, ..., Note the interface name and string type must be quoted and split by ",", address supports service name.
	• Call address 는 서비스의 루아인터페이스를 호출한다. 포맷은 call Address "foo", arg1,…인터페이스 이름과 문자열형식은 따옴표를 사용해야하며 , 로 나누어져야한다 주소는 서비스이름을 지원한다
	
	
