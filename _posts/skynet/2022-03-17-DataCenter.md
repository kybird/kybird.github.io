---
layout: post
title: DataCenter
date: 2022-03-17T13:53:54.026Z
category: Skynet
lastmod: 2022-03-17T14:14:56.199Z
---

datacenter 可用来在整个 skynet 网络做跨节点的数据共享。
Datacenter can be used for data sharing across nodes over the whole Skynet network.
Datacenter 는 모든 skynet network 위에서 노드들간의 데이터의 공유를위해 사용된다.

当你需要跨节点通讯时，虽然只要持有其它节点的地址，就可以发送消息。但地址如何获得，却是一个问题。
When you need across nodes data transmission, you only need the address of the other node to send a message, but it's a problem on how to get this address.
노드들간에 데이터전송이 필요할때 다른노드의 주소만있으면 메시지를 보낼수있다. 하지만 문제는 어떻게 이 주소를 가져오는가 이다. 

早期的 skynet 提供了具名服务的特性，可以给一个服务起一个唯一的名字，用名字即可发送消息。但目前更推荐的做法是通过 datacenter 或 UniqueService。
In the early implementation of Skynet, it offers a naming service, from the name you can tell it assigns each service a unique name and send messages with that name. But now the most recommended way is to send it over Datacenter or use UniqueService.
스카이넷 초기구현에서 이름서비스를 제공했다. 한개서비스에 한 개의 유일한 이름을 지정하고 이름을 사용하여 메시지를 발송했다. 하지만 현재 제일 추천하는 방식은 datacenter나 uniqueService 를 통하는것이다

datacenter 类似一个全网络共享的注册表。它是一个树结构，任何人都可以向其中写入一些合法的 lua 数据，其它服务可以从中取出来。所以你可以把一些需要跨节点访问的服务，自己把其地址记在 datacenter 中，需要的人可以读出。
Datacenter is similar to a shared registry for the network. It's in tree struct, anybody can write data in with valid Lua format, the other services can get data from there. So you can put the address of services that require across nodes data sharing in Datacenter and data consumer can read it.
데이터센터는 전체 네트워크공유의 등록표와 비슷하다. 이것은 트리구조이다. 누구나 합법적인 lua데이터를 쓸수있다. 다른서비스는그것을 읽어올수있다. 데이터센터에 데이터 공유가필요한 노드들의 주소를 넣을수있다 그리고 데이터가 필요한자가 그것을 읽을수있다.

datacenter 是一个 lua 库，使用：

Datacenter is a Lua lib, use:
데이터센터는 루아라이브러리이다 사용:

local datacenter = require "skynet.datacenter"




可以进入。
to get in.
진입한다.

一共有三个方法：
There are three ways to use it:
세가지사용방법이 존재한다

datacenter.set(key1, key2, ... , value) 可以向 key1.key2 设置一个值 value 。这个 api 至少需要两个参数，没有特别限制树结构的层级数。
datacenter.set(key1, key2, ... , value) set key1, key2... with value. this API requires at least two parameters, there is no limitation on how many levels of the tree.
Datacenter.set(key1, key2, … , value) key1과 key2를 value 로 설정한다 이 API 는 적어도 두개의 파라메터가 필요하다. 트리의 레벨이 몇이든 제한이 없다.

datacenter.get(key1, key2, ...) 从 key1.key2 读一个值。这个 api 至少需要一个参数，如果传入多个参数，则用来读出树的一个分支。
datacenter.get(key1, key2, ...) get the value from key1, key2... this API requires at least one parameter, if multiple values are passed in, it'll return the branch of the tree.
datacenter.get(key1, key2, …) key1 과 key2… 들에서 값을 가져온다. 이 API 는 적어도 한 개의 파라메터가 필요하다. 만약 다수의 값을 넣어준다면 이것은 트리의 가지를 리턴할것이다.



datacenter.wait(key1, key2, ...) 同 get 方法，但如果读取的分支为 nil 时，这个函数会阻塞，直到有人更新这个分支才返回。当读写次序不确定，但你需要读到其它地方写入的数据后再做后续事情时，用它比循环尝试读取要好的多。wait 必须作用于一个叶节点，不能等待一个分支。
datacenter.wait(key1, key2, ...) similar to get, but if the branch is nil, it will be blocked until there is an update in this branch. When read-write is not in order but you need to wait until you get the data of a specified node, it's better to use it than polling reading. wait has to be on a leave node not on a branch.
Datacenter.wait(key1, key2, …) get 과 비슷하다 하지만 만약 가지가 nil 이라면 이가지에 갱신이있을때까지 블럭된다. 읽고쓰기가 순서에 맞지않지만 지정한 노드의 데이터를 가져올때까지 기다려야할때는 polling reading 보다 이함수를 사용하는것이 좋다. 이함수는 leave node 에 사용되어야하고 branch 에 사용되어선 안된다,


注意：这三个 api 都会阻塞住当前 coroutine ，留心异步重入的问题。

Note: these 3 API will block the current coroutine. be careful on the reentrant async problem.

주의: 이 세가지 API 는 현재 coroutine 을 블럭한다. Async재진입 문제를 주의해야한다

和 UniqueService 相比较，datacenter 使用起来更加灵活。你还可以通过它来交换 Multicast 的频道号等各种信息。但是，datacenter 其实是通过在 master 节点上部署了一个专门的数据中心服务来共享这些数据的。所有对 datacenter 的查询，都需要和中心节点通讯（如果你是多节点的架构的话），这有时会造成瓶颈。如果你只需要简单的服务地址管理，UniqueService 做的更好，它会在每个节点都缓存查询结果。

Compared to UniqueService, datacenter is more flexible. You can use it to exchange the channel info of Multicast. However, Datacenter set a data-sharing service on the master node to achieve data sharing. Querying Datacenter requires communications with the master node (if you are using a multiple nodes framework), which might cause a bottleneck sometimes. If you only need simple data addressing management, UniqueService would be a better choice, it'll cache the querying results on each node.
UniqueService와 비교했을때 datacenter 는 더욱 유연하다. 이것을 멀티캐스트의 채널인포의 교환을 위해 사용할수 있다. 어쨋던 Datacenter 는 마스터 노드에 데이터공유 서비스를 설정하여 데이터 공유목적을 달성한다. DataCenter 에 요청하는것은 마스터노드와 통신이필요하다 (만약당신이 다중노드 프레임워크를 사용한다면) 이는 보틀넥을 야기할수있다.  만약 간단한 데이터 주소관리만을 필요하다면 UniqueService 는 더 좋은 선택이 될수있다. 이것은 각각의 노드에대한 쿼리 결과를 캐쉬할것이다.

在cluster模式下使用 Use in cluster mode
클러스터모드에서사용
该功能在cluster模式下不能直接使用，需要自己实现。

this feature is not available in cluster mode, use needs to implement it.
이기능은 클러스터모드에서 그대로 사용할수없다 스스로 구현해야한다.



