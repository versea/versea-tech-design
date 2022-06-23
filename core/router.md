# 路由

### reroute 流程

```plantuml
@startuml
actor User
rectangle "Versea Core" as VerseaCore #F5F5F5;line.dashed {
  rectangle "RewriteNavigationEvent"

  rectangle "Router Module" as RouterModule #DAE8FC {
    rectangle "Router"
  }

  rectangle "Application" #F8CECC {
    rectangle "AppService"
  }

  rectangle "AppSwitcher Module" #D5E8D4 {
    rectangle "AppSwitcher" as AppSwitcher
    rectangle "AppSwitcherContext" as AppSwitcherContext

    AppSwitcher --> AppSwitcherContext: 4.createContext
  }

  Router --> Router: 1.match routes
  Router --> Router: 2.getMatchedRoutesTree
  Router --> AppSwitcher: 3.performance
  AppSwitcher --> AppService: 5.changeApp
  AppService -> AppService: 6. load, mount, unmount
}
User --> VerseaCore: import
VerseaCore -> RewriteNavigationEvent

note top of RewriteNavigationEvent
  1. autorun while import
  2. rewrite pushState, replaceState
  3. rewrite addEventListener hashchange event, popstate event
end note

User --> RouterModule: start or event
@enduml
```

reroute 流程较为复杂，是整个 versea/core 的核心，控制整个应用加载，卸载流程，这里重点介绍一下 AppSwitcherContext，它需要记录 App 的加载顺序，mount 顺序和卸载顺序。当 changeApp 触发时，会生成一个新的 AppSwitcherContext。然后销毁当前的 AppSwitcherContext，使用新的 AppSwitcherContext 替代它。
