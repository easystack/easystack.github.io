---
title: "TypeScript项目实践——在使用Typescript中，如何减少any使用？"
date: 2022-04-06T14:58:47+08:00
draft: false
description: ""
author: "前端团队"
categories: ["前端"]
tags: ["前端","TypeScript"]
---

# TypeScript 项目实践

该系列文章主要是总结TypeScript在项目中实践经验，遇到什么样的问题，针对具体问题的解决方案，目的是给希望使用或正在使用TypeScript的同行提供参考，高效正确的使用TypeScript。  

## 在使用Typescript中，如何减少any使用？
本文主要是总结项目中实践经验，在Typescript中减少any的使用。

### 联合类型代替any
当一个对象需要指定多个类型时，使用了any作为类型，为了减少any的使用，我们可以用联合类型(|)组成多类型来代替any。
例如：定义一个格式化数字类型和日期类型formatTime方法，方法的入参类型就可以使用联合类型(number | Date)

```typescript
// bad
formatTime(time: any) {
    if (typeof time === 'number') {
      time = new Date(time * 1000);
    }
    const year = time.getFullYear();
    const month = time.getMonth() + 1;
    const day = time.getDate();
    const hours = time.getHours();
    const minu = time.getMinutes();
    const second = time.getSeconds();
    return `${year}-${month}-${day} ${hours}:${minu}:${second}`;
}

// good 使用联合类型代替any
function formatTime(time: number | Date) {
    ...
}

// 传入Date类型参数
formatTime(new Date());
// 传入number类型参数
formatTime(1600000000);
```

### & 集合生成一个新类型代替any
当一个对象需要指定的类型缺失属性时，使用了any作为类型，为了减少any的使用我们可以用集合(&)组成新类型来代替any。
例如：已有一个createInstance方法，需定义一个迁移方法migrationInstance，入参类型与createInstance相似，只是多一个oldID属性，我们可以用集合(&)组成含有oldID的新类型。

```typescript
export class ApiInstanceService {

  // 定义createInstance
  createInstance(params: {
    name: string;
    cpu: number;
    member: number;
    ...
  }) {
  ...
  }
  
  // bad
  migrationInstance (params: any) {
  ...
 }

  // good 将createInstance的参数类型通过集合方式生成包含oldID的新类型
  migrationInstance(params: Parameters<ApiInstanceService['createInstance']>[0] & { oldID: string }) {
  ...
  }
}
```

### ReturnType获取函数返回值的类型
当一个对象类型为函数返回值的类型，并且无法定位到这个类型时，使用了any作为类型，为了减少any的使用。
例如：调用了setInterval方法，当前无法确定返回值的类型，使用ReturnType获取返回值类型。

```typescript
// bad
let interval: any;

// good
let interval!: ReturnType<typeof setInterval>;

interval = setInterval(() => {
...
}, 3000);
```

### as指定类型
当一个对象中的属性在当前类型中匹配不到时，使用了any作为类型，为了减少any的使用
我们用as指定类型来代替any。（注意：指定的类型必须是正确的类型）
例如：定义searchChange方法，为了获取e.target.value，将参数e.target通过as指定
HTMLInputElement类型获取value。

```typescript
// 触发input调用searchChange方法
<input nz-input (input)="searchChange($event)" />

// bad
searchChange(e: any) {
    const value = e.target.value;
    .....
}

// good e.target通过as指定HTMLInputElement类型获取value
searchChange(e: Event) {
    const value = (e.target as HTMLInputElement).value;
    .....
}
```

### 可索引接口代替any
当一个对象中的属性存在不确定性，无法进行固定约束，使用了any作为类型，为了减少any的使用,我们可以用索引接口代替any.
例如：TemplateOptions中的formlyItemStyle属性用于指定样式数据，例如：{marginBottom: '0'} formlyItemStyle属性无法固定约束类型，我们使用索引接口代替any。

``` typescript
// TemplateOptions中的formlyItemStyle属性用于指定样式，例如：{marginBottom: '0'}

// bad 
interface TemplateOptions {
    //...
    formlyItemStyle?: any;
}

// 索引接口代替any
interface TemplateOptions {
    ...
    formlyItemStyle?: { [key: string]: string };
}
```

### 总结
代码中应减少any使用，可增加代码的可读性和可维护性，频繁使用 any 不利于项目的后续维护，
会出现本可避免的bug。