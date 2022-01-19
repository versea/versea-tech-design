# 简介

### 结构

versea 主要分为两部分：core 和 plugins。core 是 versea 的骨架，包含路由状态，路由匹配，应用加载，应用销毁，插件接入等；plugins 提供了解决常见问题的能力，例如：plugin-dependencies 用于解决应用的依赖加载和卸载能力，plugin-resource-entry 用于解决按 js 和 css 入口配置资源文件加载应用，plugin-html-entry 用于解决按 html 入口配置资源文件加载应用。当然，你也可以自己自定义插件，解决自己的业务问题。