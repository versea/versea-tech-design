# 简介

versea 是一个微前端框架

### 现状

现阶段微前端的能力有以下不足：

1. 嵌套能力弱。虽然各种框架都有实现嵌套能力，但是普遍存在多级基座的问题，如下图所示，`ChildApp` 既是 `MainApp` 的字应用，也是 `SubChildApp` 的主应用，是一个二级基座，这样的设计造成信息在 `MainApp` 丢失，如：`MainApp` 无法直接决定 `SubChildApp` 在在哪个页面渲染，是否需要渲染。这样的场景通常出现在权限和404 设计中。

```plantuml
@startuml
rectangle MainApp #F5F5F5 {
  rectangle ChildApp #F8CECC {
    rectangle SubChildApp    
  }
}

note top of ChildApp
  基座应用还是子应用？
end note

@enduml
```

2. 嵌套情况下，加载速度不如预期

```plantuml
@startuml
MainApp -> ChildApp: 加载 ChildApp
ChildApp --> MainApp: 加载 ChildApp 完成
MainApp -> ChildApp: 渲染 ChildApp
ChildApp -> SubChildApp: 加载 SubChildApp
SubChildApp --> ChildApp: 加载 SubChildApp 完成
ChildApp -> SubChildApp: 渲染 SubChildApp
@enduml
```

预期如下:

```plantuml
@startuml
group 并行加载
  MainApp -> ChildApp: 加载 ChildApp
  MainApp -> SubChildApp: 加载 SubChildApp
end
group 串行渲染
ChildApp --> MainApp: 加载 ChildApp 完成
MainApp -> ChildApp: 渲染 ChildApp
SubChildApp --> MainApp: 加载 SubChildApp 完成
MainApp -> SubChildApp: 渲染 SubChildApp 到 ChildApp 子容器
end
@enduml
```

3. 插件能力不足
目前微前端普遍只有 dispatch Event，但是无法改变加载和渲染行为。因此只能获取事件，但是无法增加插件，也就完成修改加载顺序，自定义沙箱这样的需求。

### 目标

1. 微前端加载应用，渲染，销毁能力。
2. 单主应用，不存在二级基座。
3. versea 路由匹配和拦截。双路由配置，versea 路由决定加载应用，应用路由决定加载组件。
4. 插件能力，类似 webpack 的插件能力。
5. 扩展能力，可以扩展任何代码，包括 versea 核心逻辑，深度用户可以基于此框架开发自己特定需求的微前端框架。

### 架构

![image](./assets/architecture.jpg)

- 底层能力：插件能力，扩展能力
- core: versea 核心逻辑，应用注册，路由匹配，应用加载
- plugin-xxx: 插件是可选项，用户选择自己需要的插件