---
layout: post
title: GettingStarted
date: 2020-01-02 19:20:23 +0900
category: Skynet
lastmod: 2022-03-17T14:14:53.106Z
---


# GettingStarted

## skynet 입문
skynet은 온라인게임서버를 위해 디자인된 경량 프레임워크 이다. 어쨌든, 게임전용은 아니다. 어떰 영역에도 사용할 수 있다.
skynet은 즉시 사용할 수 있는 엔진이 아니다. 사용하기전에 skynet 프레임워크의 기본아이디어를 이해 해야 하고 이 이해는 또한 개발자가 이 프레임워크가 해결할 수 있는 것이 무엇인지 알 수 있게 도와준다. 만약 이 프레임워크를 온라인게임서버에 사용하기를 원한다면, skynet 위키가 게임서버를 어떻게 만드는지 가르쳐주지 않는다는 것을 알게 될 것이다. 도구의 세트에 더욱 가까우며 그리고 오직 필요한 게 무엇인지 알았을 때 skynet 으로 당신이 원하는 목표를 성취할 수 있을 것이다.
skynet을 이해하는 것은 복잡하지 않다.  이문서를 읽는 것을 통해 skynet 을장악하기를 희망한다. 이 문서는 어떠한 API 의 구체적인 사용방법에 대해 언급하지 않는다. 또한 skynet 개발환경을 설정 하는 것도 언급 하지 않는다. 또한 어떠한 간단한 게임서버를 만드는지에 대한 한 줄 한 줄 설명하지 않는다. 그저 skynet의기본개념에 대한 소개 일 뿐이다. 그러므로 정말로 skynet 으로 개발할 시 wiki 상 다른 문서들도 참고 해야 한다
## Framework
서버 역할이라 함은 일반적으로 여러 종류의 서비스를 동시에 처리하는 것이 필요하다. 예를 들어 인터넷 게임 중.  동시 수천유저에 서비스를 제공 해야 하고 동시에 백개 이상의 인스턴스를 운영해야 하고 인스턴스 던전내의 전투를 계산하고, NPC 가 AI를통해 동작하게 하고 등등. 싱글 코어 시대에서 이러한 서비스를 CPU 의 시분할서비스로 처리했다. 유저에 병렬처리 되고 있는 느낌을 주기위해. 현대 커퓨터에서는 십 수개이상의 CPU 코어를 설치하는 것이 가능하다. skynet의 목표는 모든 코어들을  효율적으로 모두 사용하여 수천개의 상호 독립적인 비즈니스 로직을 처리하는데 맞춰져 있다.

간단한 웹 앱에서 유저상태나 데이터를 데이터베이스에 넣는것은 일반적이다. 클라이언트의 요청을 받은 후 서버 어플은 유저의 데이터를 로드하고 처리하고 데이터베이스에 써넣는다.  온라인 게임 어플에서는 데이터와 서비스간에 좀 더많은 컨텍스트와 상호작용이 있을것이다. 만약 이두타입의 어플리케이션에 같은 모델을 사용한다면 데이터베이스와 기능모듈에서 보틀넥이발생하는 것은 매우쉬울것이다.  이럴경우 메모리캐쉬를 추가한다하더라도 보틀넥을 해결할수없을것이다 

스카이넷에서, 서비스라는 개념으로 어떤종류의 구체적 로직을 표현한다. 서비스는 비즈니스 로직의  처리 그리고 관련된 데이터상태를 포함한다. Skynet 을 사용하여 서버를 구현할때 비지니스상태를 데이터베이스에 동기화 하는 것은 추천하지 않는다.  서비스의 내부메모리에 저장하는것을 추천한다서비스, 어플리에키연 로직코드그리고 게임데이터는 로컬메모리에 보존된다. 만약 데이터베이스가 이미 당신의 게임 서버프레임워크의 일부분이라면 대부분의 경우 데이터베이스의 역할은 데이터를 백업하는 역할을 할것이다.  상태가변할때 데이터를 데이터베이스에 푸쉬하여 저장하거나 일정시간간격으로 데이터를 백업할수도있다. 비지니스처리시 서버내의 내부데이터를 직접 사용한다

Skynet 서비스는 독립적인 프로세스이기 때문에, 서로 다른 서비스들 간의 통신이 매우 효율적이다. 그러한한편, 이들 서비스가 하나의 같은 프로세스내에서 같이 동작하고 있기때문에 서비스들의 생사가 함께하게된다.  어플리케이션을 개발할때  서비스들의 협동이 필요할때 우리는 다른서비스가 살아있는지 아니면 연결이 안정적인지 신경쓸필요가 없다. 대부분의 skynet 서비스들은 lua 로 개발된다. Lua 의 가상머신은 각각의서비스를 고립시키는것을 도와준다. 하지만 skynet의 초기디자인에서 서비스를 어떻게 개발 할지에 대한 제한은 없다. 이론상 skynet 서비스를 Lua 대신 다른언어로 개발할수있다. 하지만 만약 skynet 이 처음이라면 이런 상세사항은 무시하고 단순히 lua 를 골라라

간단히 말해서, skynet을 하나의 단순한 OS라고 생각할수있다. Skynet 은 수천개의 Lua VM 을 관리할 수 있고 이것들이 동시에 작업을 수행하게 할 수 있다. 각각의 lua VM 은 모두 다른VM 에서 발송 되어온 메시지를 처리할 수 있다.  그리고 다른VM 에 대해 메시지를 발송할 수 있다. 각각의 LuaVM 을 skynet 이라는 OS 에서 실행되는 독립적인 프로세스라고 생각할수있다. Skynet 이 동작시 새로운 프로세스를 기동할 수 있고 사용하지 않는 프로세스를 소각할 수 있다. 또한 디버그 콘솔을 통해 프로세스들을 감시할 수 있다. Skynet 은 외부네트워크 데이터의 입력과 타이머들을 관리하고 이것들을 한 개의 통일된 (프로세스간 메시지) 이들 프로세스들의 입력으로 메시지로 변환한다.

예를들어:

네트워크게임에서, 각온라인유저에 한 개의 lua vm 을생성할수있다. ( lua서비스라칭한다), 일단 이것을 agent 라고 부르자. 유저가 다른유저와 상호작용없이 혼자 놀고있다면, agent 는 완전히 요구사항을 만족한다. 유저가 온라인일때 Agent 는 데이터베이스에서 유저의 모든 데이터를 lua VM으로 로드한다. 유저의 네트워크요청에 응답한다. 물론 한 개의 lua 서비스가 다수의 온라인유저를 관리 하게할 수있고 각유저는 Lua VM 의 한 개의 오브젝트가 된다.
인스턴스 던전(전투), 유저들의 반응, AI와 다수의 유저들간의 전투등을 처리하기위해 한 개의 서비스를 사용할 수 도 있다. Agent 는 메시지들로 인스턴스 던전과 통신할 수 있고 client 는 instance 던전과 직접 통신할 필요가 없다.
이러한 내용은 모두 게임서버 구조의 구체적인 설계이다, skynet 은 어떻게 만들어야 하는지 아무런 요구사항이 없다. 기본적으로 당신이 어떻게 만들 것 인지 의견을 주지 않는다. 모든 것은 당신이 설계시에 결정해야한다.
## 네트워크
네트워크서버 framework로서,  필연적으로 Wrapping 된 네트워크층이 있게 된다. Skynet 에서는 더욱더 그러하다. Skynet 은 간단한OS를 시뮬레이션하기 때문에 skynet 에게 제일 중요한 작업은 수천개의 서비스를 관리하는 것이다. 서비스가 종료할 때 최대한 시스템에 영향을 미치지 않도록 하는 것이 해결 해야 할 첫번째 문제 였다. Skynet을 사용할 때 직접 시스템네트워크모듈과 통신하는 API 나 모듈을 사용하지 않을 것을 권장한다. 일단 이런 모듈에 네트워크 IO가 블록 되면 해당서비스 뿐 아니라 skynet 내부의 worker thread 에게까지  그 영향이 미친다. Skynet 은 설정을 통해 worker thread 의 개수를 정할 수 있다. Worker thread 의 수는 통상 시스템의 물리적인 코어 수량과 관계 가 있다. 게다가 skynet 이 관리하는 서비스의 수량은 동적이고 worker thread 의 개수보다 훨씬 더 많다. Skynet 의 네트워크층은 coordinator? 로 동작할 것이다. Skynet 네트워크 API 를 사용하는것이 네트워크 blocking IO 가있을시 cpu 처리량의 최고의 효율을 뽑을수 있다.

Skynet 은 TCP 연결의 listen, TCP연결의 생성, UDP 의 패킷의 송수신기능을 제공한다. 단 한줄의 코드로 한 개의 포트를 listen 할수 있고 외부 tcp 연결을 받을 수 있다. 새로운 연결이 생성 되었을 때 callback 함수를 통해서 연결의 핸들을 얻을 수 있다. 그후 이 handle 을 skynet에 다른 서비스에게 넘겨서 처리할 수 있다. 이렇게 하여 병렬처리 능력을 얻을 수 있다. 이것은 기존의 posix 시스템이 새로운 연결을 받은 후 child 프로세스를 fork 되어 이 핸들을 처리하는 방식과 같은 패턴이다. 다른점은 skynet 의 서비스는 부자 관계가 없다

Port listen 과 접속을 처리를 위해 gateway 서비스를 사용을 추천한다. 일단 유저의 신분이 확인되면 유저의 데이터를 명시한 서비스로 패스하여 처리 하게 한다. 동시에 gateway 는 미리 정의된 데이터 protocol 에 의거하여 데이터를 작은 패키지로 나누는 책임을 진다. 서비스는 데이터 unpacking 에 대해 처리할 필요가 없다. 어플리케이션 서비스는 소켓을 직접 처리할 필요가 없다. 내부메시지를 위한 skynet 인터페이스를 사용한다. Skynet code base 에서 gateway 서버를 찾을 수 있다. 어쨌든 이것은 선택사항이다. 당신의 어플리케이션의 요구사항에 맞게 당신 자신의 것을 만들 수 있다.

그 외에, skynet의 websocket 지원은 현재 실험단계이다, 필요하다면 websocket branch에서 볼 수 있다.
## Client

Skynet 은 클라이언트가 어떻게 구현되는지 전혀 상관하지 않는다. Skynet 기반의 서버는 브라우져가 클라이언트 역할을 할 수 있다 (http나 websocket기반의 프로토콜 통신), 또한 C/C++/Flash/Unity3d 등등 직접 만든 클라이언트도 가능하다. Tcp소켓의 연결지향을 선택할 수도 있고, http프로토콜을 사용하여 비지향연결을 선택할 수 있다. 아니면 UDP기반 통신을 사용할 수도 있다. 모두 자신이 선택한다. Skynet 은 상관된 모듈을 제공하지 않는다 모두 스스로 구현 해야 한다.

Skynet 메인branch에 C + Lua 를 사용하여 구현한 간단한 클라이언트 데모가 있다.  Tcp 연결 지향기반이며 기본 프로토콜은 2바이트의 big-endian 으로 패킷의 크기를 정의한다, skynet gateway 서비스는 데이터를 이 길이에 기반하여 비즈니스 로직패킷으로 절단한다. 분리하여 내부서비스처리에 전달한다. 만약 skynet 내부의 gateway 모듈을 사용하고 싶다면, 이 기본적인 패킷 분리규칙만 지키면 된다.

각각 비즈니스 패킷의 인코딩 규약에 대해, demo 는 sproto 라고 불리는 사용자정의 프로토콜을 사용한다. 이것은 skynet 메인 브랜치가 포함하고 있다. Demo 는 sproto 가 어떻게 데이터를 인코딩하고 디코딩하는지 보여준다. 하지만 sproto 프로토콜 사용여부는 skynet 은 아무런 제약을 하지 않는다. Json 이나 google protocol buffer 등을 사용할 수 있다. 오직 해당하는 프로토콜 디코딩모듈을 lua 에 통합 하는법을 알기만 하면 된다.  Gateway 에서 아니면 한 개의 독립적인 서비스를 사용하여서 네트워크 메시지를 decode 하여 skynet 의 내부메시지로 변환하여 대응하는 서비스로 보내기를 추천한다. 내부 서비스는 네트워크층이 이러한 메시지를 어떻게 전송하는지 관심을 가질 필요가 없다.

## Service
Skynet 의 서비스는 lua 5.3(영문은 5.4로명시)으로 작성된다. 지정한 규칙에 맞추어 skynet 이 찾을 수 있는 디렉토리에 넣으면 다른 서비스들에 의해 기동 된다. Skynet 설정파일에 서비스 검색 경로를 설정한다. 그리고 기동할 첫번째 서비스를 설정한다. 다른 서비스들은 모두 이 서비스가 직간접적으로 기동한다 각 서비스는 스카이 넷 framework 가 분배한 유일한 32bit id 를 가진다, skynet 은 이 id 를 서비스의 주소로  사용한다. 서비스가 퇴출하여도 이 주소는 오랜 시간 동안 보류된다. 새로운 서비스가 이전의  서비스의 주소를 재사용하여 잘못된 메시지전송을 방지하기 위해서

각서비스는 3개의 스테이지들 가진다
첫번째는 서비스로딩단계이다, 서비스의 원본 파일이 로딩 될 때 lua의 운행규칙에 따라 실행된다. 이단계는 서비스를 block 가능성이 있는 어떠한 skynet api 도 호출할 수 없다. 왜냐하면 이단계에서 이 서비스와 관련된 skynet 설정이 초기화 되지 않았기 때문이다.
다음은 서비스초기화 단계이다, API skynet.start 함수로 등록한 초기화함수를 실행한다. 이 초기화 함수는 이론상 어떠한 skynet api 든 모두 호출할 수 있다. 단 서비스를 기동하는 skynet.newservice 이 api 는 모든 초기화함수가 완료 되었을 때 return 한다   
마지막단계는 서비스의 동작 단계이다. 초기화단계에서 메시지 dispatcher 를 등록했다면 메시지가 들어오기만 하면 등록된 메시지처리 함수가 trigger 된다. 이러한 메시지들은 모두 skynet 내부 메시지이다. 외부의 네트워크데이터, 타이머 또한 내부 메시지형식으로 동작한다.
Skynet framework 관점에서, 각각의 서비스는 한 개의 메시지 처리기이다. 하지만 어플 측면 에서는  그렇지 않다. 서비스는 lua 의 coroutine 을 사용하여 동작한다. Service 가 요청을(세션정보를 포함한 메시지) 다른 서비스에게 보낼 때, 이것은 메시지가 처리되었음을 의미한다. Skynet 은 서비스(보낸이)를 정지한다.  응답하는 서비스(받는이)는 메시지를 받고 응답(응답메시지타입으로) 하고 서비스(보낸이)는  정지된 coroutine을 찾아내고 메시지를 배달하고 남은 흐름을 완료한다. 유저의 관점에서 task 를 처리하는 single thread 처럼 보인다. 각각의 task들은 자신의 context 를 가진다. 이것은 nodejs 나 다른 framework 에서 사용되는 callback 과는 같은 전력이 아니다. 
Erlang 과는 다르게, skynet 은 응답이 없는 메시지로 요청이 정지되었어도 여전히 다른 메시지를 처리할 수 있다. 그래서 같은 skynet 서비스에서 여러 개의 타스크들을 실행할 수 있다. 이것들은 병렬처리처럼 보인다. 진정한 분할과 서로 다른 서비스에서 처리의 차이점은. 이처리의 과정은 영원히 진정한 병렬처리가 될 수 없음이다. 그저 교대로 동작할 뿐이다. 한 개의 task 는 다음 IO 블록 지점 까지 계속 수행된다. 그리고 다음 로직으로 교체 된다. 이것을 사용해서 다수의 타스크 들을 처리할때 공유된 데이터를 사용하게 할 수 있다.이러한 데이터는 하나의 luaVM 하에 있다면 읽고 쓰기는 모두 메시지를 사용보다 매우 값싸다
장단점으로는, 일단 현재 task처리 스레드가 중지 되면 응답을 기다린 후 처리를 진행할때, 내부상태는 다른 task 에 의해 변경되었을 가능성이 매우 높다는 것이다. 그러니 이것에 대해 매우 조심 해야한다. Skynet api 문서에 이미 block 가능한 API 들에 대해 명시했다. 두개의 Block API 호출은 Atomic하다.  이 특성을 사용하여 전통적인 multi thread 개발보다 훨씬 쉽게 개발할 수 있다.
같은 서비스내에서  다수의 유저의 스레드가 존재할 수 있다. 이러한 스레드는 skynet.fork 함수 호출을 통해 기동 된다. 또한 skynet의 timer 의 callback 함수를 통해 기동할 수 있다. 위에서 말한 메시지처리 함수역시 한 개의 독립된 user thread 이다. (어떤 요청에대한 새로운 한 개의 user thread 를 기동 으로 이해) . 이러한 점은 os 시스템의 thread 의 다수의 core 가 스레드의 병렬운행을 하는것과 다르다. 한 개의 서비스내에 서로 다른 user thread 는 영원히 순서대로 실행되는 권리를 가진다. 각각의 thread 는 제어권을 양보하기 위해 하나의 block 동작을 필요로 하고 다른 thread 가 제어권을 양보할 시 실행을 계속한다.
만약 user thread 가 영원히 block api 를 호출하지 않아 제어권을 양보하지 않을 경우, 이 스레드는 cpu 의 worker thread 를 영원히 차지한다. Skynet 은 선점 scheduler가 아니다. 시간 관련된 설계가 없다. 하나의 task 동작시간이 매우 길어진다고 강제로 그것을 정지시키지 않는다. 그래서 개발자가 무한루프를 만들지 않도록 스스로 조심해야한다.  그럼에도 skynet framework 는 task 모니터를 가지고 있다. 어떤 서비스내에 어떤 task 가 너무 긴 시간을 사용한다면 log 형식으로 보고한다. 개발자에게 무한루프 버그를 알려준다. Lua 코드에서 무한루프 bug (lua가 호출한 C 모듈에 의한 무한루프는 아니다) 가 있다면  framework 로 강제로 중단할 수 있다. 구체적인 방법은 개발 중 마주치게 된다면 이해 할 수 있을 것이다.
## Message
각각의 skynet 메시지는 6부분으로 이루어져있다: 메시지종류, session, 발송서비스주소, 수신서비스주소,  메시지의 C 포인터, 메시지의 길이

각 skynet 서비스는 다른 종류 의 메시지들을 처리할수 있다. Skynet 에서 type 을 통해 메시지를 분류한다. 하지만 이 type 은 메시지의 타입보다는 네트워크의 포트 와같은 개념이다. 각 서비스는 255개의 포트를 지원한다. 메시지 분류발송 함수는 서로 다른 port 에 따라 서로 다른 메시지 처리 프로세스를 설정한다.

Skynet 은 미리 설정한 한조세트의 메시지 타입들이 있다.  개발자가 관심을 가져아할것 으로: Response Message, Network Message, Debug Message, Text Message, Error Message 가 있다.

Response 메시지는 통상 특별한 처리가 필요하지 않다. Skynet 기본라이브러리로 관리되고 서버내의 coroutine 을 관리하는데 사용된다. 외부로 메시지를 발송한후, 상대방은 하나의 응답 메시지를 보내올수 있다. 이 메시지의 타입이 바로 Response 메시지 이다. 발송요청측은 Response 메시지를 받고 이 메시지의 session 을 통해 이전에 요청을 보낸 모든 coroutine을 찾아내고 그것을 이어간다
Network 메시지 또한 직접처리 할 필요가 없다. Skynet 은 이 타입의 메시지를 관리하는 socket 라이브러리를 제공한다. 라이브러리는 socket api 를 편리하게 사용할 수 있도록 수정되어 있다.
Debug 메시지는 skynet 라이브러리내에 default handler 를 가지고 있다. 모든 서비스에 대해 기본적인 기능을 제공한다.  현재 LuaVM 의 메모리사용량에 대한 메트릭,  정지된 타스크의 수, 디버깅목적으로 lua script 를 inject 할 수도 있다. Skynet 은 각각의 LuaVM 에 대해 중앙화된 제어를 가지고 있지 않다.  Debugging 컨트롤러는 오직 debugging 메시지들을 관련된 서비스에 전송할 뿐이다. 서비스는 대응할 내용을 직접 작성해야한다

진정한 비지니스 로직은 Text Message 와 Lua Message 들로 동작된다. 차이점은 메시지 인코딩의 차이 이다. Text Message 는 저수준의 C module 에서 사용하기 좋다. 간단한 text 스트링이다. 하지만 lua Message 는 보다 복잡한 데이터 구조로 직렬화 될수 있으며 대부분의 경우에 우리는 오직 Lua Message 만 사용한다.

接管某类消息需要在服务的初始化过程中注册该消息的序列化及反序列化函数，以及消息回调函数。lua 类的序列化函数已经由 skynet 基础库默认注册，它们会把框架传入的消息 C 指针及长度信息转换为一组 Lua 数据。编写业务的开发者只需要注册消息回调函数即可。这个回调函数会接收到别的服务发过来的一系列 Lua 值，以及发送服务的地址和该请求的 session 号（一个 31bit 正整数）。一般我们不必关心地址和 session ，因为 skynet.ret 和 skynet.response 这两个 api 可以帮助你正确的将回应消息发还给请求者。另外，skynet 还约定，如果一个请求不需要回应（单向推送），就置 session 为 0 。

어떤 타입의 메시지를 접수하려면  서비스의 초기화 과정에 메시지의 타입에 직렬화/역 직렬화 함수를 등록해야 한다. Lua 형의 직렬화 함수는 이미 skynet 기본라이브러리에 등록 되어있다. 이 함수는 C Pointer 와 메시지길이를 lua 데이터로 변환한다. 어플리케이션을 개발하기위해 callback 함수를 추가하면 된다. 이 callback 함수는 다른 서비스에서 lua data 를 받을 것 이고 발송한 서비스의 주소와 요청의 session 번호(31비트 부호없는 정수) 를 받는다. 일반적으로 우리는 이 주소와 세션을 신경 쓸 필요가 없다. Skynet.ret 과 skynet.response 이 두개의 API 가 요청자에게 응답을 보내도록 도와주기 떄문이다. 게다가 skynet은 약속되어있다 만약 응답이 필요하지 않은 요청이 있다면 (단방향 PUSH) 세션은 0으로 설정한다.

어플리케이션레벨에서 skynet 은 Error Message 에 대한 룰을 가지고 있다. 필요하지 않으면 개발자는 처리할 필요가 없다. 이종류의 메시지는 일반적으로 실제내용이 없다. 발송원의 주소와 세션이 있을 뿐이다. 이것들은 어떤 요청에서 이상이 발생했는지 표현한다. 아니면 요청을 완성할 수 없다면 어떤 서비스가 이미 종료되었는지 표현한다.  이종류의 오류 메시지는 skynet 기본 라이브러리에서 lua 층의 에러로 변환되어 호출자에게 던져진다. RPC 호출의 오류라고 이해할 수 있다.
## External Service
하드웨어 성능을 최대한 이끌어내기 위해 될 수 있는 한 같은 skynet 프로세스내의 모든 어플리케이션 로직을 종료해야 한다. 하지만 가끔 외부 프로세스를 사용하지 않을 수 없다. 예를 들어 다른 내부서비스를 위한 SQLite Service 를 가질수 있다. 하지만 또한 데이터서비스들을 위한 MySQL, Redis, MongoDB와 같은 비독립적인 외부서비스를 사용하길 원할 수도 있다.

skynet 发布版中提供了 mysql redis mongo 的驱动模块，省去了开发者自行封装的烦恼。这些驱动模块都是基于 skynet 的 socket API 实现的，可以很好的协同工作。如果你希望使用别的外部数据库，则需要自行封装。需要注意的是，大多数外部数据库的默认驱动模块都内含了网络部分，它们直接使用了系统的 socket api ，和 skynet 的网络层有一定的性能冲突。一个比较简单的兼容方案是额外再自定义一个中间进程，一边使用外部数据库的默认驱动模块，另一边用 skynet 提供的 socket channel 和 skynet 交互。

Skynet 메인 브랜치는 mysql, redis, mongo 를 위한 확장 모듈을 제공한다. 이들을 위한 가상 레이어를 직접 작성할 필요가 없다. 이들 확장모듈은 모두 skynet의 socket API 로 구현되어 있어서 매우 잘 동작한다. 만약 다른 외부의 라이브러리를 사용하기 원한다면 스스로 wraping 할 필요가 있다. 주의할점으로는 대부분의 외부 라이브러리의 기본 확장모듈은 모두 네트워크 부분을 포함하고 있고 이들은 시스템의 socket API 를 사용하고 이것은 skynet 네트워크층과 충돌을 할수 있다. 비교적 간단한 해결방법으로 소켓채널을 사용하는 미들웨어를 하나 작성하여 외부 데이터베이스 소켓과 skynet socket을 브릿지하는것이다. (서비스하나를 추가해서 한쪽은 외부모듈사용 한쪽은 스카이넷모듈사용 이라는 뜻인가)

Skynet main branch provides an extension module for MySQL, Redis, MongoDB so you don't need to write your own abstraction layer. This extension module is implemented with Skynet socket API and it works pretty well. If you want to use other external databases, you'll need to develop your own extensions. Please keep in mind that, most external databases have their own network module, they use system socket API which can be conflicted with the Skynet network layer. A compatible solution is to add a middle process using socket channel as the bridge of external database socket and Skynet socket.

你还可能需要让 skynet 提供 http 协议的 web 服务，或是使用 http 协议和外部 web 服务对接。skynet 自带了 http 模块，实现一个简单的 http 服务器不会比用其它框架开发更复杂。即使用 skynet 做一个 web 服务器也可以轻松获得高性能。但是，出于简化编译依赖的想法，skynet 的默认编译脚本并没有将 openssl 链接进去，而 https 支持需要它。如果需要支持 https ，需要额外设置 TLS_MODULE=ltls。另外，你也可以用 nginx 制作一个 https 反向代理服务器，而不必直接使用 skynet 的 https 模块。
원할수있다 스카이넷이 http 프로토콜의 웹서비스를 제공하기를, 아니면 http 프로토콜과 외부 web 서버와 통신. Skynet 은 http 모듈을 가지고있다, 간단한 http 구현은 다른프레임워크의 그것보다 복잡하지 않다. Skynet 을 사용하여 web 서버를 만드는것은 매우 간단하고 고성능을 얻을수 있다. 하지만  컴파일과정을 간단하게 하기위해 skynet 은 기본 컴파일 스크립트는 openssl 을  초함하지않는다. Https 를 지원하려면 그것이 필요하다. 만약 https 를 지원해야한다면 TLS_MODULE=ltls 를 설정해야한다. 아니면 nginx 를 사용하여 reverse proxy 를 설정할수 있다. 이러면 skynet 의 https 모듈을 사용할 필요가 없다.
You may also want to use Skynet as an Http Web Server, or use HTTP protocol to talk with external Service. Skynet includes an Http module, it's not more work to develop an HTTP Server than other network frameworks. It can also be used as a high-performance web server. However, to simplify the compiling process, Skynet does not include OpenSSL by default in the build script, which is required by HTTPS. If you want to support HTTPS, you need to add TLS_MODULE=ltls. In addition, you can use Nginx as a reverse proxy server instead of using HTTPS in Skynet.

不建议把连接管理的网关实现成一个外部服务，因为 skynet 在管理大量 TCP 连接这方面已经做的很好了。放在 skynet 内部做可以减少大量不必要的进程间数据传输。
추천하지 않는다. 연결을 관리하는  Gateway 서비스를  외부서비스로 사용하는것을,. 왜냐하면 skynet 은 대량의 tcp 연결 이부분에 이미 매우 잘만들어져있다. Skynet 내부에 넣으므로서  프로세스들간의 대량 데이터 교환을 줄여주수 있다.
It's not recommended to use gateway service, which manages connections, as an External Service, because Skynet is very optimized in managing TCP connections. It's better to make it a Skynet Internal Service to reduce the data transit between OS processes.

## Cluster
skynet 在最开始设计的时候，是希望把集群管理做成底层特性的。所以，每个服务的地址预留了 8bit 作为集群节点编号。最多 255 台机器可以组成一个集群，不同节点下的服务可以像同一节点进程内部那样自由的传递消息。
Skynet 은 최초에 설계시점에, 희망했다 클러스터를 저수준층의 특성으로 관리하기를. 그래서 각각의 서비스의 주소는 예약보류되었다 8비트는 클러스터 노드아이디, 최고 255개의 머신이 한 개의 클러스터를 이룰수있게 되어있다. 서로다른 노드아래의 서비스는  동일한 노드의 프로세스 내부처럼 자유롭게 메시지를 전송할수 있다.
At the beginning of Skynet design, I was hoping to make Cluster embedded infrastructure. So 8 bit in the Service address was reserved for Cluster node. A cluster can contain at most 255 machines, a message can be transmitted over Services of different nodes just like the Service within one node.

随着 skynet 的演进和实际项目的实践，发现其实把节点间的消息传播透明化，抹平节点间和节点进程内的消息传播的区别并不是一个好主意。在同一进程内，我们可以认为服务以及服务间的通讯都是可靠的，如果自身工作所处的硬件环境正常，那么对方也一定是正常的。而当服务部署在不同进程（不同机器）上时，不可能保证完全可靠。另外一些在同一进程内可以共享访问的内存（skynet 提供的共享数据模块就基于此）也变得不可共享，这些差异无法完全被开发者忽视。

In the developing of Skynet and real-live practicing, we find that it's not a good idea to hide the fact that the messaging within the same node is different than two nodes of different machines. Within the same process, we think the communications between Services are more reliable, we can assume that if my current Service is working normally so do other Services within the same physical environments. However, when it's distributed on different processes of different machines, it's not guaranteed. In addition, memory can be shared within the same process will become unsharable as well, these cannot be ignored by developers either.

所以，虽然 skynet 可以被配置为多节点模式，但不推荐使用。
So, although Skynet is configured working as multiple nodes, it's not recommended to use.


目前推荐把不同的 skynet 服务当作外部服务来对待，skynet 发布版中提供了 cluster 模块来简化开发。
For now, we recommend treating Services from different Skynet instances as external Service, in the main branch of Skynet there is a Cluster module for quick development.

출처: <https://github.com/cloudwu/skynet/wiki/GettingStarted> 
