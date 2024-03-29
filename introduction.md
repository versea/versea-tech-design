# 简介

versea 是一个微前端框架

### 现状

现阶段微前端的能力有以下不足：

1. 嵌套能力弱。虽然各种框架都有实现嵌套能力，但是普遍存在多级基座的问题，如下图所示，`ChildApp` 既是 `MainApp` 的子应用，也是 `SubChildApp` 的主应用，是一个二级基座。这样的设计造成信息在 `MainApp` 丢失，如：`MainApp` 无法直接决定 `SubChildApp` 在在哪个页面渲染，是否需要渲染。这样的场景通常出现在权限和 404 设计中。另外还有一个严重的问题，`ChildApp` 只是一个业务开发团队，不需要深入理解 `MainApp` 的各种业务功能和微前端框架，可是这种架构让业务开发团队不得不了解上面那些功能，这种要求对业务开发团队太苛刻了。

<!--
```plantuml
@startuml
rectangle MainApp #F5F5F5 {
  rectangle ChildApp #F8CECC {
    rectangle SubChildApp {
    }
  }
}

note top of ChildApp
  基座应用还是子应用？
end note
@enduml
```
-->

![](https://www.plantuml.com/plantuml/svg/SoWkIImgAStDuIfAJIv9p4lFILNmJSpCSomeK51sCmL1XMek1GM4h7D6PauAHDh2sTNP6IoDh22uD0ck1WY0ihLSjLnSUVabgGeb-GKA-MbmCu1oJ-VlUhfh-TDTKvvFMV5Y_undCzO_NJi1mds_PpvNQbwA0aWdbmEG1NGc0000)

2. 嵌套情况下，加载速度不如预期

<!--
```plantuml
@startuml
MainApp -> ChildApp: 加载 ChildApp
MainApp <-- ChildApp: 加载 ChildApp 完成
MainApp -> ChildApp: 渲染 ChildApp
ChildApp -> SubChildApp: 加载 SubChildApp
ChildApp <-- SubChildApp: 加载 SubChildApp 完成
ChildApp -> SubChildApp: 渲染 SubChildApp
@enduml
```
-->

![](https://www.plantuml.com/plantuml/svg/SoWkIImgAStDuV9DpCnpB2XGqBLJSCx8p4a1igqKddOjUB5tBro02ym0gm8gW-Z0LANmT5tFiuu9s4ryjcFJi_cJqKo4gWekJKAt6bcC2y96E02dOeGZqCr7TmYA-GwfUId0e000)


预期如下:

<!--
```plantuml
@startuml
group 并行加载
  MainApp -> ChildApp: 加载 ChildApp
  MainApp -> SubChildApp: 加载 SubChildApp
end
group 串行渲染
MainApp <-- ChildApp: 加载 ChildApp 完成
MainApp -> ChildApp: 渲染 ChildApp
MainApp <-- SubChildApp: 加载 SubChildApp 完成
MainApp -> SubChildApp: 渲染 SubChildApp 到 ChildApp 子容器
end
@enduml
```
-->

![](https://www.plantuml.com/plantuml/svg/SoWkIImgAStDuKeloYyjK7YwSzkBXJrFknQysBkNIq71Dp4ppx6WG47NJi4v8pCd1SYsKe38mWLGbGMN9g6hH1RZIirBWLhrPCSce5NFTcnwDdyo5qnUGLVN3ceSfaqAJzVrFEkOm8NLLH234Au2MehNKA0yRdCHiWYZKNLqR41orje9JzVjV3fp1TXt3a026A00)

3. 插件能力不足
目前微前端普遍只有 dispatch Event，但是无法改变加载和渲染行为。因此类似修改加载顺序，自定义沙箱这样的需求无法实现。

### 目标

1. 微前端加载应用，渲染，销毁能力。
2. 单主应用，不存在二级基座。
3. versea 路由匹配和拦截。双路由配置，versea 路由决定加载应用，应用路由决定加载组件。
4. 插件能力，类似 webpack 的插件能力。
5. 扩展能力，可以扩展任何代码，包括 versea 核心逻辑，用户可以基于此框架开发自己特定需求的微前端框架。

### 架构

![image](./assets/architecture.jpg)

- 底层能力：插件能力，扩展能力。
- core: versea 核心逻辑，应用注册，路由匹配，应用加载。
- plugin-xxx: 插件是可选项，用户选择自己需要的插件。

> DI: 依赖注入。
> 替换能力：用户可以自己重写任何一个函数，实现整个框架所有内容均可扩展，也包括插件。
