---
layout: post
title: UniqueService
date: 2020-01-02 19:20:23 +0900
category: Skynet
lastmod: 2022-03-17T14:26:54.371Z
---

skynet.newservice 를 통해 lua 로 쓰여진 서비스를 기동할 수 있다. 하나의 스크립트를 여러개 기동시킬 수 있으며 모두 서로다른 주소를 가진다. 주소는 서로 다른서비스의 유일한 식별자이다.


하지만 가끔, 전체 시스템에서 한종류의 일을 해결하는데 한개의 서비스만 필요하며, 시스템 기동시, 기동이 완료된후, 다른 서비스들은 그 서비스의 주소를 알아야 할 필요가 있을때가 있다.
이럴때, skynet.uniqueueservice 는 좋은 선택이다.


skynet.uniqueservice 와 skynet.newservice 의 입력 파라메터는 동일하며, 모두 스크립트이름으로 lua 스크립트를 찾아 기동한며, 응답값은 이서비스의 주소이다. 하지만 newservice 의 다른점은, 각각의 이름의 스크립트가 하나의 skynet 노드에서 한번만 실행된다. 만약 이미 같은 이름의 서비스가 기동되었거나 기동중이라면, 나중에 호출된 반환값은 첫번째 기동한 서비스의 주소이다
> 역자: 테스트해볼까
 

UniqueService 는  네임서비스(이젠 추천하지 않는 초기의 특성) 의 기능을 대체한다. 많은 skynet 라이브러는 모두 독립적인 서비스를 가지고 있으며, 라이브러리 초기화 할때, 비슷한 코드를 사용할 수 있다.



```lua
local SERVICE
skynet.init(function()
  SERVICE = skynet.uniqueservice "foobar"
end)
```


이 예제는 초기화 함수를 등록하여 SERVICE 변수를 초기화한다. 라이브러리에서는 SERVICE 주소를 통하여 유일한 foobar 서비스에 접근할 수 있다.


uniqueservice는 지연 초기화전략을 채용하고 있다. 전체 시스템에서 첫번째 호출될때, 서비스는 초기화를 시작한다. 어떤 상황에서는, lazy 초기화를 원하지 않을수 있으며, skynet 기동 스크립트에서 명시적으로 필수적인 서비스를 초기화 할수 있다 (초기화 과정은 비교적 느리고 길수 있다, 지연 초기화는 불필요한 시스템 지연을 피할수 있다).그러면, 어떻게 어떤 서비스가 이미기동 되었는지 알수 있을까, skynet.queryservice 를 통해 이미 존재 하는 서비스를 검색 할수 있따. 만약 이 서비스가 존재하지 않으면, 이 API 는 블럭되어 이서비스가 기동될까지 기다린다.


기본상황에서, uniqueservice 는 크로스 노드하지 않다. 다시말해서, 서로 다른 노드에서 uniqueuservice 는 같은이름의 서비스가 될수 있으며, 서비스 또한 독립적으로 기동된다. 만약 전체 네트워크에서 유일한 서비스가 필요다면,  uniqueueservice 의 파라메터의 파라메터 앞에 true 를 추가하여, 전역서비스임을 명시한다.



대응하는, 서비스를 검색하는 queryservice 또한 첫번째 파라메터가 true 인 상황을 지원한다. 이런 전역서비스, queryservice 는 매우 요용하다. 늘 전역서비스가 어느노드에 위치해있는지 명확히 알 필요가 있으며, 이는 합리적인 구조이다. 설계한 노드에 기동스크립트에서 skynet.uniqueservice(true, "foobar")로 서비스를 기동후, 다른서비스에서 이서비스의 주소를 skynet.queryservice(true, "foobar") 를 호출하여 사용한다.



DataCenter 와 다르게, uniqueservice 는 서비스관리전용 모듈이다. 서비스주소관리에 특별한 최적화를 하였다. 서로다른이름에대해, 한개의 기동만을 허용하고, 대체를 허용하지 않는다. 구현에 있어서, 모든 노드에서 검색했던 결과를 캐쉬하여, 매검색마다 중심노드의 검색을 통하지 않도록한다.



출처: <https://kybird.github.io/skynet/2020/01/02/UniqueService.html> 
