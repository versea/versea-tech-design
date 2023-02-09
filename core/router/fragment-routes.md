# 碎片和路由合并

之前留了一个疑问，[apps](./basic.md) 是一个数组，为什么一个路由会对应多个应用。我们这里会解释这个问题。

我们先看如下两个页面，路由和页面结构参考[上文](./nested-routes.md)

- case1

![image](../../assets/merge-routes-case1.png)

B 应用不具有嵌套路由的能力，没有声明 slot，但是希望在访问 `/foo/bar` 的时候能渲染 C 应用。C 应用在 B 内部。

- case2

![image](../../assets/merge-routes-case2.png)

B 应用不具有嵌套路由的能力，没有声明 slot，但是希望在访问 `/foo/bar` 的时候能渲染 C 应用。C 应用在 B 外部，但都在 A 应用下。

### 路由合并

假设 A 应用，B 应用和 C 应用都注册了 routes. 结果如下：

```ts
const routes = [
  {
    path: '/foo',
    slot: 'foo-container1',
    apps: [A],
  },
  {
    path: '/bar',
    fill: 'foo-container1',
    apps: [B],
    children: [
      {
        path: '/baz',
        apps: [B],
      },
    ],
  },
  {
    path: '/bar',
    apps: [C],
  },
];
```

C 的 `/bar` 路由也要插入 `/foo` 作为 children。

```ts
const routes = [
  {
    path: '/foo',
    slot: 'foo-container1',
    apps: [A],
  },
  {
    path: '/bar',
    fill: 'foo-container1',
    apps: [B],
    children: [
      {
        path: '/baz',
        apps: [B],
      },
    ],
  },
  {
    path: '/bar',
    fill: 'foo-container1',
    apps: [C],
  },
];
```

循环合并，第一次合并嵌套路由之后结果如下

```ts
const routes = [
  {
    path: '/foo',
    slot: 'foo-container1',
    apps: [A],
    children: [
      {
        path: '/bar',
        fill: 'foo-container1',
        apps: [B],
        children: [
          {
            path: '/baz',
            apps: [B],
          },
        ],
      },
    ],
  },
  {
    path: '/bar',
    fill: 'foo-container1',
    apps: [C],
  },
];
```

第二次合并发现相同路由，合并 apps，于是变成如下结构

```ts
const routes = [
  {
    path: '/foo',
    slot: 'foo-container1',
    apps: [A],
    children: [
      {
        path: '/bar',
        fill: 'foo-container1',
        apps: [B, C],
        children: [
          {
            path: '/baz',
            apps: [B],
          },
        ],
      },
    ],
  },
];
```

为什么这样的结构，访问 `/foo/bar` 就可以渲染成上面的两种页面结构呢？这个等会解释，先看一下子应用的主次，也就 B 应用为什么在 C 应用之前。

### 子应用的主次

我们先看如下路由结构

```ts
const routes = [
  {
    path: '/foo',
    slot: 'foo-container1',
    apps: [A],
    children: [
      {
        path: '/bar',
        fill: 'foo-container1',
        slot: 'bar-container1',
        apps: [B, C],
        children: [
          {
            path: '/baz',
            apps: [B],
          },
        ],
      },
    ],
  },
  {
    path: '/zoo',
    fill: 'bar-container1',
    apps: [D],
  },
];
```

根据前面提到的内容，`/zoo` 对应的 route 需要作为 `/bar` 这个 route 的 children，那么这种情况下 D 应用应该渲染在 B 应用内部还是 C 应用内部呢？

我们好像需要增加字段声明，但是这种结构过于复杂。我们为了解决 D 应用渲染在 B 应用内还是 C 应用内这个问题，versea 在这里做了妥协，每一个路由只能有一个普通应用，其他全部是碎片应用。在这种设计下，我们把应用 C 叫做碎片应用。B 叫做普通应用，D 应用只能渲染在父级 Route 的普通应用 B 下。因此，B 和 C 应当满足一定条件才能合并。

碎片应用对应路由条件

- 必须声明 `isFragment: true`
- 不能声明 children

所以 C 想要正常合并作为 `/foo` 的 children，应该修改成如下路由结构

```ts
const routes = [
  {
    path: '/foo',
    slot: 'foo-container1',
    apps: [A],
  },
  {
    path: '/bar',
    fill: 'foo-container1',
    apps: [B],
    children: [
      {
        path: '/baz',
        apps: [B],
      },
    ],
  },
  {
    path: '/bar',
    fill: 'foo-container1',
    isFragment: true,
    apps: [C],
    // 不能具有 children
  },
];
```

合并之后，`/bar` 对应的 `apps: [B, C]`，B 作为普通应用，C 作为碎片应用，如果有 `/zoo` 这样的 D 应用合并，那么 D 应该渲染在 B 应用内部。

C 不能具有 children，那是不是 C 就不能有路由呢？C 应用内部可以使用 VueRouter, ReactRouter 都没有关系，他只是能不能在那里具有 App Route。简单来说 C 应用内 Component Router 随意使用，但是 C 应用不能具有 App Route。

### 渲染顺序

为什么这种结构可以渲染成上面的两种页面结构呢？

versea 规定，优先渲染普通应用，再渲染碎片应用，于是，无论 C 应用在 B 应用内部，还是不在 B 应用内部，都可以保证 C 正常渲染，因为渲染时序一定是 A -> B -> C。

### meta 在合并中的处理

路由设计一般都会涉及 meta，versea 的 `Router` 也不例外，那么合并时 meta 冲突怎么处理呢？

我们采用 scope 的方式处理
```ts
const routes = [
  {
    path: '/foo',
    slot: 'foo-container1',
    apps: [A],
  },
  {
    path: '/bar',
    fill: 'foo-container1',
    apps: [B],
    meta: {
      test: 1,
    },
    children: [
      {
        path: '/baz',
        apps: [B],
      },
    ],
  },
  {
    path: '/bar',
    fill: 'foo-container1',
    isFragment: true,
    apps: [C],
    meta: {
      test: 2,
    },
    // 不能具有 children
  },
];
```

合并之后结构如下

```ts
const routes = [
  {
    path: '/foo',
    slot: 'foo-container1',
    apps: [A],
  },
  {
    path: '/bar',
    fill: 'foo-container1',
    apps: [B, C],
    meta: {
      test: 1,
      [C App Name]: {
        test: 2,
      }
    },
    children: [
      {
        path: '/baz',
        apps: [B],
      },
    ],
  },
];
```

meta 中增加一个 C 应用名称的 key 用于存放 C 的 meta 信息。
