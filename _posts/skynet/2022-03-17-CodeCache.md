---
layout: post
title: CodeCache
date: 2022-03-17T13:53:54.026Z
category: Skynet
lastmod: 2022-04-09T12:31:20.284Z
---

스카이넷은 공식Lua구현을 변형하였다(옵션), 하나의 특성을 추가했다. [여러 개의 Lua VM 이 동일한 함수의 원형을 공유](https://blog.codingnow.com/2014/03/lua_shared_proto.html)할수있다. 동일한 skynet 프로세스중 대량의 lua VM을 기동했을시, 이 특성은 적지않은 메모리의 절약과 VM 시작시간의 향상을 가져온다.

---
> 역주: 먼 소리일까...  
> 
> 一旦检索到之前有加载过相同文件名的 lua 文件，则从内存中找到之前的函数原型替代.  
> 
> It uses the file name as the key, once it finds the same Lua file has been loaded, it will load the existing prototype and replace it.  
> 
> 이들은 파일이름을 key로 사용하여, 일단 같은 이름의 lua파일이 로딩된것을 찾아내면, 메모리에서 이전의 함수원형을 대체해버린다.  
> 


这个特性的使用，对一般用户来说是透明的。它改写了 lua 的辅助 API luaL_loadfilex ，所有直接或间接调用这个 api 都会受其影响。比如：loadfile 、require 等。它以文件名做 key ，一旦检索到之前有加载过相同文件名的 lua 文件，则从内存中找到之前的函数原型替代。注：Lua 函数是由函数原型以及 0 或多个 upvalue 绑定而成。


This feature is implicit to most users. It changes the auxiliary API luaL_loadfilex, it will affect all direct calls or indirect calls. For example, loadfile, require, etc. It uses the file name as the key, once it finds the same Lua file has been loaded, it will load the existing prototype and replace it. Note: Luas function is consists of prototypes and 0 or more upvalue bindings.


이 특성의 사용은, 일반 유저에게는 보이지 않는다. lua의 보조 API luaL_loadfilex 를 변경하여, 직간접적으로 이 API 를 호출 하는 모든것에 영향을 준다.

예를들어: loadfile, require 등.

이름을 key 로 사용하며, 같은이름의 lua 파일이 로딩된것을 찾아내면, 메모리에 로딩된 함수의 원형으로 대체한다.
주: Lua 함수는 원형과 0이나 여러개의 upvalue 바인딩으로 이루어져있다.



---


`loadString` 은 영향받지않는다. 그래서 만약 당신이 한 개의 lua 파일을 여러 번 로딩할필요가 있다면 io.open 을 이용해서 파일을 열고 load 를 사용하여 로드한다.


코드 캐쉬 는 추가는 하되 삭제하지 않는 전략을 사용한다. 다시말해, 스크립트의 사본을 일단 로딩하면, 프로세스가 종료되기전에, 영원히 점용한 메모리를 해제하지않는다(재로딩되지도 않는다). 대부분의 경우에, 이것은 문제가 되지 않는다.


스카이넷은 또한 디버깅목적을 위해 캐쉬된 메모리를 삭제하는 인터페이스를 제공한다.  

```lua
local cache = require "skynet.codecache"
cache.clear()
```

이렇게 코드캐쉬를 정리할수있다. 이 API 는 스레드안전하며, 이전버전의 데이터는 메모리에 남아있다(인용되고 있을수도 있다), 하지만 주의 해야한다. 단순히 캐쉬삭제에 의존하여 핫픽스를 진행하는 방안은 완전하지 않다. 핫픽스의 완전성과 이 특성의 도입여부와 관계없다. 왜냐하면 당신의 시스템이 루아 스크립트 덩어리를 로딩할때, 소스파일 업데이트에만 의존하고, 이 스크립트 로딩의 원자성을 보장할수 없기 때문이다 (어떤부분은 예전버전것, 어떤부분은 새버전의것)


주의, codecache.clear 는 그저 새로운 캐쉬를 생성한다 (API 이름이 오해를 부른다), 메모리를 해제하지 않는다. 그래서 이것을 여러 번 호출하지 말아야 한다. 만약 로드하려는 파일이 캐쉬에 영향을 받지 않는게 필요하다면, 올바른방법은 스스로 코드를 읽어드린후, loadstring 방식으로 스크립트를 로딩하는것이지, 로딩전에 codecache.clear 를 호출하는것이 아니다.

```lua
cache.mode(mode)
```

이 API 는 codecache 의 현재서비스의 모드를 변경한다. 모드는 ON/OFF/EXIST 가될수있으며 기본은 ON 이다.



* 모드가 ON 일때, 현재 서비스는 모든 로딩하는 Lua 스크립트 파일을 캐쉬한다.

* 모드가 OFF 일때, 현재 서비스는 Lua 코드를 재사용하려는 모든 행위를 막으며, 다른 서비스가 같은 파일이름을 로딩하더라도 마찬가지이다.

---

* 当 mode 为 "EXIST" 的时候，当前服务在加载曾经在其它服务或自己的服务加载过同名文件时，复用之前的拷贝。但对新加载的文件则不进行 cache 。注：通常可以让 skynet 本身被 cache 。
* When mode is set to "EXIST", the current service will reuse the previous copy if it's already loaded the same file name by itself or other services. But it will not cache any new loading files. Note: this can cache Skynet itself.

* 모드가 EXIST일때, 현재서비스가 다른서비스에서 로딩되었었거나 같은이름의 파일이 로딩된적이 있다면 이전에 로딩된 내용을 재사용한다. 하지만 새로 로딩되는 파일은 cache 를 진행하지 않는다. 주: skynet 스스로가 cache 될수 있다.



> 역: ??? 무슨말일까 ???  
> 
> 注：通常可以让 skynet 本身被 cache 。 ????  
> 
> Note: this can cache Skynet itself. ????  
> 
> 주: 통상 skynet 자신이 cache 되게 한다.  ????  
> 

---


API 파라메터가 공백이면, 현재 모드를 리턴한다.

주: 기본모드가 ON 인이유로. cache.mode를 처음으로 호출한 파일은 항상 cache 된다.

> 역: ??? 무슨 소리일까  
>  
> cache.mode('off') 라고 해도 그파일은 cache 된다는말이냐?  
> 
> 처음하면 캐쉬되고 두번째는 괜찮다는말인가..먼소리인가..기본값이 ON 이면 호출안해도 항상 캐쉬되는거 아니냐?  
> 

출처: <https://github.com/cloudwu/skynet/wiki/CodeCache> 


