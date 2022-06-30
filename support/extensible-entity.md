# ExtensibleEntity

ExtensibleEntity 是一个可扩展的实体类，通常情况一般作为基类使用。

### 作用

动态的给类的实例增加字段。

假设没有 ExtensibleEntity，我们增加字段可能使用继承。举个例子：

```javascript
class Test {
  constructor(options) {
    this.foo = options.foo;
  }
}

class Test2 extends Test {
  constructor(options) {
    super(options);
    this.bar = options.bar;
  }
}

class Test3 extends Test {
  constructor(options) {
    super(options);
    this.zoo = options.zoo;
  }
}
```

我们发现 `Test2` 没有 `zoo` 属性，`Test3` 没有 `bar` 属性，如果需要实现同时具有 `zoo` 属性和 `bar` 属性，我们需要使用 `class Test3 extends Test2`，这可能并不是我们想要的结果（Test2 可能是一个插件实现，Test3 是另一个插件实现，插件一和插件二没有关系，但是因为一句继承产生了关系）。使用组合扩展的方式替换继承扩展方式。

```javascript
// core
class Test extends ExtensibleEntity {}

// plugin 1
Test.defineProp('bar', { default: 'bar' });

// plugin 2
Test.defineProp('zoo');

// 使用者
const test = new Test({ zoo: 'newZoo' })
console.log(test.bar) // bar
console.log(test.zoo) // newZoo
```

继承 ExtensibleEntity 之后，Test 便有一个静态方法 `defineProp`，调用这个方法，可以在 `new Test` 的时候增加属性。

### 为什么需要这种设计

看上面代码，其实并不需要那种设计，我们可以在实例化之后再给对象赋值，而不是 `new Test` 的时候增加属性，感觉如下代码更简单：

```javascript
// core
const callbacks = []

class Test {}

export const pushCallBack = (cb) => callbacks.push(cb)

export const createTest = (options) => {
  const test = new Test(options);
  callbacks.forEach(cb => cb(test, options))
  return test;
}

// plugin 1
pushCallBack((test, options) => {
  // 实例化之后再给对象赋值
  test.bar = options.bar
})

// plugin 2
pushCallBack((test, options) => {
  console.log(test.bar) // bar
})

// 使用者
const test = createTest({ bar: 'bar' });
```

我们换个场景，加个 `unshiftCallBack` 的方法：

```javascript
// core
const callbacks = []

class Test {}

export const pushCallBack = (cb) => callbacks.push(cb)
export const unshiftCallBack = (cb) => callbacks.unshift(cb)

export const createTest = (options) => {
  const test = new Test(options);
  callbacks.forEach(cb => cb(test, options))
  return test;
}

// plugin 1
pushCallBack((test, options) => {
  test.bar = options.bar
})

// plugin 2
unshiftCallBack((test, options) => {
  console.log(test.bar) // undefined
})

// 使用者
const test = createTest({ bar: 'bar' });
```

我们看到 plugin 2 的 callback 中打印的是 `undefined`。为了能让 plugin2 获取正确的值，应该在 `new Test` 的时候增加属性，而不能后面在插件 `callback` 中增加。

```javascript
// core
const callbacks = []

class Test extends ExtensibleEntity {}

export const pushCallBack = (cb) => callbacks.push(cb)
export const unshiftCallBack = (cb) => callbacks.unshift(cb)

export const createTest = (options) => {
  const test = new Test(options);
  callbacks.forEach(cb => cb(test, options))
  return test;
}

// plugin 1
Test.defineProp('bar', { default: 'bar' });

// plugin 2
unshiftCallBack((test, options) => {
  console.log(test.bar) // bar
})

// 使用者
const test = createTest({ bar: 'bar' })
```

### 接口设计

```typescript
export interface ExtensiblePropDescription {
  /**
   * 扩展属性对应的 option 的 key
   * @description A.defineProp('_a', { optionKey: 'a' }) A 的实例会增加 _a，取值会取 New A({ a: 'foo' }) 的 a 参数
   */
  optionKey?: string;
  /** 是否必传字段 */
  required?: boolean;
  /** 默认值 */
  default?: unknown;
  /** 验证函数 */
  validator?: (value: unknown, options: Record<string, any>) => boolean;
  /** 格式化函数 */
  format?: (value: unknown, options: Record<string, any>) => unknown;
  onMerge?: (value: unknown, otherValue: unknown) => unknown;
  onClone?: (value: unknown) => unknown;
}

export declare class ExtensibleEntity {
  [key: string]: unknown;

  static defineProp(key: string, description?: ExtensiblePropDescription): void;
}
```
