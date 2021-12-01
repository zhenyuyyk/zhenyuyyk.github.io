# map

特性：

1. map不改变原数组，会返回新数组
2. 可以使用break中断循环，可以使用return返回到外层函数

实例：

```javascript
let arr = [1, 3, 4]
let newarr = arr.map(i => {
    return i += 1;
    console.log(i);
})
console.log(arr)//1,3,4---不会改变原数组
console.log(newarr)//[2,4,5]---返回新数组
```

# forEach

特性：

1. 代码简洁，效率高，不用关心集合下标的问题
2. 没有返回值
3. 不能使用break中断循环，不能使用return返回到外层函数
4. 对于空数组是不会执行回调函数的
5. 不能使用continue跳过循环中的一个迭代（区别于for循环）,需要用 return 跳过循环中的一个迭代

实例：

```javascript
let arr = [1, 3, 4]
let newarr = arr.forEach(i => {
    i += 1;
    console.log(i);//2,4,5
})
console.log(arr)//[1,3,4]
console.log(newarr)//undefined
```

# for in

特性：

1. 可以遍历数组的键名，遍历对象简洁

实例：

```javascript
let person = {name: "小白", age: 28, city: "北京"}
let text = ""
for (let i in person) {
    text += person[i]
}
console.log(text)
// 输出结果为：小白28北京
//其次在尝试一些数组
let arry = [1, 2, 3, 4, 5]
for (let i in arry) {
    console.log(arry[i])
}
//能输出出来，证明也是可以的，但是不推荐
```

# for of

特性：

1. 可遍历map，object,array,set string等,不能遍历对象
2. 可以使用break,continue和return，不仅支持数组的遍历，还可以遍历类似数组的对象

实例：

```javascript
let arr = ["1", "2", "3", "4"];
for (let item of arr) {
    console.log(item)
}//结果为1 2 3 4
```
``
