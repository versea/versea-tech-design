# AppSwitcher

AppSwitcher 切换应用主要流程如下：

1. 加载匹配的应用。
2. 比较已有的应用和将要加载的应用，解析出需要 unmount 的应用和需要 mount 的应用。
3. unmount。
4. mount。