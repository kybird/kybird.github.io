---
layout: post
title: DataCenter
date: 2022-03-17T13:53:54.026Z
category: Skynet
lastmod: 2022-03-17T14:14:56.199Z
---

datacenter 는 모든 skynet network 위에 노드들간의 데이터를 공유하는데 사용될 수 있다.  

노드들간에 데이터전송이 필요할때, 다른노드의 주소만 있더라도, 메시지를 보낼수 있다. 하지만 어떻게 주소를 얻을수 있나, 이것이 문제이다.

초기의 skynet 은 이름서비스를 제공했다, 하나의 서비스에 하나의 유일한 이름을 지정할수 있었다. 이름을 이용해서 즉시 메시지를 발송할 수 있었다. 하지만 현재 추천하는 방식은 datacenter 나 uniqueservice 를 통하는 것이다.

datacenter 는 모든네트워크가 공유하는 등록 table 과 비슷하다. 이는 tree 구조이며, 누구라도 테이블에 lua 데이터를 쓸수 있다. 노드들에 방문이 필요한 서비스, 스스로 자신의 주소를 datacenter  내에 기록하여, 필요한이가 읽어들일 수 있도록 한다.


데이터센터는 루아라이브러리이며, 아래처럼 사용한다:

```lua
local datacenter = require "skynet.datacenter"
```

진입한다.


세가지 사용방법이 존재한다

`datacenter.set(key1, key2, … , value)` key1과 key2에 대해 value 값을 설정한다. 이 API 는 적어도 두개의 파라메터가 필요하다. 트리구조의 계층수에대한 특별한 제한은 없다.

`datacenter.get(key1, key2, …)` key1 과 key2에서 값을 읽어온다. 이 API 는 적어도 한 개의 파라메터가 필요하다. 만약 다수의 파라메터를 넘기면, 트리의 가지를 리턴할것이다.


`datacenter.wait(key1, key2, …)` get 과 동일하다, 하지만 만약 가지가 nil 일때, 이 함수는 블록될수 있으며, 누군가 이 가지를 갱신했을때 반환한다. 읽고쓰기가 순서는 불분명하며, 다른곳에서 쓰여진 데이터를 읽은후 처리를 해야할때, 이를 사용하면 순환하며 읽어보는것보다 매우 좋은 방법이다. wait 는 필수적으로 하나의 잎 노드에서 사용되어야하며  branch 를 기다릴수 없다.
> 역주: 역시 문서만 봐서는 이해가안됨 잎노드, 가지노드, 여기노드는 무엇이냐. 


주의: 이 세가지 API 는 현재 coroutine 을 블럭한다. Async재진입 문제를 주의해야한다

[UniqueService](2022-03-17-UniqueService.md)와 비교해서, datacenter 사용은 더 유연하다.  dataceneter를 사용하여 multicast 의 채널번호등 각종 정보를 교환할수 있다. 하지만, datacenter 는 실제론 master 노드상에 배치된 전문 데이터센터서비스가 이러한 데이터를 공유하는것이다. 모든 datacenter 에대한 검색, 모두 중심노드와 통신이 필요하다 (만약 다수의 노드의 구조일경우), 이는 병목을 발생시킬수 있다. 만약 간단한 서비스의 주소관리만 필요하다면, UniqueService 를 사용하는것이 훨씬 낫다. 이 서비스는 각노드에 검색결과를 캐쉬한다.



# 클러스터모드에서사용

이기능은 클러스터모드에서 그대로 사용할수없다 스스로 구현해야한다.


출처: <https://github.com/cloudwu/skynet/wiki/DataCenter> 

