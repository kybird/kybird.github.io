---
layout: post
title: LoginServer
date: 2020-01-02 19:20:23 +0900
category: Skynet
lastmod: 2022-03-17T14:15:07.948Z
---
Skynet 은 통용의 로그인서버 템플릿인 snax.loginserver 을 제공한다
구조
먼저아래와같이 정의한다
	• 로그인서버 L 。여기서 소개하는 LoginServer
	• 로그인지점은 G1, G2, G3 ...
	• 인증플랫폼A
	• 유저 C
C 가 G1에 로그인을 시도할때  아래와같은 과정을 진행한다
	1. C가 A를 향해 인증요청을 발송한다(A는 통상적으로 서드파티플랫폼이다), 토큰을 얻는다. 이토큰은 통상적으로 유저의이름과 유저의 합법성을 인증할수있는 기타정보를 포함하고있다.
	2. C 가로그인하려고하는 로그인지점 G1 (아니면 부하분배를통해 시스템이 정해준 다른지점과 로그인서버 L 에 스텝1에서 획득한 토큰을 양쪽에 발송한다
	3. C 와 L 은 이후의통신을위한 암호화키 secret 를 교환 ， 인증한다。
	4. L 은 로그인지점이 존재하는지 검증하고，token 의 합법성을 검증한다（여기서 L 과 A 가 한번 확인이 필요할수있다)
	5. (옵션) L이 C 가 이미 로그인했는지 체크하고 만약이미로그인했다면 C 가로그한지점 (한 개거나 여러개일수도있다)으로 메시지를 전송한다, 로그인지점의 확인을 기달린다. 통상 여기서 이미 로그인한 유저를 로그아웃처리한다
	6. L 이 G1 에 유저 C의 로그인요청을 하고 동시에 secret 를 발송한다
	7. G1 이 step 6에 요청을 받은후 C의로그인준비작업을 진행한다(통상데이터로딩등) secret 을 기록한다. 동시에 G1 은 subid 를 L 에게 분배한다。Subid 는 userid 에 대응하는 중복하지않는다。
	8. L 은 subId 를 C 에게 발송한다. subid 는 다수의 유저가 다수의 로그인(한개계정으로 동시접속을 허용），한 개의 userId 와 한 개의 subId 를 합쳐서 한 개의 로그인한유저의 이름이 된다. 더군다나 모든 username 은 모두 유일한 secret 과 바인딩된다.
	9. C는 L 의 확인을 획득후 L 과의연결을 끊고 G1에 연결한다 username 과 secret 으로 handshake 를 진행한다
이상과정에서 ,  단한개의과정만 실패하더라도, 모두 로그인과정을 중단한다. 유저는 오류코드를 받게된다
로그인지점은 비지니스로직의 요구사항에따라 유저가 로그아웃후 L 에 통지해야할수도있다. 연결이 끊어질때 （지속연결의 앱일경우）、유저가 스스로나갈때、특정시간내 유저의 메시지를 받지못할때 등 로그아웃이 발생한다
한명의유저가 로그인시, 이유저가 이미 시스템에있다면 통상 세가지방안이있다
	1. 동시로그인을 허용。매번로그인시 subId가 다르다 그래서 한 개의 계정에 여러개의실체가 있다。
	2. 동시로그인 금지，새로운 로그인인증후 이전에로그인실체를 로그아웃시킨다. 로그아웃완료후 새로운 로그인을 받아들인다。
	3. 만약 유저가 시스템상에 있다면, 다시 로그인을 금지한다.
LoginServer 당신이 어떤방안을 사용하는지 강제하지 않는다 자유롭게지정할수있다 하지만 뒤의 두가지방안을 어느정도 지지하며 이는 비지니스로직 구현의 복잡성을 간소화한다
사용
lualib/snax/loginserver.lua 는한개의 보조라이브러리다. 로그인모듈구현을도와준다
locallogin =require"snax.loginserver"
localserver ={
	host ="127.0.0.1",
	port =8001,
	multilogin =false,	
          --disallow multiloginname ="login_master",
            --config,
          etc
}
login(server)
snax.loginserver 모듈을 얻은후，설정표를 생성한다, 그리고 호출하면 즉시 로그인모듈이 기동한다
	• host 리슨주소이다, 통상 0.0.0.0 이다
	• port 는 리슨포트이다
	• name 은 내부사용용이름이다, 다른 서비스 이름과 중복하지 말라 위의 예제에서, 로그인서비스는 .login_master 이름으로 등록된다
	• multilogin  boolean 값이다. 기본값은 false. 유저가 로그인과정을 진행할때 동일한 유저이름으로 로그인진행을 금지한다. 만약 동시로그인을 허용하고싶다면 true 로설정한다. 하지만 잠재적인 병행상태관리에 따른문제는 스스로 해결해야한다
또한 관련비지니스로직처리함수를 추가해야한다
functionserver.auth_handler(token)
이함수를 구현해야한다.  클라이언트가 발송해온 token(step2) 를 인증한다. 인증이실패한다면 유저에게 error를통해 예외를던지고, 인증이 성공한다면 유저가 로그인할지점을 리턴한다（로그인할지점은 토큰내유저자신이 지정할수도있다. 아니면 부하분배를통해 정해줄수도있다 그리고 유저명도똑같다?(유저명도 포함한다?)
이함수내에 원격호출 skynet.call 은 안전하다
functionserver.login_handler(server, uid, secret)
你需要实现这个方法，处理当用户已经验证通过后，该如何通知具体的登陆点（server ）。框架会交给你用户名（uid）和已经安全交换到的通讯密钥。你需要把它们交给登陆点，并得到确认（等待登陆点准备好后）才可以返回。
如果关闭了 multilogin ，那么对于同一个 uid ，框架不会同时调用多次 login_handler 。在执行这个函数的过程中，如果用户发起了新的请求，他将直接收到拒绝的返回码。
如果打开 multilogin ，那么 login_handler 有可能并行执行。由于这个函数在实现时，通常需要调用 skynet.call 让出控制权。所以请小心维护状态。例如，你希望在这个函数中将上一个实例踢下线。那么你需要在踢人操作后再次确认用户是否真的不在线（很有可能另一个登陆的竞争者恰好在此时又登陆成功了）。
一般你还希望这个登陆服务器可以接受一些 skynet 内部控制指令，比如让登陆点可以通知玩家下线了，动态注册新的登陆点等等操作。所以你可以定义这个函数来接收 skynet 内部传递过来的 lua 协议的消息：
functionserver.command_handler(command, ...)
command 是第一个参数，通常约定为指令类型。这个函数的返回值会作为回应返回给请求方。
你可以把登陆服务器做为一个单独的 skynet 进程使用，并用 cluster 模块和其它 skynet 进程做集群间通讯；也可以启动在一个 skynet 节点中。在附带的例子 examples/login/logind.lua 中，使用的后一种形式。
你可以参考 examples/login/client.lua 来实现配套的客户端。
wire protocol
登陆服务器和客户端的交互协议基于文本。每个请求和回应包，都以换行符 \n 分割。用户名、服务器名、token 等，为了保证可以正确在文本协议中传输，全部经过了 base64 编码。所以这些业务相关的串可以包含任何字符。
下列通讯流程的协议描述中，S2C 表示这是一个服务器向客户端发送的包；C2S 表示是一个客户端向服务器发送的包。
	1. S2C : base64(8bytes random challenge) 这是一个 8 字节长的随机串，用于后序的握手验证。
	2. C2S : base64(8bytes handshake client key) 这是一个 8 字节的由客户端发送过来，用于交换 secret 的 key 。
	3. Server: Gen a 8bytes handshake server key 生成一个用户交换 secret 的 key 。
	4. S2C : base64(DH-Exchange(server key)) 利用 DH 密钥交换算法，发送交换过的 server key 。
	5. Server/Client secret := DH-Secret(client key/server key) 服务器和客户端都可以计算出同一个 8 字节的 secret 。
	6. C2S : base64(HMAC(challenge, secret)) 回应服务器第一步握手的挑战码，确认握手正常。
	7. C2S : DES(secret, base64(token)) 使用 DES 算法，以 secret 做 key 加密传输 token 串。
	8. Server : call auth_handler(token) -> server, uid (A user defined method)
	9. Server : call login_handler(server, uid, secret) -> subid (A user defined method)
	10. S2C : 200 base64(subid) 发送确认信息 200 subid ，或发送错误码。
错误码
	• 400 Bad Request . 握手失败
	• 401 Unauthorized . 自定义的 auth_handler 不认可 token
	• 403 Forbidden . 自定义的 login_handler 执行失败
	• 406 Not Acceptable . 该用户已经在登陆中。（只发生在 multilogin 关闭时）

출처: <https://github.com/cloudwu/skynet/wiki/LoginServer> 
