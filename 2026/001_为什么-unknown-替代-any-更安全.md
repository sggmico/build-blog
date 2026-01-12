# 为什么 unknown 替代 any 更安全？

Jan 12, 2026

> **关键词：** `TypeScript` · `unknown` · `any` · `类型安全`

---

把 any 改成 unknown，代码会安全很多。

unknown 是 TypeScript 专门设计的"类型安全版 any"，用来解决 any 的"类型失控"问题。

核心区别在哪？**对类型的信任程度不同**。any 是"我啥都信"，unknown 是"我啥都不信，先验证再说"。

## any 哪里不安全？

any 会关闭 TypeScript 的类型检查。

它让变量变成一个"类型黑洞"：你可以随便访问属性、调用方法、赋值给任意类型。TypeScript 都不会报错，但运行时可能直接崩。

看个实际例子：

```typescript
// 接口返回的数据，用 any 接收
const data: any = fetchSomeData();

// 1. 随便访问属性：TS 不报错
console.log(data.name);
// 运行时如果 data 不是对象：Cannot read property 'name' of undefined

// 2. 随便调用方法：TS 不报错
data();
// 运行时如果 data 不是函数：data() is not a function

// 3. 随便赋值：TS 不报错
const num: number = data;
// 如果 data 是字符串，num 就被污染了
```

any 的问题就是：**TypeScript 放弃了所有类型校验，把风险完全丢给运行时**。

这违背了 TypeScript "静态类型检查" 的初衷。相当于写 TypeScript 却不用类型系统。

我之前就踩过坑：接口返回的数据用 any 接收，后端改了字段名，TS 没报错，上线后才发现页面白屏。

## unknown 为什么更安全？

unknown 的设计思路：**我知道值存在，但不知道类型是啥。在明确类型之前，不允许做任何危险操作**。

它是"类型安全的 any"，强制你先验证类型，再使用。

同样的场景，用 unknown 改写：

```typescript
// 用 unknown 接收未知值
const data: unknown = fetchSomeData();

// 1. 直接访问属性：TS 报错！
console.log(data.name); // ❌ Object is of type 'unknown'

// 2. 直接调用方法：TS 报错！
data(); // ❌ Object is of type 'unknown'

// 3. 直接赋值：TS 报错！
const num: number = data; // ❌ Type 'unknown' is not assignable to type 'number'
```

所有危险操作都被拦截了。

### 怎么用 unknown？

必须通过**类型守卫**或**类型断言**明确类型，TS 才允许后续操作。

这就把"运行时风险"提前到了"编译时检查"。

修正后的安全写法：

```typescript
const data: unknown = fetchSomeData();

// 1. 用类型守卫缩小类型（推荐）
if (typeof data === 'object' && data !== null) {
  // 确认是对象（排除 null）
  if ('name' in data) {
    console.log(data.name); // ✅ 安全
  }
}

// 2. 确认是函数
if (typeof data === 'function') {
  data(); // ✅ 安全
}

// 3. 确认是数字
if (typeof data === 'number') {
  const num: number = data; // ✅ 安全
}

// 4. 类型断言（明确知道类型时才用）
const str = data as string;
console.log(str.length); // ⚠️ 断言错误会运行时崩溃，需谨慎
```

类型守卫的方式更安全。我一般只在非常确定的情况下才用类型断言。

## 核心对比

| 特性 | any | unknown |
|:--|:--|:--|
| 类型检查 | 完全关闭 | 完全开启 |
| 直接访问属性/方法 | 允许（无报错） | 禁止（编译报错） |
| 赋值给其他类型 | 允许（可能污染） | 禁止（需先缩小类型） |
| 安全性 | 极不安全（运行时风险高） | 相对安全（编译时拦截） |
| 使用场景 | 临时兼容 JS 代码（尽量避免） | 接收未知类型值（接口返回、用户输入） |

## 总结

unknown 替代 any 更安全，原因很简单：**从"放弃类型检查"变成"强制类型检查"**。

unknown 逼你在使用变量前，先明确它的类型。通过类型守卫或断言，把可能发生在运行时的错误，提前拦截在编译时。这才是 TypeScript 作为静态类型语言的核心价值。

实际开发中，除了临时兼容老 JS 代码，应该尽量用 unknown 替代 any。这是提升代码类型安全性的关键实践之一。

我现在的习惯：看到 any 就改成 unknown，然后补充类型守卫。多写几次就成肌肉记忆了。
