---
title: "TypeScript项目实践——在使用TypeScript中，如何减少类型的重复定义？"
date: 2022-04-06T15:11:31+08:00
draft: false
description: ""
author: "前端团队"
categories: ["前端"]
tags: ["前端","TypeScript"]
---

# TypeScript 项目实践

该系列文章主要是总结TypeScript在项目中实践经验，遇到什么样的问题，针对具体问题的解决方案，目的是给希望使用或正在使用TypeScript的同行提供参考，高效正确的使用TypeScript。  

## 在使用TypeScript中，如何减少类型的重复定义？

后台管理系统，每个模块大致就是列表、详情、表单和一些操作。在定义Typescript的时候，往往会重复定义一些类似的结构。如何减少类型的重复定义呢？

### 模型已经完整定义，使用部分字段作为的新的类型

在定义列表的展示信息和详情的展示信息的时候，已经定义了一个比较完善的类。之后再去处理创建、编辑之类的操作可能只需要其中几个参数。

```typescript

// 用户列表的信息
class Uesr {
  id: string;
  name: string;
  email: string;
  createTime: string;
}
```
例如：用户列表展示需要显示的信息比较完整，但是添加用户的时候可能只需要name、email。如果我们能把这两个属性摘出来就行了。内置的 Pick 类型，就可以完美解决我们的问题。

```typescript
// 添加用户
function create(formValue: Pick<User, 'name' | 'email'>) {
  // todo:
}

// 为什么用 pick？我们看下Pick的实现原理
// 意思是选取指定一组属性，并组成一个新的类型
type Pick<T, K extends keyof T> = {
    [P in K]: T[P];
};
```
或者，当我们创建的时候，表单中除了 User.id，其他都是必填项。但是重新定义一个新的不包含id的类型的话，又显得有些重复了。但是用 Pick 的话，需要写的 keys 又太多了。这个时候就需要用内置 Omit 来处理了。

```typescript
// 添加用户
function create(formValue: Omit<User, 'id'>) {
  // todo:
}

// 我们来看为什么 Omit 能满足我们的需求
// Omit的原理就是从类型 T 中剔除 K 中的所有属性。
// 此处的 keyof any 而不是 keyof T, 我也是有点不明白，就去查了一下。简单来说就是社区选择的结果.
// 不明白的可以看 https://github.com/Microsoft/TypeScript/issues/30825
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;

// 这里牵扯到 Exclude 类型，这里就先介绍下吧。
// 表示提取存在于 T，但不存在于 U 的类型组成的联合类型。
type Extract<T, U> = T extends U ? T : never;
```
注意：此处 keyof any 得到的是 string | nuber | symbol ， 因为类型 key 的类型只能为 string | nuber | symbol 。

### 使用方法的参数类型，作为函数的返回值

某些时候我们在某个函数上定义的参数类型，可能是跟某些参数或者某些值的类型是相同的，这个时候我们就可以直接提取此函数的参数类型。具体实现就是使用 Parameters 类型。

例如，我们我们定义了一个格式化返回数据的函数，函数中我们定义了参数类型。然后我们的请求接口返回的数据与此相同。这个时候就能直接使用格式化函数的参数类型。

```typescript
// 格式化返回数据的函数
function formatData(data: {
  id: string;
  name: string;
  e_mail: string;
  create_time: string;
}) { 
  // ... 
}

// 获取列表信息
function getPersonLists(): Parameters<typeof formatData>[0][] {
  // ...
}
```

我们通过下图可以看出， T0 此时的类型就是 formatData 中参数 data 的类型。
![](/images/typescript/047a512a-57fa-48df-8a8c-946908bdec40.png)
[在线示例](https://www.typescriptlang.org/play?#code/FAMwrgdgxgLglgewgAhAgTgWwIYwCK7YAUAJoQFzIDewyycJlAzjOnBAOYDctyE2mAKbNW7br0EB9HHAA2Itpx50o6QbinwhCsTwC+ASmrJedAPRnkAOhsm9wYDACeAB0HIAKgAZkAXmQACtjoAoIwguhMADzObgggqBg4+IQAfADaXgC6wEA)

### 将已存在的类型，组合成新类型
在项目中，有些是需要集合其他几个类型或者类型中的部分参数而成的新类型。

```typescript
// 函数中的传参可能是需要多个类型中的参数，并且有些是可选项。
function create(options: Pick<User, 'name' | 'email'>
  & Partial<Pick<Department, 'department' | 'project'>>
  & Pick<UserRole, 'role' | 'type'>
) {
  // ...
}

// Partial 表示
```

代码中的 create 参数，需要 User 类型中的 name、email，需要 Department 中的 department、project，也需要 UserRole 中的 role、type。我们用 & 和 内置类型组合成一个新的类型。



### 总结

在写 Typescript 中，合理使用内置的类型，可以有效的减少重复定义类型，减少无意义的代码。当然，内置类型肯定是不限于我们所介绍的这些以及使用场景。并且我们当内置类型满足不了我们的需求的时候，我们也可以自定义类型。