# 路由

### reroute 流程

<!--
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

    AppSwitcher -down-> AppSwitcherContext: 4.createContext
  }

  Router -down-> Router: 1.match routes
  Router -down-> Router: 2.getMatchedRoutesTree
  Router -down-> AppSwitcher: 3.performance
  AppSwitcher -down-> AppService: 5.changeApp
  AppService -> AppService: 6. load, mount, unmount
}
User -down-> VerseaCore: import
VerseaCore -> RewriteNavigationEvent

note top of RewriteNavigationEvent
  1. autorun while import
  2. rewrite pushState, replaceState
  3. rewrite addEventListener hashchange event, popstate event
end note

User -down-> RouterModule: start or event
@enduml
```
-->

![](https://www.plantuml.com/plantuml/svg/VLF1Yjim4BthAnxEDMjeTrE22sKN9tlgzj1qUnVsR2om9JCQ9uMo_rvPTeaQukABtzDxJpE3vj6BPXcwLkbA7EFL4okcIhGzjeJi9x4dZT8nPT0U4nuXLi-RyVlS6ajvhNr3DNuh875_fpCReM_wP8vQZBFx4rd9r9NA3KAC5rSFxNJBn4m4Lhkd_VQvZDatV5cWtwyId_g-DLMyCGjrRijzcVWJrO7uP2fQo3YSZLHDKjfgjzblTmynQaaS6qZmVwIbiqA_976aj8hEXCTTxSxsxiiDRO67l6BIGZCnDnGdcJWdME13tkdW1u_OB-i-vaUIbr5ATUJy3oQQzRShAd2VzyHlZZjjArBBBKopBx39goOCXmAda9pWIlSfH-jqlKRd1Yjh33R-g7VrwfFeonCjOBhUiQWBDMRUVfLAMIS4SJtSsv86ONBGWpWUBCwDQUdl5GYp0aykz8Dl3gA5re7gMrrHH45qVn7fkewXNuqNiiHej6-cIO36WpLwr_jWdB4YMsCiKltRSBej1U92m_7iUGociDv_0000)

reroute 流程较为复杂，是整个 versea/core 的核心，控制整个应用加载，卸载流程，这里重点介绍一下 AppSwitcherContext，它需要记录 App 的加载顺序，mount 顺序和卸载顺序。当 changeApp 触发时，会生成一个新的 AppSwitcherContext。然后销毁当前的 AppSwitcherContext，使用新的 AppSwitcherContext 替代它。
