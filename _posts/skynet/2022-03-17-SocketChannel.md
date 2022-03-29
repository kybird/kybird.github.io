---
layout: post
title: SocketChannel
date: 2020-01-02 19:20:23 +0900
category: Skynet
lastmod: 2022-03-17T14:19:58.944Z
---


요청응답모드는 외부서비스와 상호작용시  모두 사용되는 제일 일반적으로 사용되는 모드들중 하나이다. 일반적으로 프로토콜의 설계 방식은 두개이다.

1. 모든 요청패키지에 대응하는 하나의 응답패키지, tcp프로토콜이 시간순서를 보장한다. redis 의 프로토콜이 전형적인 예이다, 하지만 응답을 받을 필요가 없어질 때만 다음 요청을 계속 발송할 수 있다.

2. 모든 요청을 시작할때 유일한session 표식을 달려있다, 발송 응답시, 이 표식을 달아준다. 이러한 설계는 모든 요청이 꼭 응답이 필요하지는 않게 할수 있다, 게다가 먼저 받은 요청을 먼저 응답할 필요가 없어진다. MongDB 의 통신 프로토콜이 이렇게 설계 되어있다.

첫 번째 모드에서는, skynet의 Socket API로 구현하기 쉽지만, coroutine에서 socket을 읽으면, 읽는 과정이 블럭돼기 때문에, 처리량이 떨어진다.. (이전 응답을 받지 못했을 때, 다음 요청전송불가)

두 번째 모드에서는, skynet.fork 를 사용하여 새로운 스레드를 기동하여 응답을 받도록 해야하며, 스스로 요청에 대응해야하며, 구현하기 비교적 번거롭다.

그래서, skynet 은 고수준레벨에 WRAPPER인 socket channel 을 제공한다.

```lua
local sc = require "skynet.socketchannel"

local channel = sc.channel {
  host = "127.0.0.1",
  port = 3271,
}
```

이렇게 channel 객체를 생성하였다, 이중 host 는 ip 주소나 DNS 이름일수 있다, port 는 포트 이다.

이리하여, 모드1에서 작업을 할수 있다:

```lua
local resp = channel:request(req [, response[, padding]])
```


여기, req 는 문자열이다. 즉 요청패키지이다. response 는 함수이며, 응답패키지를 받는데 사용된다. 반환값은 문자열이다, response 함수의 응답패키지 내용이 문자열이기 때문이다. response 함수는 아래모양처럼 정의 해야한다.
```lua
function response(sock)
  return true, sock:readline()
end
```

sock 은 request 로 입력된 하나의 객체이며, sock 은 두개의 메서드 read(self, sz) 와 readline(self, sep)를 가지고있다. read 는 지정한 바이트수를 읽어온다; readline 은 sep(기본값 "\n")로 분할한 하나의 문자열을 가져온다.(분할자 자체는 포함하지 않는다).

response 함수의 첫번째 응답값은 boolean 이 필요하다, true 일경우 프로토콜 디코딩이 정상임을 의미하고, false 일경우 프로토콜에 오류가 있음을 의미한다.

response 함수내 어떠한 이상 그리고 sock:read 나 sock:readline 가 오류가 발생하면, 모두 error 형식으로 request 한 호출자에게 주어야한다.

---

만약 프로토콜모드가 2번째 종류인 상황이라면, channel 을 생성할때 하나의 통용할수 있는 response decode 함수를 제공해야한다.

```lua
local channel = sc.channel {
  host = "127.0.0.1",
  port = 3271,
  response = dispatch,
}
```

여기 dispatch 는 응답패킷의 디코딩 함수이며, 위에서 말한 모드1에서 디코딩함수와 비슷하다. 하지만 반환값은 세개이다. 첫번째는 응답패킷의 session, 두번째는 디코딩정상여부(동모드 1), 세번째는 응답내용이다.


socket channel 은 생성할때 제공한 response 함수에 의존하여 동작모드가 1인지 2인지 결정하게 된다.
모드 2일때, request 의 파라메터는 모두 변화한다. 제 2번째 파라메터는 더이상 response 함수(이미생성시 주어짐) 가 아니고, session 이 된다. 이 session 은 어떠한형식이든 될수 있다, 하지만 response 함수가 반환하는 형식과 동일해야한다. socket channel 은 session 값이 request 반환값이 올바른지 도와줄 것이다.


channel:close() 채널을 닫는다, 통상 능동적으로 닫을 필요가 없고, gc 가 channel 이 점유한 자원을 회수할것이다.

---

socket channel 생성시에, 바로 연결이 생성되지 않는다. 만약 아무것도 하지 않는다면, 연결의 생성은 첫번째 요청이 있을때까지 지연되게 된다. 이러한 수동적 연결생성의 과정은 끊임없이 시도되며, 처음에 연결되지 않았더라도, 계속 다시 시도할것이다.

능동적으로 channel:connect(true) 를 호출하여 연결을 한번 시도할수 있다. 만약 실패하면 error 를 던진다. 여기 파라메터 true 는 한번만 시도함을 의미한다, 만약 이 파라메터를 주지 않으면, 계속해서 연결을 시도할 것이다.

연결은 어떠한 request 전에(오직 직전의 동작이 연결이 끊어진 상태라는걸 감지하면 재연결을 시작할 수 있다), 그러니 socket channel 은 인증과정을 지원하며, 연결후 , 바로 상호작용을 허용한다. 만약 이기능을 활성화했다면 channel생성시, response 함수와 동일한 auth 함수를 추가해야한다.이 함수는 channel 객체를 입력받을수 있다. auth 함수는 반환값이 필요없으며, 만약 인증이 실패하면, auth 함수내에서 error 를 던지기만 하면된다.

---

어떠한 상황에서도 연결이 끊어질수 있기 때문에, 어떠한 한번의 request 모두 error 를 던지고 실패할수 있으며, socket channel 은 다음한번의 request 시 연결 생성을 재시도한다. 주의: 재연결은  이미발생한 error 의 요청을 다시 시도하지 않는다, 연결이 끊어짐은 내부에 숨겨진것이 아니다. 그러니, 만약 필요하다면, 요청실패시에, 로직층에서 요청을 다시 보내는 작업을 해야한다.

---

socket channel 또한 발송만하고 응답을 받지 않을수 있다. request 호출시 response 를 쓰지 않으면 된다.

channel:response 는 단방향 패킷받기로 사용할수 있다.

```lua`
channel:request(req)
local resp = channel:response(dispatch)

-- 위아래는 동일하다

local resp = channel:request(req, dispatch)
```

---

request 는 또한 세번째 파라메터 padding 을 가지고 있다, 이는 엄청난 크기의 메시지를 여러개의 패킷으로 나누어 배포하는데 사용된다 (만약 당신의 프로토콜이 지원한다면)

padding 은 오직 session 모드하에서만 사용된다. padding 은 하나의 table 을 필요로 하며, 테이블 안에는 약간의 문자열이있다(모든 문자열은 모두 session 모드의 response 함수의 디코딩을 거친다). 만약 padding 파라메터를 제공하면, socket channel은 req 와 padding그룹의 문자열, 이용하여 socket 의 저우선순위 채널로 보내진다 (socket.lwrite 사용).

이러한 사용법에서 response 함수, 반환값에 padding 값이 추가될것이다, 즉, 모드1반환 succ:boolean data:String padding:boolean 새개의 값, 모드2에대해서 반환값은 session: number succ:boolean data:string padding:boolean 네개의 값.

padding 은 이후에 메시지의 뒷부분이 더있는지를 표명한다.

만약응답메시지가 다수의 잛은 메시지의 합성이라면. channel:request 는 하나의 테이블을 반환한다, 안에는 모든 짧은 메시지의 내용이 있고, 호출가가 이 짧은 메시지를 연결해야한다.

# 예비주소 
如果你想在无法连接指定的 ip 地址时,让 channel 尝试连接备用地址,可以创建 channel 对象时,给出 backup 表.这是一个数组,每一项是一个地址.
만약 연결하고자한 지정한 IP 주소에 연결하지 못할시, channel이 예비주소에 연결을 시도하게 할수 있으며, channel 객체를 생성할때, backup 표를 줄 수 있다. 여기서 한개의 숫자그룹, 각각의 항은 모두 주소이다.

이것은 mongo 의 클러스터 모드의 설계를 위한것이다, 그러니 행위와 mongo 의 클러스터 연결을 동일하다.

它是为 mongo 的集群模式设计的,所以行为和 mongo 的集群连接一致.

# 오버로드 통지

만약 짧은 시간내에 channel 을 향해 대량의 푸쉬 요청을 보내면, low층은 즉시 발송을 하지못할수 있다. 메모리에 쌓여질수 있다. 이를 오버로드라 부른다. 오버로드가 발생했을때, 상층로직은 뭔가 조취를 취해야 할것이다. 예를들어, 잠시 요청을 C 로 보내지 않는다던지; 아니면 합친후 계속해서 요청, 새로운 요청 저지 등등.


channel 객체를 생성할때 function overload(isOverload) 를 넘겨주어 overload 발생이나 다시 완화될때 조정하도록 할수 있다.

skynet 은 redis, mongo, mysql 등 데이터베이스 기동은 모두 과부하 콜백을 지원한다.

关于 socket channel 的具体用法除了阅读 lualib/socketchannel.lua （同时这也是理解 socket 模块的好材料）的实现外,也可以阅读 lualib/redis.lua 和 lualib/mongo.lua 这两个为 skynet 编写的数据库 driver .

socket channel 의 구체사용법은 lualib/socketchannel.lua (동시에 socket 모듈이해에 도움이됨) 의 구현외에, lualib/redis.lua 와 lualib/mongo.lua 이 두개의 skynet 으로 쓰여진 데이터베이스 driver 를 참고하자


출처: <https://github.com/cloudwu/skynet/wiki/SocketChannel> 


