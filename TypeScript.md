# TypeScript

## 简介

TypeScript 是什么？ 以 JavaScript 为基础构建的语言， 一个 JavaScript 的超级，扩展了 JavaScript ，并添加了类型 可以在 任意支持 JavaScript 的平台中执行，单 TS 不能被JS 解析器直接执行，需要编译成 JS

## 安装

全局安转 typescript

```powershell
npm i -g typescript
```

## 运行

想运行.ts的文件 必须先编译 tsc test.ts 会得到 test.js 然后再运行test.js
编译 生成 js 文件 执行命令： tsc xxx.ts (tsc 是 ts 的编译器) ， 执行 node xxx.js

```powershell
tsc test.ts

node test.js
```

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/c15eeb4dcfe29e8b6c6ee1ea552b095c.png)

## 使用

### 类型声明

```typescript
// a 的类型设置为了 number, 在以后的使用过程中a 的值只能是数字
let a: number;
a = 10;
a = 'hello';  //报错
console.log(a);

// b 的类型设置为了 string, 在以后的使用过程中b 的值只能是字符串
let b: string;
b = 'hello';
b = 122; // 报错
console.log(b);

// 声明变量直接进行赋值
let c1: boolean = false;
// 如果变量的声明和赋值是同时进行的， TS 可以自动对变量进行类型检测
let c2 = false;
c2 = true;
​let c3: boolean = 12; // 报错
```

![image.png](http://kodo.yelingfa.top/xyblog/aurora/articles/70b43ffb2a997846b75184ed22db2650.png)
