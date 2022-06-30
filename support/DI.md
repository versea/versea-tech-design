# DI（Dependency Inject）

DI 是 Versea 的核心设计模式。

### DI 简介

DI 相关[介绍](https://juejin.cn/post/7073361691609661453)

Versea 使用了 `inversify` 这个 DI 框架，参考[文档](https://inversify.io/)。

简单使用如下：

```typescript
import { Container, injectable } from 'inversify';

@injectable()
class Foo {
  public foo() {
    return 'hello';
  }
}

@injectable()
class Bar {
  private _foo: Foo;

  public constructor(foo: Foo) {
    this._foo = foo;
  }

  public bar() { return this._foo.foo(); };
}

const container = new Container();
container.bind<Foo>(Foo).to(Foo);
container.bind<Bar>(Bar).to(Bar);
container.get(Bar).bar() // hello
```

### Auto Binding

上面的代码最后几行需要手动绑定，用户使用不太方便，如果能自动绑定会更好。

#### Auto Binding 原理

1. 使用 provide 装饰器替换 injectable 装饰器，provide 装饰的类会存到 state 内。
2. 提供一个 bind 方法，循环 state 执行绑定类。

```typescript
import { Container, injectable, inject } from 'inversify';

const state  = {};

function provide(name: string) {
  return function(target: any) {
    state[name] = target
  };
}

function bind(container) {
  Object.keys(state).map(name => {
    container.bind(name).to(state[name]);
  });
  state = {}
}

@provide('Foo')
class Foo {
  public foo() {
    return 'hello';
  }
}

@provide('Bar')
class Bar {
  private _foo: Foo;

  // 根据名称注入
  public constructor(@inject('Foo') foo: Foo) {
    this._foo = foo;
  }

  public bar() { return this._foo.foo(); };
}

// 用户使用
const container = new Container();
bind(container);
```

#### Auto Binding 效果

借助 Auto Binding 的能力，我们只需要专注于实现类的逻辑就好，不需要关注这些类怎么与容器绑定。

### 替换

1. 使用 provide 装饰器装饰另一个类，但 name 需要与被替换的类的名称相同
2. 经过 provide 装饰器如此装饰，存在来的类就被替换掉了。

基于替换能力，Versea 的任何逻辑都可以被用户在外部修改，也就做到了可扩展能力。

```typescript
import { Container, injectable, inject } from 'inversify';
import { provide, bind } from 'my-provide'

@provide('Foo')
class Foo {
  public foo() {
    return 'hello';
  }
}

@provide('Bar')
class Bar {
  private _foo: Foo;

  // 根据名称注入
  public constructor(@inject('Foo') foo: Foo) {
    this._foo = foo;
  }

  public bar() { return this._foo.foo(); };
}

// 用户使用
@provide('Bar')
class NewBar extends Bar {
  public bar() { return super.bar() + ',world';  };
}

const container = new Container();
bind(container);
container.get('Bar').bar() // hello,world
```

### 循环依赖

如下逻辑，两个类相互依赖，实例化时将会报错。

1. 实例化 Bar，依赖 Foo，需要获取 Foo 的实例，此时 Bar 并没有实例化完成。
2. 然后实例化 Foo，依赖 Bar，需要获取 Bar 的实例，此时 Foo 也没有实例化完成。
3. 于是需要实例化 Bar，这里会报错 Cycle Dependency。

```typescript
import { Container, injectable, inject } from 'inversify';
import { provide } from 'my-provide'

@provide('Foo')
class Foo {
  private _bar: Bar;

  public constructor(@inject('Bar') bar: Bar) {
    this._bar = bar;
  }

  public foo() {
    return 'hello';
  }

  public log() {
    console.log(this._bar);
  }
}

@provide('Bar')
class Bar {
  private _foo: Foo;

  public constructor(@inject('Foo') foo: Foo) {
    this._foo = foo;
  }

  public bar() { return this._foo.foo(); };
}
```

假设我们把实例化后移，不在 constructor 内执行依赖类的实例化，而在使用的时候才实例化，依次解决循环依赖的问题。

1. 把 `Foo` 内的 `@inject('Bar')` 去掉，增加 `@lazyInject('Bar') _bar`。
2. 仅仅调用 log 方法的时候，才会去获取 Bar 的实例。

因此我们可以实现一个 `lazyInject` 的装饰器，使用 Object.defineProperty 定义 get 方法，仅仅调用 `this._bar` 的时候才会获取 Bar 的实例。

```typescript
import { Container, injectable, inject } from 'inversify';
import { provide } from 'my-provide'

function lazyInject(name: string) {
  return function(target: any, key) {
    Object.defineProperty(target, key, {
      configurable: true,
      enumerable: true,
      get(this: object) {
        // 用户才会生成 container，这里怎么获取到 container 呢？
        return container.get(name);
      },
    });
  };
}

@provide('Foo')
class Foo {
  @lazyInject('Bar') _bar!;

  public foo() {
    return 'hello';
  }

  public log() {
    console.log(this._bar);
  }
}

@provide('Bar')
class Bar {
  private _foo: Foo;

  public constructor(@inject('Foo') foo: Foo) {
    this._foo = foo;
  }

  public bar() { return this._foo.foo(); };
}
```

上面遗留一个取不到 `container` 的问题，`lazyInject` 取不到 `container`，但是 `bind` 可以取到 `container`，可以做出如下修改：

1. `lazyInject` 不再直接使用 `Object.defineProperty` 定义 `get` 方法，将注入的信息存在 `target`（类的实例） 上。
2. 在 `bind` 方法中取出 `target` 上存储的 `lazyInject` 的信息，循环使用 `Object.defineProperty` 定义 `get` 方法，这里便可以获取到 `container`。

```typescript
import { Container, injectable, inject, interfaces } from 'inversify';

const state  = {};

function provide(name: string) {
  return function(target: any) {
    state[name] = target
  };
}

function bind(container) {
  Object.keys(state).map(name => {
    container.bind(name).to(state[name]).onActivation((context: interfaces.Context, implementation: unknown) => {
      Object.keys(implementation.lazeInjectState).forEach((key) => {
        Object.defineProperty(target, key, {
          configurable: true,
          enumerable: true,
          get(this: object) {
            return context.container.get(name);
          },
        });
      });
    });
  });
  state = {}
}

function lazyInject(name: string) {
  return function(target: any, key: string) {
    target.lazeInjectState = {};
    target.lazeInjectState[key] = name
  };
}

@provide('Foo')
class Foo {
  @lazyInject('Bar') bar!;

  public foo() {
    return 'hello';
  }
}

@provide('Bar')
class Bar {
  private _foo: Foo;

  public constructor(@inject('Foo') foo: Foo) {
    this._foo = foo;
  }

  public bar() { return this._foo.foo(); };
}
```

### 接口设计

1. Versea 在接口设计上，应该增加增加命名空间，避免多个 `Container` 冲突的情况。
2. 依赖可以放到 `ContainerModule`，而不是 `Container`。

```typescript
import 'reflect-metadata';
import { injectable } from 'inversify';

interface CreateProviderReturnType {
  provide: (
    serviceIdentifier: interfaces.ServiceIdentifier,
    bindingType?: 'Constructor' | 'Instance',
  ) => (target: any) => any;
  provideValue: <T = unknown>(
    target: any,
    serviceIdentifier: interfaces.ServiceIdentifier,
    bindingType?: 'ConstantValue' | 'DynamicValue' | 'Function' | 'Provider',
    replace?: (previous: T, current: T) => T,
  ) => any;
  buildProviderModule: (container: interfaces.Container) => interfaces.ContainerModule;
}

export const createProvider: (MetaDataKey: string) => CreateProviderReturnType
```

`createProvider` 是创建一个具有命名空间的依赖注入和自动绑定能力的对象。

- `provide` 装饰器可以自动绑定类
- `provideValue` 函数是自动绑定常量，函数等。
- `buildProviderModule` 执行会返回一个 `ContainerModule`。

使用如下：

```typescript
const { provide, provideValue, buildProviderModule } = createProvider('metaKey');

@provide('test')
class Test {}

provideValue('bar', 'barKey');

const container = new Container({ defaultScope: 'Singleton' });
container.load(buildProviderModule(container));
expect(container.get('test')).toBeInstanceOf(Test);
expect(container.get('barKey')).toBe('bar');
```

lazyInject 接口设计：

```typescript
export const lazyInject: (serviceIdentifier: interfaces.ServiceIdentifier) => (target: unknown, propertyKey: string) => void
```
