# versea 架构

### 流程

<!--
```plantuml
@startuml
actor User
rectangle "Versea Core" #F5F5F5;line.dashed {
  rectangle Application #F8CECC {
  }
  rectangle Router #DAE8FC {
  }
  rectangle AppSwitcher #D5E8D4 {
  }
  Router -> AppSwitcher: 3. reroute
  AppSwitcher -down-> Application: 4. load, unmount & mount
}
User -down-> Application: 1. registerApp
User -down-> Router: 2. Location Change
@enduml
```
-->

![](https://www.plantuml.com/plantuml/svg/TT2nQiCm40RWNK_naq2dnj0s0UD2IL2SgLDBkWlhY0MA54vdCfIyUsLZMXiQD0YaxvVkRfl4i7HdhqmZaN5Cn8gf4HDEdh3u8avae2FJ0il3fb-ltWKgh4ajMNmhuC_lBXVl6YCkXgnBNMizYDjCVSHEYB7Sx-hoy1_8ptnUdJJje3PrkL__gZ6yUfkg2Yy5cBY_KvZbLpPUGzQJqYgi2_Xex2EwS8vT43nWsDLD7TEzq5F_nSab8SxdCpXMLU6vm7iS1w3Rt0sfBGMR1_m3)

1. 用户首先注册应用
2. 路由切换时会触发 `router.reroute`
3. reroute 会调用 AppSwitcher 会执行切换应用
