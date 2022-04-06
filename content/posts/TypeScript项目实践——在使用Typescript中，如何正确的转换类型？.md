---
title: "TypeScript项目实践——在使用Typescript中，如何正确的转换类型？"
date: 2022-04-06T15:35:35+08:00
draft: false
description: ""
author: "前端团队"
categories: ["前端"]
tags: ["前端","TypeScript"]
---

# TypeScript 项目实践

该系列文章主要是总结TypeScript在项目中实践经验，遇到什么样的问题，针对具体问题的解决方案，目的是给希望使用或正在使用TypeScript的同行提供参考，高效正确的使用TypeScript。  

## 在使用Typescript中，如何正确的转换类型？
类型断言是用来告诉TypeScript值的详细类型, 在实际项目中合理使用可以减少我们的工作量, 下面结合项目中的一些使用场景, 概括下类型断言在实际项目中的常见用途。

### 使用断言定义空对象

例如项目中存在用户对象数据, 当定义用户对象的时候, 可以使用interface的方式

```typescript
interface IUser {
  id: number;
  name: string;
}

const user: IUser = {};
```

但是user对象一开始是个空的对象, 接口请求成功后, 才会给对象赋值, 这时候在编译时就会报错, 提示缺少id, name属性

![](/images/typescript/cc3b43c3-a842-416a-914e-d7e075d507ba.png)

这种情况就可以利用类型断言, 满足初始化是空对象, 后续再赋值

```typescript
interface IUser {
  id: number;
  name: string;
}

const user: IUser = {} as IUser;
user.id = 1;
user.name = 'admin';
或者可以使用尖括号<>语法


interface IUser {
  id: number;
  name: string;
}

const user: IUser = <IUser>{};
user.id = 1;
user.name = 'Xiao Ming';
```

### 使用双重断言指定DOM事件类型

例如在项目中经常需要处理 DOM 事件或元素

```typescript
handler(event: Event): void {
  const mouseEvent = event as MouseEvent;
}

function handler(event: Event) {
    let element = event as HTMLElement; // Error: 'Event' 和 'HTMLElement' 中的任何一个都不能赋值给另外一个
}

function handler(event: Event) {
    let element = event as unknown as HTMLElement;
}
```

在上面的例子中，若直接使用 event as HTMLElement 会报错，因为 Event 和 HTMLElement 互相不兼容。若使用双重断言，则可以打破「要使得 A 能够被断言为 B，只需要 A 兼容 B 或 B 兼容 A 即可」的限制，将任何一个类型断言为任何另一个类型。若使用了这种双重断言，那么十有八九是非常错误的，它很可能会导致运行时错误。除非迫不得已，千万别用双重断言。

### 使用常量断言as const约束对象和定义枚举对象

项目中在给接口赋值时, 如果某个接口里面的所有属性只能被读取而不能修改, 可以通过readonly


interface IUser {
  readonly id: number;
  readonly name: string;
}

const user: IUser = {
  id: 1,
  name: 'admin',
};

user.name = 'member';
成功限制name属性不允许修改
![](/images/typescript/0ecb6eb6-e14d-4029-a162-3966ce2593c9.png)

如果user对象里面属性比较多且数据结构复杂, 这时候限制每个属性都得处理, typescript支持在类型后面加上as const, 这样相当于给对象里面每个属性添加上了readonly

as const 用作枚举

```typescript
enum roleEnum {
  admin = 'Admin',
  member = 'Member',
}
const role1 = roleEnum.admin;
const roles = {
  admin: 'Admin',
  member: 'Member',
} as const
type ValueOf<T> = T[keyof T];
type RoleEnumType = ValueOf<typeof roles>;
const role2: RoleEnumType = roles.admin;
```

枚举将限制为必要的输入集，而常量字符串允许使用不属于您的逻辑的字符串。 这样可以确保在输入数据时不会因输入域中不存在的内容而出错，并且还提高了代码的可读性。

### type与类型断言
例如用户数据存在id和name两个属性, 当获取对象值的时候,

```typescript
type keys = 'id' | 'name';

const user = {
  id: 1,
  name: 'admin',
};

const getValue = (key: any) => {
  return user[key];
};
```
提示错误
![](/images/typescript/aba66a1c-98fc-41dc-b9ce-6f7944484181.png)

除了使用类型约束方式, 还可以利用类型断言解决(该方式常用于第三方库的callback, 对返回值类型没有约束的情况)

```typescript
type keys = 'id' | 'name';

const user = {
  id: 1,
  name: 'admin',
};

const getValue = (key: any) => {
  return user[key as keys];
};
```