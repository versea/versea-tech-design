# IOP（面向接口编程）

我们对 IOP 并不陌生，IOP 的优势和缺点不再赘述。

Versea 使用 IOP 的原因不是多态，而是借助 typescript 的 interface merge 扩展类型定义。

### 简介

Versea 具有[实体类的组合扩展能力](./extensible-entity.md)，那么解决类型定义问题就要靠 interface merge 了。举例说明：

```typescript
// core
export interface ITest {
  test: () => string;
}
export const ITest = symbol.for('ITest');

export interface TestOptions {}

export class Test extends ExtensibleEntity implements ITest {
  constructor(options: TestOptions) {
    super(options);
  }

  public test() {
    return 'hello, world';
  }
}

// plugin
declare module 'core' {
  interface ITest {
    bar?: string
  }

  interface TestOptions: {
    bar?: string
  }
}

Test.defineProp('bar', { default: 'bar' });

const test = new Test({ bar: 'newBar' });
test.bar // 类型正确
```
