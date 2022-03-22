---
layout: post
title: CodeCache
date: 2022-03-17T13:53:54.026Z
category: Skynet
lastmod: 2022-03-22T15:05:57.374Z
---

skynet 修改了 Lua 的官方实现（可选），加入了一个新特性，可以让多个 Lua VM 共享相同的函数原型1。当在同一个 skynet 进程中开启了大量 lua VM 时，这个特性可以节省不少内存，且提高了 VM 启动速度。

Skynet changed the default implementation of Lua and introduced a new feature that can make Lua VM share the same function prototypes1. When many Lua VM instances are created in the same Skynet process, this feature will greatly reduce memory consumption and will improve the VM start time.

스카이넷은 공식Lua구현을 변형하였다,. 한 개의 특성을 추가했다. 여러 개의 LuaVM 이 동일한 함수의 원형을 공유할수있다. 

这个特性的使用，对一般用户来说是透明的。它改写了 lua 的辅助 API luaL_loadfilex ，所有直接或间接调用这个 api 都会受其影响。比如：loadfile 、require 等。它以文件名做 key ，一旦检索到之前有加载过相同文件名的 lua 文件，则从内存中找到之前的函数原型替代。注：Lua 函数是由函数原型以及 0 或多个 upvalue 绑定而成。

This feature is implicit to most users. It changes the auxiliary API luaL_loadfilex, it will affect all direct calls or indirect calls. For example, loadfile, require, etc. It uses the file name as the key, once it finds the same Lua file has been loaded, it will load the existing prototype and replace it. Note: Luas function is consists of prototypes and 0 or more upvalue bindings.

이특성의 사용은 일반유저에게암시적이다. 보조적인 API luaL_loadfilex 를 변경한다, 모든 직접혹은간접적으로 이 API 를 호출하는것은 모두 영향을 받을수있다. 예를들어 loadfile, require 등. 파일명을 키로 삼고, 일단검색되기전에  동일한이름의 lua 파일을 로딩한다.  메모리에서 이전의 함수 원형을 찾으면 대체한다. 주: 루아함수는프로토타입으로 이루어져있고 0이나 upvalue 바인딩이다. 

`loadstring` 不受其影响。所以，如果你需要多次加载一份 lua 文件，可以使用 io.open 打开文件，并使用 load 加载。

`loadstring` is not affected by this change. So, if you want to load the same Lua file, io.open can be used to open the file and use load to load it.

`loadString` 은 영향받지않는다. 그래서 만약 당신이  한 개의 lua 파일을 여러 번 로딩할필요가 있다면 io.open 을 이용해서 파일을 열고 load 를 사용하여 로드한다.

代码缓存采用只增加不删除的策略，也就是说，一旦你加载过一份脚本，那么到进程结束前，它占据的内存永远不会释放（也不会被加载多次）。在大多数情况下，这不会有问题。

Loading cache uses adding without deleting strategy, which means, once you load a copy of the script, it will not be freed (and will not be reloaded) for the lifetime of the process. In most cases, it works fine without a problem.

로딩캐쉬는 추가는 하되 삭제하지 않는 전략을 사용한다. 이뜻은 스크립트의 사본을 일단 로딩하면 그것은 지워지지 않을것이다(그리고 재로딩되지도 않을것이다) 프로세스의 일생동안. 대부분의 경우에 이것은 문제없이 동작한다.

skynet 留出了接口清理缓存，以做一些调试工作。接口模块叫做 skynet.codecache 。

Skynet also provides an interface to clean up the cached memory for debugging purposes, the name of the module is called skynet.codecache.

스카이넷은 또한 디버깅목적을 위해 캐쉬된 메모리를 삭제하는 인터페이스를 제공한다. 
```lua
local cache = require "skynet.codecache"
cache.clear()
```

这样就可以清理掉代码缓存。这个 api 是线程安全的，且老版本的数据依旧在内存中（可能被引用）。但需注意，单纯靠清理缓存的方式做热更新的方案是不完备的。这个完备性和是否引入这个特性无关。因为当你的系统在加载一批 lua 脚本时，单靠源文件的更新，无法保证这批脚本加载的原子性。（有部分是旧版本的，有部分是新版本的）
이렇게 코드캐쉬를 정리할수있다. 이 API 는 스레드안전하다. 동시에 이전버전의 데이터는 메모리에 남아있다(사용되어버렸을수도있다). 하지만 주의 해야한다. 단순히 캐쉬삭제에 의존하여 핫픽스를 진행하는 방안은 완전하지 않다. 이 완전성과 이특성의 도입여부와 관련없다. 왜냐하면 당신의 시스템이 루아스크립트 덩어리를 로딩할때, 소스파일업데이트에만 의존하고  이 스크립트로딩의 원자성을 보장할수 없기 때문이다 (어떤부분은 예전버전것, 어떤부분은 새버전의것)

This will clean up the code memory cache. It's a thread-safe API and the previous version of data can still be referenced in memory. Note: it's incomplete to use this feature for hot-patch. The completeness isn't related to introducing this feature. Because when reloading a set of Lua scripts, it's not guaranteed it'll be an atomic operation. (some of the scripts might be old versions while the others might be updated to new version)

注意，codecache.clear() 仅仅只是创建一个新的 cache （ api 名字容易引起误会），而不释放内存。所以不要频繁调用。如果你需要你加载文件不受 cache 影响，正确的方式是自己读出代码文本，并用 loadstring 方式加载；而不是在加载前调用 codecache.clear 。
Note, codecache.clear() only creates a new cache (the name of API may cause some confusion）, it will not free memory, so please don't call it multiple times. If the file you are about to load is not affected by the memory cache, the right way to do this is to load the script and then use loadstring to load it instead of loading it after codecache.clear call.

주의 codecache.clear 는 오직 새로운 캐쉬를 생성한다(API 이름이 오해를 부른다), 메모리를 해제하지 않는다. 그래서 이것을 여러 번 호출하지 않기를 바란다. 만약 당신이 로드하려는 파일이 메모리캐쉬에 영향을 받지 않는다면.  올바른방법은  스크립트를 로드하고 codecache.clear 를 호출한후 load 하는것이 아니라 loadstring 을 사용하여 로드하는것이다.
```lua
cache.mode(mode)
```
这个 API 可以修改 codecache 在当前服务中的工作模式。mode 可以是 "ON" "OFF" "EXIST" ，默认的 mode 为 "ON" 。

This API can change this mode of the codecache of the current service. mode can be one of these options: "ON" "OFF" "EXIST", the default is "ON".

이 API 는 codecache 의 현재서비스의 모드를 변경한다. 모드는 ON/OFF/EXIST 가될수있으며 기본은 ON 이다.

* 当 mode 为 "ON" 的时候，当前服务 cache 一切加载 lua 代码文件的行为。
When mode is set to "ON", the current service will cache all behaviors of loading Lua script files.

* 모드가 ON 으로 셋되었을때 현재서비스는 로딩중인 루아스크립트 파일의 모든 행동을 캐쉬한다.

* 当 mode 为 "OFF" 的时候，当前服务关闭任何重复利用 lua 代码文件的行为，即使在别的服务中曾经加载过同名文件。
When mode is set to "OFF", the current service will close any behavior that reuses Lua code, even if other services might have loaded the same file name.

* 모드가 OFF 일때 현재 서비스는 재사용된 루아코드의 모든 행위를 닫는다.  다른서비스들이 같은 파일을 로딩했더라도..


* 当 mode 为 "EXIST" 的时候，当前服务在加载曾经在其它服务或自己的服务加载过同名文件时，复用之前的拷贝。但对新加载的文件则不进行 cache 。注：通常可以让 skynet 本身被 cache 。
* When mode is set to "EXIST", the current service will reuse the previous copy if it's already loaded the same file name by itself or other services. But it will not cache any new loading files. Note: this can cache Skynet itself.
当 api 参数为空时，返回当前的 mode 。

모드가 exist 로 설정되었을때 만약  같은 이름의 파일을 스스로 로딩했거나 다른서비스에의해 로딩되었다면 현재서비스는  이전사본을 재사용한다.  주:  일반적으로 skynet 자체를 cache 하게 한다.
When API is set to empty, it will return its current mode.
API 가 empty 로 설정되면 이것은 현재 모드를 리턴한다.

注意：由于默认模式是打开状态，所以你第一次调用 cache.mode 的所在文件一定是被 cache 的。

Note: because default mode is "ON", so the first time you call cache.mode in a file, it will be cached for that file for sure.

주의: 기본모드가 ON 인이유로. cache.mode를 첫번째호출한 파일은 절대로 cache 된다.
