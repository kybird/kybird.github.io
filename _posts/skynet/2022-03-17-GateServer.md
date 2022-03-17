---
layout: post
title: GateServer
date: 2022-03-17T13:53:54.026Z
category: Skynet
lastmod: 2022-03-17T14:14:53.864Z
---

网关服务 (GateSever) 是游戏的接入层, 基本功能是管理客户端的连接, 分割完整的数据包, 转发给逻辑处理服务.

GateServer is the access layer of a game server and the basic functionalities are managing client connections, unpacking data packages, redirecting them to the logic layer.

게이트서버(GateServer)는 게임서버의 엑세스층이다 그리고 기본기능들은 클라이언트 연결을 관리하고 데이터패키지를 분해하고 로직서비스에 전송하는일이다.

skynet 提供了一个通用模板 lualib/snax/gateserver.lua. 同时基于 gateserver.lua, 实现了一个网关服务 gate.lua.
Skynet provides a general template in lualib/snax/gateserver.lua and also implements GateServer in gate.lua based on gateserver.lua.
Skynet 은 lualib/snax/gateserver.lua 라는 한 개의 통용 템플리트를 제공한다. 동시에 gateserver.lua 를 기반으로 한 개의 게이트웨이서비스를 구현한 gate.lua 를 제공한다.


TCP 是面向字节流的协议，我们需要把字节流流切割成数据包, 具体的方式见 分包.
TCP is a byte-oriented protocol, we need to unpack byte stream into data packages, check detail in 分包.
TCP 는 바이트단위 프로토콜이다. 우리는 바이트스트림을 풀어서 데이터패키지로 바꿔야한다. 패키지분리참고

gateserver
local gateserver = require "snax.gateserver"

local handler = {}

-- register handlers here

gateserver.start(handler)
注：gateserver.start 会调用 skynet.start 。如果你希望自己控制 skynet.start 的时机，那么可以将 handler.embed 设置为 true 。
Notice : gateserver.start will call skynet.start unless handler.embed == true .
Handler.embed == true 가 아닐경우 게이트서버는 skynet.start 를 호출한다.

这样就可以启动一个网关服务。handler 是一组自定义的消息回调函数.
Then gate server gets started. Handler is a set of user-defined callback functions.
이후 게이트서버는 시작된다. 해늘러는 정의되지않은 콜백함수로 지정했다ㅏ.

function handler.connect(fd, ipaddr)
当一个新客户端被accept后，connect 方法会被回调。 fd 是socket句柄 (不是系统fd). ipaddr是客户端地址, 例如 "127.0.0.1:8000".
connect function will be called when a new client has been accepted. fd is a socket handler (not the system fd). ipaddr is the client address, eg: "127.0.0.1:8000".
새로운 클라이언트가 accept 된후 connect 함수가 호출된다 fd 는 시스템의 그것과 다름,  ipaddr 는 클라이언트의 주소이다 예: "127.0.0.1:8000"


function handler.disconnect(fd)
当一个连接断开，disconnect 被回调，fd 表示是哪个连接。
disconnect will be called for disconnection, fd is the specified connection.
한 개의 연결이 끊어졌을때 disconnect 가 호출된다. Fd 는 명시한 연결이다

function handler.error(fd, msg)
当一个连接异常（通常意味着断开），error 被调用，除了 fd ，还会拿到错误信息 msg（通常用于 log 输出）。
error will be called when there is an exception (which usually means disconnect). Besides fd, you'll get an error message: msg (for logging purpose).
연결이상시(통상 이외 연결해제) error 가 호출된다. Fd 를 제외하고 msg 오류정보도 얻을수있다(통상 로그 출력에사용)

function handler.command(cmd, source, ...)
如果你希望让服务处理一些 skynet 内部消息，可以注册 command 方法。收到 lua 协议的 skynet 消息，会调用这个方法。cmd 是消息的第一个值，通常约定为一个字符串，指明是什么指令。source 是消息的来源地址。这个方法的返回值，会通过 skynet.ret/skynet.pack 返回给来源服务。
If you want your service to handle Skeynet internal messages, you can register the command function. This function will be called when receiving Skynet messages of lua protocol. cmd is the first value of a message and it's usually a string telling what the command it is. source is the source address of the message. The return of this function will be passed to the caller through skynet.ret/skynet.pack.

만약 당신의 서비스가 스카이넷의 내부 메시지를 처리하고 싶다면 command 함수를 등록할수있다. 이함수는 루아 프로토콜의 스카이넷 메시지를 받을때 호출된다.  cmd 는 메시지의 첫번쨰값을 의미하고 이것은 보통 문자열이며 어떤 명령인지 알려준다. Source 는 메시지의 전송의 근원이며 이함수의 응답값은 skynet.ret 나 skynet.pack 을 통해  전달된다.

open 和 close 这两个指令是保留的。它用于 gate 打开监听端口，和关闭监听端口。
open and close functions are kept, which are used for closing/opening a listening port for the gate.
Open 과 close 함수는 남겨지었다 게이트의 리스닝포트의 열고닫기에 사용된다.

function handler.open(source, conf)
如果你希望在监听端口打开的时候，做一些初始化操作，可以提供 open 这个方法。source 是请求来源地址，conf 是开启 gate 服务的参数表。
If you need init functions after opening a listening port, use the open function. source is the source address of the request, config is the parameters to start a gate server.
리스닝포트를 연후 초기화함수가 필요할경우 Open 함수를 사용한다. Source 는 요청의 source 주소이며 config 는 게이트서버실행 파라메터이다.

function handler.message(fd, msg, sz)
当一个完整的包被切分好后，message 方法被调用。这里 msg 是一个 C 指针、sz 是一个数字，表示包的长度（C 指针指向的内存块的长度）。注意：这个 C 指针需要在处理完毕后调用 C 方法 skynet_free 释放。（通常建议直接用封装好的库 netpack.tostring 来做这些底层的数据处理）；或是通过 skynet.redirect 转发给别的 skynet 服务处理。
The message function will be called when a package has been unpacked. msg is a C pointer, sz is a number meaning the size of the package (the size of memory c pointer points to). Note: the c pointer needs to be freed using the C function skynet_free once it's processed. (It's recommended to use netpack.tostring for basic data processing); or use skynet.redirect redirecting it to other Skynet services.
패킷이 분해된후 message 함수가 호출된다. Msg 는 C 포인터이다. Sz 는 패킷사이즈의 크기를 의미한다 (C 포인터가가르키는 메모리의 크기 주: c포인터는  처리된이후 C함수 skynet_free 를 통해 free 되어야한다) 그렇지않으면 skynet.redirect 를 사용하여 그것을 다른 스카이넷 서비스들에게 재발송해야한다.

function handler.warning(fd, size)
当 fd 上待发送的数据累积超过 1M 字节后，将回调这个方法。你也可以忽略这个消息。
This function will be called if the accumulated pending data from fd is bigger than 1M, you can ignore this message though.
이함수는 fd에 처리가 지연되고있는 패킷이 누적으로 1M 보다 커지면 호출된다. 이메시지는 무시할수있다.

在这些方法中，还可以调用 gateserver 模块的方法如下：
Out of all these functions, here is how to call gateserver module function:
이모든 함수들을 통하여. 게이트서버 모듈함수의 호출방법


gateserver.openclient(fd)   -- 允许 fd 接收消息
每次收到 handler.connect 后，你都需要调用 openclient 让 fd 上的消息进入。默认状态下， fd 仅仅是连接上你的服务器，但无法发送消息给你。这个步骤需要你显式的调用是因为，或许你需要在新连接建立后，把 fd 的控制权转交给别的服务。那么你可以在一切准备好以后，再放行消息。
Every time you received handle.connect, you'll need to call openclient and allow messages in from fd. By default, fd only connects the server but doesn't allow sending messages. It's required to call this function explicitly because you may want to pass the controller of fd to other services when a connection is established. In that case, you can allow messages in after everything is ready.

Handle.connect 를 받는 매번, .당신은 openclient 를호출하여 fd 로부터의 메시지들의 진입을 허가할 필요가있다. 기본적으로 fd 는 오직 서버로의 연결일뿐 메시지들을 전달할수없다. 명시적으로이함수를 호출할필요가있다. 왜냐하면 연결이 성립했을때 fd 의 컨트롤을 다른서비스로 전달하고 싶을수도있기 떄문이다. 이럴경우 모든것이 준비된후 메시지들을 허용할수있다.

gateserver.closeclient(fd) -- 关闭 fd
通常用于主动踢掉一个连接。
Close a connection explicitly.
명시적으로 연결의 해제

分包
包格式：

The format of data package:
데이터패킷의 포맷

每个包就是 2 个字节 + 数据内容。这两个字节是 Big-Endian 编码的一个数字。数据内容可以是任意字节。

Each package contains two bytes header + payload. These two bytes are Big-Endian encoded number. The payload can be any size.
각 패키지는 두바이트의 헤더와 페이로드로 구성되어있다. 이; 두바이트는 BIG-Endian 인코딩된 숫자이다. Payload 의 크기는 어떤크기라도 가능하다.

所以，单个数据包最长不能超过 65535 字节。如果业务层需要传输更大的数据块，请在上层业务协议中解决。
So, every single data package cannot exceed 65535 bytes. If you need to transfer a large data package, please resolve it in in the feature layer .

그래서 모든 한 개의 데이터패키지는 65535 바이트를 넘을수없다. 만약  더큰 데이터 페키지의 전송이 필요하다면 기능층에서 해결하라.

skynet 提供一个netpack库用于处理分包问题， 位于lua-netpack.c。netpack根据包格式处理分包问题， netpack.filter(queue, msg, size)接口，它返回一个type(“data”, “more”, “error”, “open”, “close”)代表具体IO事件，其后返回每个事件所需参数。
Skynet provides a netpack lib for processing data unpacking, it's in lua-netpack.c. netpack unpacks the data based on data format, netpack.filter(queue, msg, size) API returns a type value (“data”, “more”, “error”, “open”, “close”) to represents an IO event and the parameters needed for this event.
스카이넷은 데이터언팩처리를 위해 netpack 라이브러리를 제공한다. Lua-netpack.c 에 있다. Netpack  은 data 포맷에 기반하여 데이터를 unpack 한다. Netpack.filter(queue, msg, size) API 는 타입값을 리턴한다("data", "more", "error", "open", "close") 이것은 IO 이벤트를 의미한다, 그리고 파라메터들은 이이벤트에 필요한것이다.

对于SOCKET_DATA事件，filter会进行数据分包，如果分包后刚好有一条完整消息，filter返回的type为”data”，其后跟fd msg size。 如果不止一条消息，那么数据将被依次压入queue参数中，并且仅返回一个type为”more”。 queue是一个userdata，可以通过netpack.pop 弹出queue中的一条消息。

filter unpacks SOCKET_DATA event, if there is a complete message right after unpacking, filter returns type of "data" followed by fd msg size. If there is more than one message, data will be pushed into a queue parameter and return a type: "more". queue is a type of userdata and use netpack.pop to pop message from queue.
SOCKET_DATA 이벤트에대해, filter 는 데이터언팩을 진행한다, 만약 언팩후 한 개의 완전한 메시지가 있을경우  filter 는 type 을 "data" 로 리턴한다. 그리고 fd msg size 를 덛붙인다. 만약 한 개의 메시지가 안된다면 데이터는 큐에 입력되고 리턴값의 type 은 "more" 가된다 queue 는 한 개의 user-data 이다. netpack.pop 을 통해 queue 에서 한 개의 메시지를 뺴올수있다.

其余type类型”open”，”error”, “close”分别SOCKET_ACCEPT, SOCKET_ERROR, SOCKET_CLOSE事件。

Other types like "open", "error", "close" are for SOCKET_ACCEPT, SOCKET_ERROR, SOCKET_CLOSE events respectively.
"Open", "error", "close" 들과같은 다른타입들은 SOCKET_ACCEPT, SOCKET_ERROR, SOCKET_CLOSE 에 순서대로 대응되는 이벤트이다.

netpack的使用者可以通过filter返回的type来分别进行事件处理。
The caller of netpack should process events based on the returned type.
Netpack 호출자는  filter 의 리턴값의 type 을통해 이벤트의 처리를 할수있다.

netpack会尽可能多地分包，交给上层。并且通过一个哈希表保存每个套接字ID对应的粘包，在下次数据到达时，取出上次留下的粘包数据，重新分包.

netpack unpacks as much data as possible and passes it to the upper layer. And it's using a hash table to map each socket ID to its sticky package, when extra data arrives it retrieves the remaining sticky package and unpacks it again.
넷팩은 가능한한 많은 데이터를 상층으로 전달한다. 그리고 해쉬테이블을 이용하여 각소켓아이디와 패키지를 매핑한다. 추가데이터가 도착했을때 이전에 남은 데이터를 다시 가져와서 다시 언팩한다.

netpack api
lualib-src/lua-netpack.c 是处理这类数据包的库。
lualib-src/lua-netpack.c is the lib to process this type of data package.
lualib-src/lua-netpack.c 는 이종류의 데이터 패키지를 처리하는 라이브러리이다.

local netpack = require "skynet.netpack"
可以加载这个库。
Use the above code to load this lib.
위와 같은코드로 이라이브러리를 로드한다.

netpack.pack(msg, [sz]) 把一个字符串（或一个 C 指针加一个长度）打包成带 2 字节包头的数据块。这个 api 返回一个lightuserdata 和一个 number 。你可以直接送到 socket.write 发送（socket.write 负责最终释放内存）。
netpack.tostring(msg, sz) 把 handler.message 方法收到的 msg,sz 转换成一个 lua string，并释放 msg 占用的 C 内存。

netpack.pack(msg, [sz]) pack a string (or a c pointer with a size) into a data package with 2 bytes header. This function returns a lightuserdata and a number. You can send it with socket.write (it'll handle memory free.)
한 개의 문자열 (아니면 길이를가지는 C포인터) 를 2바이트 헤더를 가지는 패키지로 변환한다. 이함수는 한 개의 lightuserdata 와 숫자를 리턴한다. Socket.write 함수를 사용하여 이것을 전송할수있다. (이것은 메모리 해제를 처리한다)

netpack.tostring(msg, sz) convert msg and sz from handle.message function into a lua string and release the c memory of msg.
Netpack.tostring(msg, sz)  handle.message 함수로부터 받은 sz 와 msg 를 lua 문자열로 변환한다. 동시에 msg 가 사용하는 씨메모리를 릴리즈한다.

netpack 还有一些内部 api 用于 gate server 的实现。

netpack also provides other internal APIs for the implementation of gate server.
Netpack 은 또한 gate 서버의 구현을 위한 내부 API 들을 제공한다

注意：除非你认为已经了解了细节和具备出错调试的能力，否则请不要直接使用 netpack 。
Note: unless you know the detail and know how to debug it, don't use netpack directly.
주: 어떻게 디버그하는지 모른다면 netpack 을 직접 사용하지 말것!

Gate Server
service/gate.lua 是一个实现完整的网关服务器，同时也可以作为 snax.gateserver 的使用范例。examples/watchdog.lua 是一个可以参考的例子，它启动了一个 service/gate.lua 服务，并将处理外部连接的消息转发处理。

service/gate.lua is a complete gate serer, it can also be used as an example of snax.gateserver. Another example can be found in examples/watchdog.lua, which starts a service in service/gate.lua and redirects messages needed by external connections.

Service/gate.lua 는 완전한 gate server 이다. 이것은 또한 snax.gateserver 의 예제로 사용될수있다. 다른 예쩨는 examples/watchdog.lua 에서 찾을수있다. 이것은 service/gate.lua 에서 서비스를 시작하고 외부연결들에 의해필요한 메시지들을 다시 전달한다.

gate 服务启动后，并非立刻开始监听。要让 gate 服务器开启监听端口，可以通过 lua 协议向它发送一个 open 指令，附带一个启动参数表，下面是一个示范：

gate server won't be listening right away when it's started. To make it listen to a port, use an open command with starting parameters using lua protocol, here is an example:
게이브서버기동후  리슨하지 않는다. 게이터서버기동후 리슨하고싶다면  lua 프로토콜로 open 명령을 보내어 기동파라메터 테이블를 전달한다. 아래는 예제이다

skynet.call(gate, "lua", "open", {
    address = "127.0.0.1", -- 监听地址 127.0.0.1
	port = 8888,    -- 监听端口 8888
	maxclient = 1024,   -- 最多允许 1024 个外部连接同时建立
	nodelay = true,     -- 给外部连接设置  TCP_NODELAY 属性
})
注: 这个模板不可以和 Socket 库一起使用。因为这个模板接管了 socket 类的消息。
Note: It's not allowed to use this module with Socket lib because it takes over socket messages.
주:  소켓라이브러리와 같이 이 모듈을 사용할수없다. 이모듈이 소켓메시지를 가져가버린다.

其它方案 Other Solutions
다른방법
skynet 并不限制你怎样编写网关，比如你还可以使用以下模块:
Skynet doesn't have a limitation on how you implementing gate server, you can use the following modules:
스카이넷은 게이트서버의 구현에 어떠한 제한도 가지고있지 않다. 당신은 아래의 모듈을 사용할수있다.

(tcp)http://blog.codingnow.com/2016/03/skynet_tcp_package.html

(websocket)https://gist.github.com/cloudfreexiao/851af8acff4ae5cf225d2ff19a94d327

(websocket-gate)https://github.com/xzhovo/skynet-websocket-gate

(websocket-gate reponse转发模式)https://github.com/hanxi/skynet-demo/blob/master/service/ws_gate.lua
