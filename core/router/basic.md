# Router

`Router` 即 App Router，他负责整个路由注册，路由匹配逻辑。

### RouteConfig 和 Route

`RouteConfig` 是用户传给我们的路由信息，`Route` 是每个路由信息的实例，举个例子：

```ts
interface RouteConfig {
  path: string;
  children: RouteConfig[]
}


class Route {
  path: string;
  children: Route[];

  constructor(config: RouteConfig) {
    this.path = config.path;
  }
}

const config = { path: foo };
const route = new Route(config);
```

### 注册 Route

前面提到，我们[注册应用](../app.md)会把 routes 的传给 Router.registerRoutes，我们这里看一下 registerRoutes 逻辑。

```ts
class Route {
  constructor(config: RouteConfig) {
    this.path = config.path;

    this.children = config.children.map(item => new Route(item));
  }
}

class Router {
  routes: Route[] = [];

  registerRoutes(configList: RouteConfig[]) {
    this.routes = [
      ...this.routes,
      ...configList.map(config => new Route(config)),
    ];
  }
}
```

- 我们处理了 children, 这样便形成了一个 `Route Tree`。
- 增加一个 Router 类，用于管理 `Route Tree`。

### 关联 App

前面的处理，只是注册了 Route，但是没有关联 `App`，我们注意到，[注册应用](../app.md)会把 routes 的传给 Router.registerRoutes，同时还传了 `App` 的实例。

```ts
class Route {
  apps: App[]

  constructor(config: RouteConfig, app: App) {
    this.path = config.path;
    this.apps = [app];

    this.children = config.children.map(item => new Route(item, app));
  }
}

class Router {
  routes: Route[] = [];

  registerRoutes(configList: RouteConfig[], app: App) {
    this.routes = [
      ...this.routes,
      ...configList.map(config => new Route(config, app)),
    ];
  }
}
```

增加 app 参数，每个 `Route` 便关联了 `App`。这里为什么是 apps，一个数组而不是一个 App，这个和碎片应用有关，我们会在后续介绍。

### Route parent

为了方便取值，我们可以增加 parent 属性。

```ts
class Route {
  apps: App[]

  constructor(config: RouteConfig, app: App, parent: Route | null = null) {
    this.path = config.path;
    this.apps = [app];
    this.parent = parent;

    this.children = config.children.map(item => new Route(item, app, this));
  }
}
```

### 路由匹配

前面介绍了大量代码，只是希望对 `Route`, `Router` 有一个基础的认识，后续的逻辑可能会稍微复杂，我们不会写大量实现。

路由匹配分为两步

- 转化 Route Tree 为 Route Array
- 使用 Route Array 严格匹配路由

### 转化 Route Tree 为 Route Array

`Route[]` 的结构如下：

```ts
const routes = [
  { path: '/foo', children: [{ path: '/bar' }] },
  { path: '/baz' },
];
```

使用深度优先遍历转换 Route Tree 为 Route Array，转换后的结构如下。

```ts
const routeArray = [
  { path: '/foo' },
  { path: '/bar' }
  { path: '/baz' },
];
```

### Route Array 匹配路由

给每个 routes 增加一个 fullPath, fullPath 会拼接出完整的链接, 结构如下：

```ts
const routes = [
  { path: '/foo', fullPath: '/foo', children: [{ path: '/bar', fullPath: '/foo/bar' }] },
  { path: '/baz', fullPath: '/baz' },
];

// 转换成 Route Array 之后如下
const routeArray = [
  { path: '/foo', fullPath: '/foo' },
  { path: '/bar', fullPath: '/foo/bar' }
  { path: '/baz', fullPath: '/baz' },
];
```

使用 path-to-regexp 匹配当前链接，得到路由，取到所有 parent 加入数组， 生成 matchedRoutes。

例如：当前 location 是 `/foo`, matchedRoutes 是 `[{ path: '/foo' }]`。如果 当前 location 是 `/foo/bar`, 那么匹配结果是 `{ path: '/bar', fullPath: '/foo/bar' }`, 取parent 加入数组，那么 `{ path: '/foo' }` 也要加入，`{ path: '/foo' }` 的 parent 是 null, 不需要加入数组，matchedRoutes 的结果是 `[{ path: '/foo' }, { path: '/bar', fullPath: '/foo/bar' }]`。

严格匹配：如果当前 location 是 `/foo/xxx` 是不会匹配到 `{ path: '/foo' }` 的，结果是未匹配，这和大多数路由的设计一致。
