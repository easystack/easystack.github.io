---
title: "TypeScript项目实践——在使用TypeScript中，配合VSCode可维护性、开发效率有哪些提升？"
date: 2022-04-06T14:46:52+08:00
draft: false
description: ""
author: "前端团队"
categories: ["前端"]
tags: ["前端","TypeScript"]
---

# TypeScript 项目实践

该系列文章主要是总结TypeScript在项目中实践经验，遇到什么样的问题，针对具体问题的解决方案，目的是给希望使用或正在使用TypeScript的同行提供参考，高效正确的使用TypeScript。  

## 在使用TypeScript中，配合VSCode可维护性、开发效率有哪些提升？  

 本文主要总结我们在开发过程遇到代码维护性、开发效率方面比较典型的问题，以及使用TypeScript带来的改善。

### 友好的类型提示、错误检测  

在维护一个项目经常遇到各自各样问题，但最常用的解决办法肯定有一种是：  

> console.log 输出看下参数（或者下个断点）


分析下为什么总要看参数，原因一般也就这几种：

> 代码不是我写的，搞不清楚接口参数。

> 很久之前的写的，接口参数定义忘记了。

> Javascript就这样，不下断点，能知道参数最终结构啊！（这是被坑过的。。。js自由的代价，代码不执行到当前行，永远是可变的数据结构）

例如，这样一段JavaScript代码：

``` javascript
const createModal = options => {
 //... 
}
```

在通常业务开发中很难看到清楚options参数结构描述，即使有，在多个版本迭代后也可能变了不准确，如何解决这些问题呐？在正确的使用TypeScript后，这些大部分可以得到改善，比如上面的代码使用TypeScript实现：

```TypeScript
interface Options {
  title?: string;
  content?: string;
  onOK: ()=> void;
  onCancel: ()=> void;
}

const createModal = (options: Options)=> {
 //... 
}
```

通过阅读上面的代码，可以清楚的了解这个方法的参数结构，以及是否必传、参数值类型等信息，再配合VSCode可以得到这样的使用体验：
![](/images/typescript/4651df33-64c9-4728-8e74-59cc13f9785b.gif)

而且在TypeScript中还有枚举之类特性，可以让常量定义提示更准确，修改更便利，在编码过程中变量的值、函数入参出参数、对象的成员，可以快速、清楚知道，对开发效率的提高，体验过都会离不开。

### 友好的代码调用关系树，以及IDE代码重构工具支持

``` TypeScript
// 1.公共方法
const createForm = options => {
  options.fields.forEach(field=>{
    if(field.inputType === 'text'){    
      // TODO ...
    }
    
    if(field.inputType === 'radio'){    
      // TODO: ...
    }
  })
}

createForm({ fields:[{inputType: 'text'}] });
createForm({ fields:[{inputType: 'radio'}] });


// 2.业务接口变动

function fetchUserInfo(){
  // TODO: ajax request server
  
  const responseJson = {
    username: 'test1',
    realname: 'test1_nickname'
  };
  
  return Promise.resolve(responseJson)
}

// calls 
var user = await fetchUserInfo();
var realname = user.realname;
```
像上面的示例代码中的两段代码，第一段如果要增强或者优化公共方法的options ，第二段出现需求变更导致接口数据调整，在前端代码中需要做的改动很简单但量却很大的，常规手段就是查找替换，不熟悉代码的人要改动就很难评估影响。在TypeScript中可以怎么解决这样的问题，首页要正确的声明类型，然后在使用VSCode更改代码可以这样：  

![](/images/typescript/4651df33-64c9-4728-8e74-59cc13f9785b.gif)
![](/images/typescript/bc25b3d4-f0b4-4814-936e-2f7aaab54e79.gif)

> 快速一键改名，变量、方法、

> 参数变更，快速检测相关引用通知错误提示

> 快速查找方法引用

有这些功能支持后，优化重构代码的便利可以大大提升，可以随着业务变化对代码进行一定范围的重构，便于提高代码质量。

### 针对大量功能相似但又有区别抽象能力
![](/images/typescript/dd3b9cf9-7983-409e-8cb0-2db028ca87a9.png)
![](/images/typescript/dd9133f3-d2c5-4b6b-9f02-bc9693cc233e.png)

像这样一个管理功能，增删改查很常见，在一团队里实现的人不同就有不同的设计，比如这样：

``` TypeScript
// coder a ...
class ListAComponent {
  list=[];
  query(){
    // TODO: validate filter ...
    
    // TODO: fetch data ...
    
    // TODO: update list ...
  }
}

// coder b ...
class ListBComponent {
  data=[];
  search(){
    // TODO: validate filter ...
    
    // TODO: fetch data ...
    
    // TODO: update list ...
  }
}
  
// coder c ...
class ListCComponent {
  dataSet=[];
  load(){
    // TODO: validate filter ...
    
    // TODO: fetch data ...
    
    // TODO: update list ...
  }
}

// ...
```
每个人都有自己的实践，其中的逻辑也有很多相似，这样容易造成一个结果，维护一段代码就要熟悉一种风格，在团队成员快速成长的过程中更是会不断变化着，但解决这种问题的办法很早就有了，为什么很少有人用呐，JavaScript缺少个关键字抽象以及配套的IDE支持，不过TypeScript中实现了，下面看一下用TypeScript怎么解决上面的问题以及能减少重复逻辑：

首先，我们根据上面的列表业务进行抽象定义公共逻辑，并定义抽象方法为列表不同部分预留需要实现接口；

```TypeScript 
abstract class ListComponent<Row, Filter> {
  list = new Array<Row>();
  loading = false;

  onSearch(): Promise<void> {
    // validate filter
    if (!this.checkFilter(this.filter)) {
      return Promise.resolve();
    }

    this.loading = true;

    // fetch data
    return this
      .search(this.filter)
      .then((rows) => {
        // update list
        this.list = rows;
        this.loading = false;
      })
      .catch(error => {
        // update list (error handle)
        this.list = [];
        this.loading = false;
        this.toast(error.message);
      })

  }

  toast(message: string) {
    // TODO: toast message
  };
  abstract filter: Filter;
  protected abstract checkFilter(filter: Filter): boolean;
  protected abstract search(filter: Filter): Promise<Row[]>;
}
```
然后，列表去继承功能并实现接口，像下面这样:
![](/images/typescript/bffb582b-14b9-45d0-a596-eaa43e303782.gif)

配合IDE可以快速得到我们需要做的事情，只关注实现不同的业务逻辑、定义显示的数据结构、以及显示部分，同时提升项目可维护性，类似的功能可以得到一样的接口就能更快速的理解逻辑。

### 总结
以上是我在前端代码维护中遇到问题，以及在使用正确使用TypeScript定义类型后如何解决，这里强调下要正确使用，因为TypeScript的一个特色是JavaScript类型的超集，它可以编译成纯JavaScript ,所以完全是可以在按照JavaScript的方式去用，我想这本意是让使用者快速适应，但是实际应用中会造成另一个问题没有类型，按照JavaScript的方式编码有时候会没有类型声明，会产生很多any 、unknown 类型，这个一定要在项目禁止，最次也要减少影响范围，不然上面的一切美好可能都会成为虚妄。