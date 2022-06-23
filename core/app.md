# 应用

### 应用注册流程

<!--
```plantuml
@startuml
actor User
rectangle "Versea Core" #F5F5F5;line.dashed {
  rectangle "Application" #F8CECC {
    rectangle "AppService"
    rectangle "App"
    AppService -> App: 2.new instance
  }
  rectangle "Router Module" #DAE8FC {
    rectangle "Router"
    rectangle "Route"
    Router -down-> Route: 4.new instance & generate RoutesTree
    Router -> Router: 5.reroute
  }
  Router <-up- App: 3.registerRoutes
}
AppService <-up- User: 1.registerApp
@enduml
```
-->

![](https://www.plantuml.com/plantuml/svg/RT5DIyGm40RWUtx5KC5R2tuiB5MMNUhkpOiVtaFoKWCnAPFKFSZ-TnDJT2kbb-RjCvrfPnkYv3X-M25Lz4ol0ImOAahNMr3r1WwGr7b6HHU7LRxkh75ej0plqFGbYCxyRXYiKJ8Qxx9VT_kk-p7_rJFuqoXK2uzAzcUetkHJIzUDmv6C2mah97MQDt_oOmJJezUZpUC-xFRhGsc_uAh5kAH5KAtzqTMRScpfTjQVBgc70yk80i8B0xFogP9RMZKCplVJr9EuhyUXBXztaqHlGahBoyH9dFs20lDsMhhwbGc8BPnE-_i6)

AppService 有一个重要的方法是 registerApp，registerApp 会生成 App 的实例，并把 registerApp 创建时传入的路由信息传给 Router 创建 Route 的实例，生成 RoutesTree。后续会用这个 RoutesTree 做路由匹配。
