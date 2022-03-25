---
layout: post
title: GateServer
date: 2022-03-17T13:53:54.026Z
category: Skynet
lastmod: 2022-03-22T14:10:01.567Z
---

# GateServer
게이트서버(GateServer)는 게임서버의 엑세스층이며, 기본기능은 클라이언트의 연결을 관리하며, 전체 데이터패킷을 분해야여, 로직 처리 서비스에 전달하는일이다.  

skynet 통용 템플리트 lualib/snax/gateserver.lua 를 제공한다. 동시에 gateserver.lua 를 기반으로, 게이트웨이 서비스인 gate.lua 를 구현했다.  

TCP 는 바이트단위 프로토콜이다. 바이트스트림을 조각내어 데이터 패키지로 바꿔야한다. [패키지분리참고](#패키지분리)  

# gateserver
```lua
local gateserver = require "snax.gateserver"

local handler = {}

-- register handlers here

gateserver.start(handler)
```

주: gateserver.start 는 skynet.start 를 호출할것이다. 스스로 skynet.start 가 호출되는 시기를 컨트롤하고 싶다면, handler.embed 를 true 로 설정하자.  

이제 게이트서버가 기동되었다. Handler 는 사용자 정의한 메시지 콜백 함수 세트이다.  

```lua
function handler.connect(fd, ipaddr)
```

새로운 클라이언트가 accept 된후 connect 함수가 호출된다. fd 는socket핸들이다(시스템의 그것과 다름), ipaddr 는 클라이언트의 주소, 예: "127.0.0.1:8000"  

```lua
function handler.disconnect(fd)
```

한 개의 연결이 끊어졌을때, disconnect 가 호출된다. fd 는 어떤 연결인지 나타낸다.  

```lua
function handler.error(fd, msg)
```

当一个连接异常（通常意味着断开），error 被调用，除了 fd ，还会拿到错误信息 msg（通常用于 log 输出）。
error will be called when there is an exception (which usually means disconnect). Besides fd, you'll get an error message: msg (for logging purpose).
연결이상시(통상예상외 연결해제), error 가 호출된다. fd 를 제외하고, 오류정보 msg도 얻을수있다(통상 로그 출력에사용)  

```lua
function handler.command(cmd, source, ...)
```

만약 서비스가 스카이넷 내부 메시지를 처리하고 싶다면, command 메서드를 등록한다. lua 프로토콜의 skynet 메시지를 받았을때, 이 메서드가 호출된다. cmd 는 메시지의 첫번쨰값이며, 통상 한개의 문자열로 예약하며, 이는 어떤명령인지 지정한다. Source는 메시지의 전송 근원이다. 이메서드의 응답값은, skynet.ret/skynet.pack 을 통해 메시지를 전달한쪽에 반환한다.  


Open 과 close 이 두개 명령은 보류되었다. gate 의 listening 포트를 열고, 닫을때 사용된다.  
```lua
function handler.open(source, conf)
```

listening 포트를 열때, 어떤 초기화 작업을 하고싶다면, open 함수를 사용한다. source 는 요청의 근원지이며, conf 는 gate 서비스 기동시의 파라메터이다.  


```lua
function handler.message(fd, msg, sz)
```

전체패킷이 분해된후, message 함수가 호출된다. Msg 는 C 포인터이며, sz 는 숫자이며, 패킷사이즈를 나타낸다.(C포인터가 가르키는 메모리조각의 크기). 주: c포인터는  처리된이후 C함수 skynet_free 를 통해 free 되어야한다. (통상 wraping된 라이브러리 netpack.tostring을 사용하여 이러한 저수준의 데이터처리를 하는것을 추천한다); 아니면 skynet.redirect 를 사용하여 다른 skynet 서비스에서 전달하여 처리하도록 한다.  

```lua
function handler.warning(fd, size)
```

fd 에서 발송대기중인 데이터가 누적 1M 바이트가 넘을때, 이함수가 호출된다. 이 메시지는 무시할수 있다.  


이 함수들에서, 호출하는있는 gateserver 모듈의 메서드는 아래와같다:  


```lua
gateserver.openclient(fd)   -- allow fd receive message
```


每次收到 handler.connect 后，你都需要调用 openclient 让 fd 上的消息进入。默认状态下， fd 仅仅是连接上你的服务器，但无法发送消息给你。这个步骤需要你显式的调用是因为，或许你需要在新连接建立后，把 fd 的控制权转交给别的服务。那么你可以在一切准备好以后，再放行消息。
Every time you received handle.connect, you'll need to call openclient and allow messages in from fd. By default, fd only connects the server but doesn't allow sending messages. It's required to call this function explicitly because you may want to pass the controller of fd to other services when a connection is established. In that case, you can allow messages in after everything is ready.
Handle.connect 를 받는 매번, openclient를 호출하여 fd 로부터의 메시지들이 들어오도록한다. 기본상태는 아래와같다, fd 는 서버에대한 연결이지만, 메시지를 발송할 방법이 없다. 명시적으로 이를  시지들의 진입을 허가할 필요가있다. 기본적으로 fd 는 오직 서버로의 연결일뿐 메시지들을 전달할수없다. 이 단계에서 명시적으로 호출이 필요한것은, 새로운 연결이 만들어진후,  fd의 통제권을 다른 서비스에 넘겨줘야하기 떄문이다. 명시적으로이함수를 호출할필요가있다. 왜냐하면 연결이 성립했을때 fd 의 컨트롤을 다른서비스로 전달해야하기 때문이다. 모든 준비가 끝난후, 다시 메시지를 발행한다. (??????? 이상해 이상해 ????)


``` lua
gateserver.closeclient(fd) -- close fd
```

통상 능동적으로 연결을 kick 한다.


# 패키지분리  

패킷의 포맷:  


모든 패키지는 2개바이트 + 데이터 내용이다.  이 2바이트는 Big-Endian 인코딩되있다. 데이터내용의 크기는 임의이다.  

所以，单个数据包最长不能超过 65535 字节。如果业务层需要传输更大的数据块，请在上层业务协议中解决。
So, every single data package cannot exceed 65535 bytes. If you need to transfer a large data package, please resolve it in in the feature layer .
그래서 하나의 데이터 패키지는 최장 65535 바이트를 넘을수 없다. 만약 로직층에서 더큰 데이터조각의 전송이 필요하다면, 로직 프로토콜 상층에서 해결하라  
(????? 로직층에서하라는건지 그위층??어딘지는모르겠지만 해결하라는건지 모르겠다 ????)

스카이넷은 데이터 디코딩 처리를 위해 netpack 라이브러리를 제공한다. 위치는 Lua-netpack.c 이다. netpack  은 package 포맷에 의거하여 unpack 처리한다. netpack.filter(queue, msg, size) API, 는 타입값("data", "more", "error", "open", "close")을 반환한다. 이것은 IO 이벤트를 대표한다, 그외의 반환값들은 각 이벤트에 필요한 파라메터들이다.  

SOCKET_DATA 이벤트에대해, filter 는 데이터 unpack을 진행한다, 만약 언팩후 한 개의 완전한 메시지가 있을경우,  filter 는 type 을 "data" 로 리턴하며, 이를 fd msg size 뒤에 붙인다. 만약 한나의 메시지가 안된다면 데이터는 큐에 입력되고 리턴값의 type 은 "more" 가된다. queue 는 userdata 이며, netpack.pop 을 통해 queue 에서 메시지를 뺴올수있다.  

"Open", "error", "close" 들과같은 다른타입들은 SOCKET_ACCEPT, SOCKET_ERROR, SOCKET_CLOSE 에 순서대로 대응되는 이벤트이다.  

netpack 호출자는  filter 의 리턴값의 type 을통해 분별하여 이벤트처리를 진행할 수 있다.

넷팩은 가능한한 많은 데이터를, 상층으로 전달한다. 그리고 해쉬테이블을 이용하여 모든 소켓ID와 대응하는 미완성 패킷을 보존하며, 다음 데이터가 도착했을때, 이전에 저장해났던 데이터와 합쳐서, 다시 언팩한다.  

# netpack api
lualib-src/lua-netpack.c 는 이 종류의 데이터 패키지를 처리하는 라이브러리이다.  

```lua
local netpack = require "skynet.netpack"
```

위와 같은코드로 이라이브러리를 로드한다.  

* netpack.pack(msg, [sz]) 문자열 (아니면 길이를 가지는 C포인터)을 2바이트 헤더를 가지는 패키지로 변환한다. 이 API 는 lightuserdata 와 한개의 number 이다.  socket.write 함수를 사용하여 전송할수 있다 ((socket.write 는 메모리 해제를 책임진다)  


* netpack.tostring(msg, sz)  handle.message 함수로부터 받은 sz 와 msg 를 lua 문자열로 변환한다. 동시에 msg 가 사용하는 씨메모리를 릴리즈한다.  

netpack 은 또한 gate 서버의 구현을 위한 내부 API 들을 제공한다

주: 어떻게 디버그하는지 모른다면 netpack 을 직접 사용하지 말것!

# Gate Server
Service/gate.lua 는 완전한 gate server 이다. 이것은 또한 snax.gateserver 의 예제로 사용될수있다. 다른 예쩨는 examples/watchdog.lua 에서 찾을수있다. 이것은 service/gate.lua 에서 서비스를 시작하고 외부연결들에 의해필요한 메시지들을 다시 전달한다.

gate serve 는 기동후 listen하지 않는다. gateserver 가 listen 하게하고싶다면  lua 프로토콜로 open 명령을 보내어 기동파라메터를 전달한다. 아래는 예제이다  

```lua
skynet.call(gate, "lua", "open", {
    address = "127.0.0.1", -- listen address 127.0.0.1
	port = 8888,    -- listen port 8888
	maxclient = 1024,   -- max connection 1024 
	nodelay = true,     -- TCP_NODELAY property
})
```

주: socket 라이브러리와 같이 이 모듈을 사용할수없다. 이 모듈이 소켓메시지를 가져가버린다.

# 다른방법

스카이넷은 게이트서버의 구현에 어떠한 제한도 가지고있지 않다. 직접 아래모듈중 하나를 사용할수 있다.

(tcp)http://blog.codingnow.com/2016/03/skynet_tcp_package.html

(websocket)https://gist.github.com/cloudfreexiao/851af8acff4ae5cf225d2ff19a94d327

(websocket-gate)https://github.com/xzhovo/skynet-websocket-gate

(websocket-gate reponse转发模式)https://github.com/hanxi/skynet-demo/blob/master/service/ws_gate.lua

출처: <https://github.com/cloudwu/skynet/wiki/GateServer> 
