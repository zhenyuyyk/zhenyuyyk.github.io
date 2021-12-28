<!-- toc -->

# 前言

es6已经出来很久了，但是前几天进行代码评审的时候发现小伙伴们依然不习惯使用es6的东西，明明可以一行代码搞定的东西却写了好几行。所以想写一篇博客，整理一下日常开发中常用到的es6的东西。

> 以下语法不仅仅是es6，可能包括es6以上版本

# 关于取值

在对象中取值很常见，比如：

```javascript
let obj = {
    a: 1,
    b: 2,
    c: 3
}
```

取值使用：

```javascript
let a = obj.a;
let b = obj.b;
let c = obj.c;
```

这时候可以使用解构赋值一行代码搞定：

```javascript
let {a, b, c} = obj;
// 解构的对象不能为undefined、null。否则会报错，所以需要给一个默认值
let {a, b, c} = obj || {}
```

# 合并数组

合并两个数组或对象

```javascript
let arr1 = [1, 2, 3];
let arr2 = [3, 4, 5, 6];

let obj1 = {
    a: 1
}
let obj2 = {
    b: 2
}
```

可以这样：

```javascript
let arr = arr1.concat(arr2)

let obj = Object.assign({}, obj1, obj2)
```

这时候可以使用扩展运算符：

```javascript
let arr = [...arr1, ...arr2]
// 如果需要去重，还可以这样
let arr = [...new Set([...arr1, ...arr2])]

let obj = {...obj1, ...obj2}
```

# 关于拼接字符串

es6中有``可以用于字符串拼接，但别忘了在${}中可以放入任意的JavaScript表达式

原代码：

```javascript
const name = '小明';
const score = 59;
let result = '';
if (score > 60) {
    result = `${name}的考试成绩及格`;
} else {
    result = `${name}的考试成绩不及格`;
}
```

经过改造：

```javascript
const name = '小明';
const score = 59;
const result = `${name}${score > 60 ? '的考试成绩及格' : '的考试成绩不及格'}`;
```

# 关于if中判断条件

```javascript
if (
    type == 1 ||
    type == 2 ||
    type == 3 ||
    type == 4 ||
) {
    //...
}
```

可以使用includes：

```javascript
const condition = [1, 2, 3, 4];

if (condition.includes(type)) {
    //...
}
```

# 关于列表搜索

有时候需要前端来做一些精确搜索和模糊搜索的功能，搜索也要叫过滤，一般用filter来实现。

```javascript
const a = [1,2,3,4,5];
const result = a.filter( 
  item =>{
    return item === 3
  }
)
```

可以使用find来找到符合的项，不会继续遍历数组

```javascript
const a = [1,2,3,4,5];
const result = a.find( 
  item =>{
    return item === 3
  }
)
```

# 关于扁平化数组

一个部门JSON数据中，属性名是部门id，属性值是个部门成员id数组集合，现在要把有部门的成员id都提取到一个数组集合中。

```javascript
const deps = {
'采购部':[1,2,3],
'人事部':[5,8,12],
'行政部':[5,14,79],
'运输部':[3,64,105],
}
let member = [];
for (let item in deps){
    const value = deps[item];
    if(Array.isArray(value)){
        member = [...member,...value]
    }
}
member = [...new Set(member)]
```

使用Object.values+flat优化：

```javascript
const deps = {
    '采购部':[1,2,3],
    '人事部':[5,8,12],
    '行政部':[5,14,79],
    '运输部':[3,64,105],
}
let member = Object.values(deps).flat(Infinity);
```

其中使用Infinity作为flat的参数，使得无需知道被扁平化的数组的维度。

> flat方法不支持IE浏览器

# 关于获取对象属性值

```javascript
const name = obj && obj.name;
```

可以使用可选链操作符

```javascript
const name = obj?.name;
```

#  关于输入框非空的判断

在处理输入框相关业务时，往往会判断输入框未输入值的场景。

```javascript
if(value !== null && value !== undefined && value !== ''){
    //...
}
```

使用空值合并运算符：

```javascript
if((value??'') !== ''){
  //...
}
```

# 关于异步函数

异步函数很常见，经常是用 Promise 来实现。

```javascript
const fn1 = () =>{
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve(1);
    }, 300);
  });
}
const fn2 = () =>{
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve(2);
    }, 600);
  });
}
const fn = () =>{
   fn1().then(res1 =>{
      console.log(res1);// 1
      fn2().then(res2 =>{
        console.log(res2)
      })
   })
}
```

改进：

```javascript
const fn = async () =>{
  const res1 = await fn1();
  const res2 = await fn2();
  console.log(res1);// 1
  console.log(res2);// 2
}
```
