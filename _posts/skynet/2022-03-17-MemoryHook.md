---
layout: post
title: MemoryHook
date: 2020-01-02 19:20:23 +0900
category: Skynet
lastmod: 2022-05-23T07:09:02.318Z
---
Skynet 은 메모리관리 모듈로 jemalloc 을 기본으로 사용하고있다. 하지만꼭필요한건아니다.  Jemalloc 이 가져올수있는 장점과 실제 응용프로그램구현과 관련이있다. Jemalloc 의 사용여부의 결정은 build 를 참고하라

Skynet 은 jemalloc 과 무관한 memory hook 을 한 개 구현하고있다.  서비스의 내부 메모리 점용통계용으로 사용되고 C층에 메모리 릭이있는지 분석하는데도 사용된다.


원리는 각워킹스레드에 한 개의 TLS구역을 분배한다,워커에서 서비스의 메시지를 처리하기전 먼저 현재 서비스의 주소를 설정한다. 이렇게 메모리내부의 분배가 발생했을때 어떤서비스의 메모리인지 알수있다. 이건 어느정도 비용이 들어간다( 모든메모리에몇바이트의 오버헤드발생, 또한 어느정도의 이외의 비용지출), 만약 이게 문제로 생각된다면 꺼버려도 된다.

일반적으로 이 통계가 필요하지 않다, 대부분의 서비스는 lua 로 작업되고, DebugConsole 을통해 lua서비스의 메모리 상황을 체크할수 있기 때문이다. 이 통계의 주요 사용처는 직접 작성한 Lua 의 C 확장 라이브러리의 통계이다. 통계 인터페이스는 memory 라는 이름으로 Wrapping 되어 Lua에서 호출을 제공한다. 아니면 직접 cmemory 루아 서비스를 기동하여 통계 정보룰 출력하게 할 수 있다.

주: 1.0 beta 이후 lua 서비스가 사용하는 기본 분배기는 hook 메모리사용량통계를 사용하지 않는다. C 모듈의 메모리 사용만 통계한다. 만약 이행위를 변경하고 싶다면 다음을 읽어라
 https://github.com/cloudwu/skynet/blob/master/skynet-src/malloc_hook.c
참고  https://github.com/cloudwu/skynet/blob/master/service/cmemory.lua

단일 VM 에서 메모리의 최대량을 제한할 수있다 
참고: https://github.com/cloudwu/skynet/blob/master/test/testmemlimit.lua

Double Free 메모리 오버플로우 감시


C 모듈 중, 조심하지 않으면 동일한 메모리주소에 여러 번의 free 가 일어날 수 있다. 아니면 설정된 메모리공간을 넘어서서 데이터를 쓸 수도 있다 이것들은 모두 비교적 자주 보이는 버그의 원인이다. Skynet 이슈 중 아주 많은 coredump 보고의 원인은 모두 직간접적으로 이 오류와 관련이 있다.
비록 이종류의 버그 skynet 프레임워크가 책임지고 나오지 안도록 해야 하는 것은 아니지만 여기에 MEMORY_CHECK 검사의 큰 도움이 되는 것을 제공하고있다.


Malloc_hook.c 에 주석된 #define MEMORY_CHECK 를 해제한다. 매번 메모리를 할당할 때 메모리 꼬리에 추가적인 tag 를 달게 된다. FREE 할 때 이것을 다시 수정한다. 만약 중복 FREE 하게 되면 즉시발견 하게 된다 또한 이 기법은 부분적으로 메모리 침범도 검출할 수 있다.



출처: <https://github.com/cloudwu/skynet/wiki/MemoryHook> 
