---
title: TypeScript infer
date: 2021-12-23 17:33:00
---

## 前言

[TypeScript](https://www.typescriptlang.org/) 是微软开发的一个开源的编程语言，通过在 JavaScript 的基础上添加静态类型定义构建而成。它是 JavaScript 的超集，不仅支持原生JS的写法，还增加了很多有用、强大的功能。众所周知， JavaScript 是弱类型语言，一些很低级却难以发现的 Bug 只有在运行时才能发现，而 TypeScript 却能够在编写代码的过程中就能避免绝大部分的低级报错。本专栏用于记录和分享我在使用 TypeScript 中的感悟和心得，以下来介绍一下 `infer` 的用法：

## infer 关键字

> 在有条件类型的 `extends` 子语句中，允许出现 `infer` 声明，它会引入一个待推断的类型变量。 这个推断的类型变量可以在有条件类型的 true 分支中被引用。

这就是 `infer` 的解释，是不是上来一看就有点懵。晦涩的描述从字面意思来看确实很难懂，通过分解知识点以及结合一些例子后就很好理解了。首先看一下什么是条件类型：

## 条件类型

像这样 `T extends U ? X : Y` 的表达式就是条件类型，结合以下代码：

```typescript
type ConditionType<T> = T extends Boolean ? number : undefined

const a: boolean = false
const b: string = 'false'

type TypeA = typeof a // boolean
type TypeB = typeof b // string

type T1 = ConditionType<TypeA> // number
type T2 = ConditionType<TypeB> // undefined
```

`ConditionType` 是我定义的一个简单的工具类型，它可以接收一个 **泛型参数 T**，并分别返回不同的类型。以上第一行代码可以理解为：如果 `T` 是布尔类型则返回 `number` 类型，不是则返回 `undefined` 类型。甚至，我们还可以约束传入的 **泛型参数 T** 的类型，让我们的代码更严谨一点，如下：

```typescript
type Types = boolean | string
type ConditionType<T extends Types> = T extends Boolean ? number : undefined

type TypeC = object
type T3 = ConditionType<TypeC> // Error: Type 'object' does not satisfy the constraint 'Types'. 
```

通过对泛型参数类型的约束，传入其他类型会报错。

## ReturnType（获取函数返回值类型）

结合 Typescript 内置工具类型 `ReturnType` 来理解 “使用 `infer` 关键字来声明一个待推断的类型变量”。`ReturnType` 可以获取函数返回值类型，它的核心也是 `infer` 来实现的，代码如下：

```typescript
/**
 * Obtain the return type of a function type
 */
type ReturnType<T extends (...args: any) => any> = T extends (...args: any) => infer R ? R : any;
```

它的用法非常简单，如下：

```typescript
const add = (x: number, y: number) => x + y

type Type1 = typeof add // (x: number, y: number) => number
type T1 = ReturnType<Type1> // number
```

`T extends (...args: any) => any` 这一行代码就是约束传入 `ReturnType` 的泛型参数类型必须是函数类型，这就类似于参数校验。这里的 `T` 就是 `add` 函数的类型 `(x: number, y: number) => number` 。

那么 `T extends (...args: any) => infer R ? R : any` 代码的含义可以理解为：如果 T *(x: number, y: number) => number* 满足于 (...args: any) => infer R 的表现形式的话，就返回 R，否则返回 `any` 。

`infer R` 表示声明一个待推断的类型，并将推断出来的类型存储在 R 中。这里 T 是 `(x: number, y: number) => number` ，`infer R` 被放在 `(...args: any) => infer R` 函数类型的返回值处，因此推断出了 `number` 类型并赋值给 R 。

## infer 推断函数参数类型

`ReturnType` 借助 `infer` 实现获取函数的返回值类型，那么如何获取函数参数类型？虽然 TypeScript 已实现该工具函数，但不妨碍我们写一个。很简单，在 `ReturnType`  上改一下即可：

```typescript
type ParamsType<T extends (...args: any) => any> = T extends (...args: infer R) => any ? R : any;
```

仅是将 `infer R` 跟参数类型 `any` 调换一下就实现了，看下打印：

```typescript
type ParamsType<T extends (...args: any) => any> = T extends (...args: infer R) => any ? R : any;

const add = (a: number, b: string) => a + b


type TypeAdd = typeof add // (a: number, b: string) => string

type T1 = ParamsType<TypeAdd> // [number, string]

const arr: T1 = [1, 2] // Error: Type 'number' is not assignable to type 'string'.
```

由于上述采用的是 ES6 rest 参数写法，所以它会推断出一个元组类型，如果是想获取单个参数类型可以这样写：

```typescript
type ParamType<T extends (arg: any) => any> = T extends (arg: infer R) => any ? R : any;
```

## 总结

以上便是我对于 `infer` 的粗浅理解，其实 `infer` 很强大能实现更多的功能。比如：将类构造函数类型的参数推断为元组的 `ConstructorParameters` 、获取构造函数类型的返回类型的 `InstanceType` ，还能通过 `infer` 推断出数组的子元素类型等等一系列骚操作……

并且 Vue3 中的 `ref` 为什么能够实现各种复杂的类型推断，也是通过 `infer` 来实现的，写的有点复杂，感兴趣的童鞋可以看一下 [Vue3 ref 源码](https://github.com/vuejs/vue-next/blob/master/packages/reactivity/src/ref.ts)  。

## 参考

- https://www.tslang.cn/docs/release-notes/typescript-2.8.html
- https://www.wenjiangs.com/doc/typescript-infer

