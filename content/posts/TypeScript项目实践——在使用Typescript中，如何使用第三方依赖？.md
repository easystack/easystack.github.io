---
title: "TypeScript项目实践——在使用Typescript中，如何使用第三方依赖？"
date: 2022-04-06T15:40:35+08:00
draft: false
description: ""
author: "前端团队"
categories: ["前端"]
tags: ["前端","TypeScript"]
---

# TypeScript 项目实践

该系列文章主要是总结TypeScript在项目中实践经验，遇到什么样的问题，针对具体问题的解决方案，目的是给希望使用或正在使用TypeScript的同行提供参考，高效正确的使用TypeScript。  

## 在使用Typescript中，如何使用第三方依赖？

在实际使用Typescript项目开发过程中， 我们经常会引入第三方依赖包，且使用npm来管理第三方依赖包的安装管理， 如下：

```bash
// 以 ng-zorro-antd 为例， npm 安装
$ npm install ng-zorro-antd --save
```
```typescript
// 项目中直接 import 导入即可
import { NzModalModule } from 'ng-zorro-antd/modal';
```
然而是否所有的包都可以通过上述方式直接 import 导入？ 当然不行。

ng-zorro-antd 是基于 TypeScript 开发，包含完整的声明文件。比如我们安装 ng-zorro-antd 后可以发现 node_modules/ ng-zorro-antd中包含 ng-zorro-antd.d.ts 或者package.json 中包含 typings 属性指定了声明文件。
![](/images/typescript/1e267ba5-e9a0-454c-bdf2-69e9aab09c0e.png)

但实际上有很多包并不是基于TypeScript开发的，自然也不会导出Typescript声明文件。由于历史原因，仍然有很多基于javascript开发的第三方依赖，比如jquery。或者即使有些包基于TypeScript开发的， 但如果没有导出声明文件， 也是没用的，直接导入的话就会编译报错， 告诉我们没有找到 相关模块或者声明。

```typescript
// 以jquery为例
npm install jquery --save

// 项目代码
import * as $ from 'jquery';

// Could not find a declaration file for module 'jquery'. '/root/my-app/node_modules/jquery/dist/jquery.js' implicitly has an 'any' type.
// Try `npm i --save-dev @types/jquery` if it exists or add a new declaration (.d.ts) file containing `declare module 'jquery';`
```

我们会发现， 即使已经安装了jQuery， 依然会编译报错，因为编译器查找到定位到 jquery module 的声明文件。 那么Typescript 模块依赖声明文件又是如何查找的？

### Typescript 如何查找第三方依赖声明？

TypeScript是模仿Node.js运行时的解析策略来在编译阶段定位模块定义文件， 查找依赖声明文件，按照上层递归 node_modules 查找，具体来说就是：

* node_modules/jquery/jquery.d.ts
* node_modules/jquery/package.json types/typings属性定义
* 如果找不到，则会去 node_modules 中的@types（默认情况，目录可以修改）目录下去寻找对应包名的模块声明文件
* 向上层 node_modules 目录继续查找

如果都没有查找到， 则编译提示报错，针对安装的依赖包中缺少声明文件情况，社区提供两种解决方案：

* 手动下载 jquery 的声明文件（优先推荐使用， 需要注意版本问题）
* Typescript 2以前也会使用 typings search 查找依赖声明 （不常用）
* 自行定义一份*.d.ts 声明文件，并将 jquery 声明为 module

### 下载第三方依赖包@types声明

至于自定义 .d.ts 文件 declare module，其实非常简单，直接在src文件夹下定义 typings/jquery/jquery.d.ts 文件。

然后在jquery.d.ts文件中编写声明：

```typescript
declare module 'jquery' {
  // export function init(): void
  // 亦或者扩展模块声明， 比如我们在业务代码中 jquery 新增了一个方法
  // export function func(): number
}

// 或者可直接简写
declare module 'jquery';
``` 

修改 tsconfig.json 配置文件，typeRoots 属性中新增一个添加 typings 目录，这样如果node_modules/@types 中找不到声明文件，就会去src/typings中查找。

```json
{
   "typesRoot": [
      "node_modules/@types",
      "src/typings"
    ]
}
```
然后大工告成。