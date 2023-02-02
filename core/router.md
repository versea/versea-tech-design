# Router

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
  Router -down-> AppSwitcher: 3.switch
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

![](https://www.plantuml.com/plantuml/png/VLF1Yjim5BphAuRacZMqkod1XRABaprrUsYwlGlxDHR8af6UaaBPVwyifKGDyUABPpGpUXgaHy_ISHYqa2rRXrTFJZXgMPg39Yn-alCaqLX72qYFZ2U8vVDw-ZvhPQZgfE-fmny15ExlT7AAwPiygeDaPStkI8ONuafb0vF3Y-s2phja9XDORfzsc-ScPT_mBIBzTfNuD8vQjMd7HPnpq-oQmb-ezkIEggMPZFr9STiNeostwzrc-v2ZPiJf00L-HzfOm_IR2qT9Y-GiUDnrzcJkljpGrdYeMaUIwKoS3vIQB9mPrlXG3JBwuIRivtKVgmFnooIbBd7-XoCwwszFLEE-ykbVp4-VQw-nhje-zaAH4oXMSANxL45RsQqms61uXM3IZtWJhxw8ljpE6ceOhMDneQRCsnTI26EPm7Q4_JMdGImDCh1rmU3KAqt_ja2i2IwxC0RVxraohI8rQjbIGWAuluXarwNGhyep6NBefUzp4w1fOONHZduoPYp8T9Y65FyVk7meDT9RXdLdpsCqXK7_1G00)

reroute 流程较为复杂，是整个 versea/core 的核心，控制整个应用加载，卸载流程，这里重点介绍一下 AppSwitcherContext。

- 每次路由匹配，都会生成一个新的 AppSwitcherContext，AppSwitcher 每次都保留一个最新的 AppSwitcherContext。
- 每个 AppSwitcherContext 都记录了当前路由信息和本次匹配到的路由信息，在路由切换过程中，只需要比较当前路由信息和匹配到的路由信息的差异，就可以决定如何卸载和加载应用。
