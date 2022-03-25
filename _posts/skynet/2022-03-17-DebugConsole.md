---
layout: post
title: DebugConsole
date: 2022-03-17T13:53:54.026Z
category: Skynet
lastmod: 2022-03-25T13:59:55.610Z
---

스카이넷은 내장 디버그콘솔을 가지고 잇다. 기동 스크립트에서 기동해야한다

```lua
skynet.newservice("debug_console",8000)
```

이 예제에서는,  8000포트를 리슨하고, 이것은 설정가능하다.

보안을 위해, 디버그콘솔은 오직 127.0.0.1 로컬주소만을 리슨한다. 리모트에서 사용하려면 해당머신에 로그인한후 연결한다

Telnet 이나 nc 를 사용하여 로그인할수 있다. 시작후 다음이 보여진다

```
Welcome to skynet console
```

이게보여지면 연결이 성공한것이다.

주: 스카이넷은 자신만의 IO 라이브러리를 사용하기 때문에, libreadline 을 통합하기가 어렵다. (readline 의 hook 함수에서 yield를 사용할 수 없다). 만약 debug console 에서 readline 의 히스토리를 사용하기를 원한다면 rlwrap 을 사용해보자.

이제, 디버그 명령을 입력할 수 있으며, help 를 입력하여 모든 지원하는 명령을 리스트할 수 있다. 이들 명령들은 계속 변경될 수 있으니 help명령으로 확인한다

명령의 일반적 포맷은 command address 이다. 어떤명령들은 주소가 필요치않다. 주소를 입력할때, :01000001 과 같은 포맷은 서비스의 주소를 의미한다. :로 시작하는 8자리의 16진수이다, 또한 첫번째 두개의 숫자인 harborid 와 연속된 0 은 생략될수 있다, 예를들어 :01000001 은 간단하게 1로 쓸수 있다. 모든 활성 서비스는 list 입력을 통해 출력 할 수 있다.

curl 을 사용하여 http GET 요청으로 명령을 실행할수 있다.
```
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
```

루아 서비스를 위한 자주 사용되는 명령어들은 다음과 같다:  
* List 모든 서비스, 기동서비스의 명령 파라메터 목록 출력
* Gc 강제로 모든 루아서비스가 가버지 컬렉트 하도록 함. 동시에 갱신된 메모리사용량을 보고함
* mem 모든 루아서비스들이 자신의 메모리 사용량을 레포트하도록함 (주: 오직 루아서비스의 루아 VM 메모리상황, 만약 C 모듈의 메모리 사용 보고가 필요하면, MemoryHook 을 참고)
* stat 모든 루아서비스의 메시지큐의길이, 지연된 요청 수량, 처리된요청 수량, 만약 Config 에 profile 이 true 일경우, CPU 사용량도 보고한다.
* service 모든 단일 루아서비스를 리스트하고 존재하지 않은서비스에게 전송되어 지연된 요청을 보여준다.
* netstat 네트워크 상태를 리스트한다
주의, 이 모든 명령들은 모든서비스에 전송되어 응답을 기다린다, 어떤 lua 서비스가 오버로드 되었을시, 응답을 받을때까지 오랜 시간이 걸릴수 있다.

단일루아서비스의 명령은 다음과 같다.:
* start service_name 은 skynet_newservice 를 사용하여 새로운 lua service 를 시작한다
 
* snax service_name 은 snax.newservice 를 이용하여 새로운 snax service 를 시작한다
  
* exit addresss 루아서비스를 종료한다
  
* kill address 루아서비스 종료를 강제한다
  
* info address lua 서비스가 자신의 내부정보를 보고하도록한다. 참고 [Profile](2022-03-17-Profile.md)
  
* Signal address sig 서비스에 메시지를 전송한다, sig의 기본값은 0이다. 서비스가 무한루프에 진입 했을때, 기본 시그널은 루아의 바이너리코드를 종료한다. 그리고 콜스택과 함께 에러를 발생한다. 이는 endless loop 를 디버깅하는 최선의 방법이다 주: 여기의 시그널은  시스템의 시그널과 같은것은 아니다.  

* Task address 서비스중 모든 정지된 요청의 콜스택을 표시한다.  

* debug address  명시한 lua 서비스에 내장된 single 디버거를 기동한다. http://blog.codingnow.com/2015/02/skynet_debugger.html

* Logon/logoff address 서비스의 모든 입출력 메시지를 파일로 기록한다. Config 에 logpath 항목을 설정해야 한다.

* Inject address script script 이름에 해당하는 스크립트를 지정한 서비스에 삽입하여실행한다 (핫패치시에 사용된다)

* Call address 서비스의 루아 인터페이스를 호출한다. 포맷은 call Address "foo", arg1,… 인터페이스 이름과 문자열형식은 따옴표를 사용해야하며 , 로 나누어져야한다 주소는 서비스이름을 지원한다
	
	
출처: <https://github.com/cloudwu/skynet/wiki/DebugConsole> 