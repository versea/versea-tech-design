# Tapable

我们对 Tapable 并不陌生，一种发布订阅模式，Versea 的 Tapable 和 webpack 的 [Tapable](https://juejin.cn/post/7040982789650382855) 相似，仅仅少量区别。

Versea 有三种钩子：`AsyncParallelHook`, `AsyncSeriesHook`, `SyncHook`。

### Context

每种钩子都只接收一个 Context 参数，Context 会在整个钩子的执行流程存在，你可以读取或者修改 Context 值。

```typeScript
interface HooContext {
  foo: string;
}

const hook = new SyncHook<HooContext>();
hook.tap('ahook', context => {
  console.log(context.foo); // hello
  context.foo = 'world'
})
hook.tap('bhook', context => {
  console.log(context.foo); // world
})

hook.call({ foo: 'hello' })
```

### AsyncParallelHook, AsyncSeriesHook 和 SyncHook

- AsyncParallelHook 异步并行
- AsyncSeriesHook 异步串行
- SyncHook 同步串行

### 熔断（AsyncParallelHook 除外）

熔断用于中断 Hooks 执行，如下代码，`bhook` 将不会执行。

```typeScript
interface HooContext {
  foo: string;
}

async function delay(time: number): Promise<void> {
  return new Promise((resolve) => {
    setTimeout(resolve, time * 100);
  });
}

const hook = new AsyncSeriesHook<HooContext>();
hook.tap('ahook', async (context) => {
  console.log(context.foo); // hello
  await delay(1);
  context.bail = true
})
hook.tap('bhook', async context => {
  console.log(context.foo);
  await delay(1);
})

hook.call({ foo: 'hello' }) // 不执行 bhook
```

### ignoreTap（AsyncParallelHook 除外）

跳过某些 Hooks 执行。如下代码，`ahook` 将不会执行。

```typeScript
interface HooContext {
  foo: string;
}

async function delay(time: number): Promise<void> {
  return new Promise((resolve) => {
    setTimeout(resolve, time * 100);
  });
}

const hook = new AsyncSeriesHook<HooContext>();
hook.tap('ahook', async (context) => {
  console.log(context.foo);
  await delay(1);
})
hook.tap('bhook', async context => {
  console.log(context.foo); // hello
  await delay(1);
})

hook.call({ foo: 'hello', ignoreTap: ['ahook'] }) // 不执行 ahook
```

### Hooks 的优先级

before, after, 和 priority 都可以影响优先级。

- before: 将 hook 插入某个已有的 hook 之前。
- after: 将 hook 插入某个已有的 hook 之后。
- priority: 根据 priority 大小，越小优先级越高，会越靠前。

如下代码：会先执行 `bhook`, 在执行 `ahook`。

```typeScript
interface HooContext {
  foo: string;
}

async function delay(time: number): Promise<void> {
  return new Promise((resolve) => {
    setTimeout(resolve, time * 100);
  });
}

const hook = new AsyncSeriesHook<HooContext>();
hook.tap('ahook', async (context) => {
  console.log(context.foo); // world
  await delay(1);
})
hook.tap('bhook', async context => {
  console.log(context.foo); // hello
  await delay(1);
  context.foo = 'world'
}, {
  before: 'ahook'
})

hook.call({ foo: 'hello' });
```


### 接口设计

```typeScript
export interface TapOptions {
  before?: string;
  after?: string;
  priority?: number;
  replace?: boolean;
  once?: boolean;
}

export interface AsyncSeriesHook<T exnteds HookContext> {
  call: (context: T) => Promise<void>
  tap: (name: string, fn: (context: T) => Promise<void>, options: TapOptions)
}
```
