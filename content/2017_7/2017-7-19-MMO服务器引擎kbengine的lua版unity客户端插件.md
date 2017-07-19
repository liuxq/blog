之前做的基于kbengine的一个lua版的mmo小游戏https://github.com/liuxq/StriveGame
有同学提issue希望把插件层独立出来。于是做了逻辑和插件层的解耦，主要代码结构如下：

![dock](https://raw.githubusercontent.com/liuxq/blog/master/images/kbeplugins/code.png)

其中Csharp文件夹中的memoryStream, NetworkInterface, PacketReceiver, PacketSender 都是网络基础类，基本上不会改动热更，并没有lua化；Dbg, Profile, Event, 在lua层都有类似功能，暂时保留c#版本；
KBELuaUtil是lua层需要用到的一些基础函数，还有c#调用lua函数的工具函数指针（为了和tolua解耦，比如如果要用xlua之类的可以传入相应lua插件的调用lua接口）。

其中Lua文件夹中的eventlib.lua和events.lua是lua层的事件结构（event依赖于tolua，如果不用tolua需要修改这里）；
其他的lua文件都是同名c#插件的lua化。

为了验证插件的可用性，把kbengine的官方demo做了lua化，包含了所有逻辑：ui，移动，镜头等
游戏截图：

![dock](https://raw.githubusercontent.com/liuxq/blog/master/images/kbeplugins/login.png)
![dock](https://raw.githubusercontent.com/liuxq/blog/master/images/kbeplugins/game.png)

最后附上plugins和demo的github地址:

plugins: https://github.com/liuxq/kbengine_unity3d_lua_plugins

demo: https://github.com/liuxq/kbengine_unity3d_tolua_demo
