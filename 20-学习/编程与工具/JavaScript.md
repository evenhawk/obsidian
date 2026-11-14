---
title: JavaScript 基础
tags:
  - javascript
  - 前端
  - 基础
status: draft
updated: 2026-10-06
---

# JavaScript 基础

## 定义变量

### var
ES5旧声明方式
函数作用域,全局有效
### let
声明一个变量
块级作用域`if/for/while/{}`内部有效
### const
`const` 用来声明一个块级作用域变量
`const` 声明的变量绑定不能被重新赋值，但如果它引用的是对象或数组，对象内部内容仍然可以修改
块级作用域`if/for/while/{}`内部有效

## console.log()

`console.log()` 用于在控制台输出信息或值
相当于 Python 的 `print` 和 C 的 `printf`

## typeof()

用于判断值的类型，常用写法是 `typeof 值`；写成 `typeof(值)` 结果相同

## 强制类型转换

### 隐式转换

在`JavaScript`中允许字符串string和数字number之间的直接运算
但是
如果是 `+` 运算
只要有一个操作数是字符串，通常就会执行字符串拼接
```JavaScript
console.log("20"+"10"+10);
// 输出为
201010
```
如果是-减法运算
会把数字字符串自动转为数字类型
```javascript
console.log("100"-"20"-10);
// 输出为
70
```
还有`*`乘法和`/`除法
它们和减法一样,会自动转换

*注意：如果字符串无法转换为数字，减法、乘法和除法通常会得到 `NaN`，而不是抛出所有类型的错误。*

### 强制转换

#### 转为数字`Number()`

下面是处理各种类型的时候的输出
```javascript
Number('123')     // 123
Number('12a3')    // NaN 是数字类型但是不是数字
Number('')        // 0
Number(null)      // 0
Number(undefined) // NaN
Number(true)      // 1
Number([])        // 0
Number([123])     // 123
Number([1,2])     // NaN
```
##### 补充：parseInt /parseFloat

- `parseInt(字符串,进制)`：从左提取整数，遇到非数字停止

```
parseInt('123abc') // 123
parseInt('abc123') // NaN
```

- `parseFloat`：提取浮点数

#### 转为字符串`String()`

所有的类型可以稳定转为字符串类型
```JavaScript
String(123)      // "123"
String(null)     // "null"
String(undefined)// "undefined"
String(true)     // "true"
String([1,2])    // "1,2"
String({})       // "[object Object]"
```
如果要明确把值转换成字符串，可以使用 `String(value)`；如果只是拼接文本，模板字符串通常更清晰

#### 转为布尔 `Boolean ()`

只用6个假值,其余全为真
`0、NaN、''、null、undefined、false`

```JavaScript
Boolean(0)        // false
Boolean('')       // false
Boolean(NaN)      // false
Boolean(null)     // false
Boolean(undefined)// false
Boolean([])       // true
Boolean({})       // true
Boolean(-1)       // true
```
快捷写法`!!变量`

## 使用反引号作为模板字符串

在输出到控制台的过程中
`console.log("string1"+变量+"string2")`
输出一个句子需要不断拼接字符串和变量

使用反引号可以快速解决
```JavaScript
console.log("string1"+变量+"string2")
console.log(`string1${变量}string2`)
```
上面两种所达成的效果一致,使用模板字符串可以少打几个引号

## JavaScript的 if 语句

JavaScript的 if 语句和C语言较为类似

```JavaScript
if(条件){
执行语句
}else{
执行语句
}
```

## JavaScript 的布尔变量

在JavaScript中布尔变量为`true`和`false`并且都是小写
首字母大写的不作为布尔变量.

## JavaScript 中的 `==` 和 `===`

`==` 表示宽松相等，在 JavaScript 中 `"100" == 100` 的值为 `true`。
一个字符串类型的数字和一个数字类型相等 
因为两个=是表示不严格相等,会自动进行类型转换.

而 `===` 表示严格相等，上述代码的值为 `false`。
它不会进行隐式类型转换：类型不同时直接返回 `false`，只有类型和值都相同才返回 `true`。
建议在编写代码时使用三个=号,养成良好习惯

## 函数

函数:重复使用的多条语句
```JavaScript
function 函数名(参数)
{
函数体;
return 返回值;
}
```
