<!-- TOC -->

- [一、Promise的含义](#一promise的含义)
- [二、手写一个Promise](#二手写一个promise)
    - [让resolvePromise符合规范](#让resolvepromise符合规范)
    - [resolve的也是promise](#resolve的也是promise)
    - [catch方法](#catch方法)
- [三、完整代码](#三完整代码)

<!-- /TOC -->
# 一、Promise的含义

Promise 是异步编程的一种解决方案，ES6将其写进了语言标准。所谓Promise就是一个容器，里面保存着未来才会结束的事件（通常是一个异步操作）的结果。

**Promise对象有以下两个特点：**

+ 对象的状态不受外界影响。Promise对象代表一个异步操作，有三种状态：pending（进行中）、fulfilled/resolved（已成功）、rejected（已失败）。只有异步操作的结果，可以决定当前是哪一种状态，任何其他操作都无法改变这个状态。
+ 一旦状态改变，就不会再改变，任何时候都可以得到这个结果。Promise对象的状态改变，只有两种可能：从pending变为fulfilled和从pending变为rejected。只要这两种情况发生，状态就凝固了，不会再变了，会一直保持这个结果，这时就称为 resolved（已定型）。如果改变已经发生了，你再对Promise对象添加回调函数，也会立即得到这个结果。

**Promise的优点：**

可以将异步操作以同步操作的流程表达出来，避免了层层嵌套的回调函数，解决了回调地狱的问题。

# 二、手写一个Promise

智者见者，仁者见仁，不同的人就会有不同的Promise实现，但是大家都必须遵循[promise a+](https://promisesaplus.com/) 规范

那符合规范的一个Promise到底是长什么样的?

1. Promise是一个类, 类中需要传入一个executor执行器， 默认会立即执行
2. promise有内部会提供两个方法，注意不是原型对象上的，这两个方法会传给用户,可以更改promise的状态。
3. promise有三个状态 
    + 等待（PENDING)
    + 成功（RESOLVED）返回成功的结果，如果不写结果返回undefined
    + 失败（REJECTED）返回失败的原因，如果不写原因返回undefined
4. promise只会从等待变为成功或者从等待变为失败。
5. 每个promise实例上都要有一个then方法， 分别是成功和失败的回调。
6. promise必须支持异步

ok，基于以上所述我们写一个最基本的promise

```
const PENDING = 'PENDING';
const RESOLVED = 'RESOLVED';
const REJECTED = 'REJECTED';

class myPromise {
	constructor(executor) {
		this.status = PENDING; // 宏变量, 默认是等待态
		this.value = undefined; // then方法要访问到所以放到this上
		this.reason = undefined; // then方法要访问到所以放到this上
		this.onResolvedCallbacks = [];// 专门存放成功的回调函数
		this.onRejectedCallbacks = [];// 专门存放成功的回调函数
		let resolve = (value) => {
			if (this.status === PENDING) {// 保证只有状态是等待态的时候才能更改状态
				this.value = value;
				this.status = RESOLVED;
				// 需要让成功的方法依次执行
				this.onResolvedCallbacks.forEach(fn => fn());
			}
		};
		let reject = (reason) => {
			if (this.status === PENDING) {
				this.reason = reason;
				this.status = REJECTED;
				// 需要让失败的方法依次执行
				this.onRejectedCallbacks.forEach(fn => fn());
			}
		};
		// 执行executor传入我们定义的成功和失败函数:把内部的resolve和reject传入executor中用户写的resolve, reject
		try {
			executor(resolve, reject);
		} catch(e) {
			console.log('catch错误', e);
			reject(e); //如果内部出错 直接将error手动调用reject向下传递
		}
	}
	then(onfulfilled, onrejected) {
		if (this.status === RESOLVED) {
			onfulfilled(this.value);
		}
		if (this.status === REJECTED) {
			onrejected(this.reason);
		}
		// 处理异步的情况
		if (this.status === PENDING) {
			// this.onResolvedCallbacks.push(onfulfilled); 这种写法可以换成下面的写法，多包了一层，这叫面向切片编程，可以加上自己的逻辑
			this.onResolvedCallbacks.push(() => {
				// TODO ... 自己的逻辑
				onfulfilled(this.value);
			});
			this.onRejectedCallbacks.push(() => {
				// TODO ... 自己的逻辑
				onrejected(this.reason);
			});
		}
	}
}
```

写点测试代码试试看吧

```
let promise = new myPromise((resolve, reject) => {
	setTimeout(() => {
		resolve('xxx');
	}, 1000);
});
// 发布订阅模式应对异步 支持一个promise可以then多次
promise.then((res) => { 
	console.log('成功的结果1', res);
}, (error) => { 
	console.log(error);
});

promise.then((res) => { 
	console.log('成功的结果2', res);
}, (error) => { 
	console.log(error);
});
```

结果

```
成功的结果1 xxx
成功的结果2 xxx
```

到此，我们其实做了很少的工作但已经实现了promise最基本也是最核心的功能了。接下来我们加上链式调用，这里面可能比较绕，但只要我们记住下面几条就会很轻松掌握其中原理：

1. then方法返回的必须是一个promise，这样才能保证链式调用。
2. 如果then内部的回调函数执行结果依然是一个promise那就把这个promise的结果resolve出去。
3. 任何一个promise必须是resolve之后才能走到它then方法，从而创建下一个的promise。
4. 什么时候走成功回调？then中返回一个普通值或者一个成功的promise
5. 什么时候走失败回调？返回一个失败的promise，或者抛出异常

接下来看看代码理解下吧

```
const PENDING = 'PENDING';
const RESOLVED = 'RESOLVED';
const REJECTED = 'REJECTED';

function resolvePromise(promise2, x, resolve, reject) {
	if((typeof x === 'object' && x != null) || typeof x === 'function') {
		// 有可能是promise, 如果是promise那就要有then方法
		let then = x.then;
		if (typeof then === 'function') { // 到了这里就只能认为他是promise了
			// 如果x是一个promise那么在new的时候executor就立即执行了，就会执行他的resolve，那么数据就会传递到他的then中
			then.call(x, y => {// 当前promise解析出来的结果可能还是一个promise, 直到解析到他是一个普通值
				resolvePromise(promise2, y, resolve, reject);// resolve, reject都是promise2的
			}, r => {
				reject(r);
			});
		} else {
			// 出现像这种结果 {a: 1, then: 1} 
			resolve(x);
		}
	} else {
		resolve(x);
	}
}

class myPromise {
	constructor(executor) {
		this.status = PENDING; // 宏变量, 默认是等待态
		this.value = undefined; // then方法要访问到所以放到this上
		this.reason = undefined; // then方法要访问到所以放到this上
		// 专门存放成功的回调函数
		this.onResolvedCallbacks = [];
		// 专门存放成功的回调函数
		this.onRejectedCallbacks = [];
		let resolve = (value) => {
			if (this.status === PENDING) { // 保证只有状态是等待态的时候才能更改状态
				this.value = value;
				this.status = RESOLVED;
				// 需要让成功的方法一次执行
				this.onResolvedCallbacks.forEach(fn => fn());
			}
		};
		let reject = (reason) => {
			if (this.status === PENDING) {
				this.reason = reason;
				this.status = REJECTED;
				// 需要让失败的方法一次执行
				this.onRejectedCallbacks.forEach(fn => fn());
			}
		};
		// 执行executor 传入成功和失败:把内部的resolve和 reject传入executor中用户写的resolve, reject
		try {
			executor(resolve, reject); // 立即执行
		} catch (e) {
			console.log('catch错误', e);
			reject(e); //如果内部出错 直接将error 手动调用reject向下传递
		}
	}
	then(onfulfilled, onrejected) {
		// 为了实现链式调用，创建一个新的promise
		let promise2 = new myPromise((resolve, reject) => {
			if (this.status === RESOLVED) {
				// 执行then中的方法 可能返回的是一个普通值，也可能是一个promise，如果是promise的话，需要让这个promise执行
				// 使用宏任务把代码放在一下次执行,这样就可以取到promise2,为什么要取到promise2? 这里在之后会介绍到
				setTimeout(() => {
					try {
						let x = onfulfilled(this.value);
						resolvePromise(promise2, x, resolve, reject);
					} catch (e) { // 一旦执行then方法报错就走到下一个then的失败方法中
						console.log(e);
						reject(e);
					}
				}, 0);
			}
			if (this.status === REJECTED) {
				setTimeout(() => {
					try {
						let x = onrejected(this.reason);
						resolvePromise(promise2, x, resolve, reject);
					} catch (e) {
						reject(e);
					}
				}, 0);
			}
			// 处理异步的情况
			if (this.status === PENDING) {
				// 这时候executor肯定是有异步逻辑
				this.onResolvedCallbacks.push(() => {
					setTimeout(() => {
						try {
							let x = onfulfilled(this.value);
							// 注意这里传入的是promise2的resolve和reject
							resolvePromise(promise2, x, resolve, reject);
						} catch (e) {
							reject(e);
						}
					}, 0);
				});
				this.onRejectedCallbacks.push(() => {
					setTimeout(() => {
						try {
							let x = onrejected(this.reason);
							resolvePromise(promise2, x, resolve, reject);
						} catch (e) {
							reject(e);
						}
					}, 0);
				});
			}
		});

		return promise2;
	}
}
```

主要就是多了resolvePromise这么一个函数，用来递归处理then内部回调函数执行后的结果，它有4个参数：

1. promise2: 就是新生成的promise，这里至于为什么要把promise2传过来后面会介绍。
2. x: 我们要处理的目标
3. resolve: promise2的resolve, 执行之后promise2的状态就变为成功了，就可以在它的then方法的成功回调中拿到最终结果。
4. reject: promise2的reject, 执行之后promise2的状态就变为失败，在它的then方法的失败回调中拿到失败原因。

到了这里基本上完整的Promise已经实现了，接下来我们做一些完善工作。

## 让resolvePromise符合规范

上面曾问到resolvePromise第一个参数promise2到底有什么用？其实很简单就是为了符合promise a+ 规范。下面我们来完善resolvePromise

```
function resolvePromise(promise2, x, resolve, reject) {
	// 1)不能引用同一个对象 可能会造成死循环
	if (promise2 === x) {
		return reject(new TypeError('[TypeError: Chaining cycle detected for promise #<Promise>]----'));
	}
	let called;// promise的实现可能有多个，但都要遵循promise a+规范，我们自己写的这个promise用不上called,但是为了遵循规范才加上这个控制的，因为别人写的promise可能会有多次调用的情况。
	// 2)判断x的类型，如果x是对象或者函数，说明x有可能是一个promise，否则就不可能是promise
	if((typeof x === 'object' && x != null) || typeof x === 'function') {
		// 有可能是promise promise要有then方法
		try {
			// 因为then方法有可能是getter来定义的, 取then时有风险，所以要放在try...catch...中
			// 别人写的promise可能是这样的
			// Object.defineProperty(promise, 'then', {
			// 	get() {
			// 		throw new Error();
			// 	}
			// })
			let then = x.then; 
			if (typeof then === 'function') { // 只能认为他是promise了
				// x.then(()=>{}, ()=>{}); 不要这么写，以防以下写法造成报错， 而且也可以防止多次取值
				// let obj = {
				// 	a: 1,
				// 	get then() {
				// 		if (this.a++ == 2) {
				// 			throw new Error();
				// 		}
				// 		console.log(1);
				// 	}
				// }
				// obj.then;
				// obj.then

				// 如果x是一个promise那么在new的时候executor就立即执行了，就会执行他的resolve，那么数据就会传递到他的then中
				then.call(x, y => {// 当前promise解析出来的结果可能还是一个promise, 直到解析到他是一个普通值
					if (called) return;
					called = true;
					resolvePromise(promise2, y, resolve, reject);// resolve, reject都是promise2的
				}, r => {
					if (called) return;
					called = true;
					reject(r);
				});
			} else {
				// {a: 1, then: 1} 
				resolve(x);
			}
		} catch(e) {// 取then出错了 有可能在错误中又调用了该promise的成功或则失败
			if (called) return;
			called = true;
			reject(e);
		}
	} else {
		resolve(x);
	}
}
```

对于1）不能引用同一个对象 可能会造成死循环，我们举个例子：

```
let promise = new myPromise((resolve, reject) => {
	resolve('hello');
});
let promise2 = promise.then(() => {
	return promise2;
});
promise2.then(() => {}, (err) => {
	console.log(err);
});
```

这样就会报错，因为promise的then方法执行的时候创建了promise2，这个时候promise2状态是pending， 而成功回调里又返回promise2,既然返回的结果是一个promise那就继续解析尝试在它的then方法中拿到这个promise的结果，此时promise2的状态依然是pending，那么执行promise2.then方法只会添加订阅，而一直得不到resolve, 于是自己等待自己就死循环了。

## resolve的也是promise

有这么一种情况比如

```
new Promise((resolve, reject) => {
	resolve(new Promise((resolve, reject) => {
		resolve('hello');
	}));
});
```

我们上面实现的代码就无法完成这么一个操作了，修改很简单

```
let resolve = (value) => {
	// 判断value的值
	if (value instanceof Promise) {
		value.then(resolve, reject);//resolve和reject都是当前promise的， 递归解析直到是普通值, 这里的resolve,reject都取的到，因为resolve的执行是在这两个函数执行之后，这里递归是防止value也是一个promise
		return;
	}
	if (this.status === PENDING) { // 保证只有状态是等待态的时候才能更改状态
		this.value = value;
		this.status = RESOLVED;
		// 需要让成功的方法一次执行
		this.onResolvedCallbacks.forEach(fn => fn());
	}
};
```

## catch方法

catch方法其实就是没有成功回调的then方法，这个很好理解，因为一旦失败之后就会调用reject,最终都会走到then方法的失败回调中，只是简单的把then方法换个名字而已。

```
catch(errCallback) {
    return this.then(null, errCallback);
}
```

# 三、完整代码

```

const PENDING = 'PENDING';
const RESOLVED = 'RESOLVED';
const REJECTED = 'REJECTED';

function resolvePromise(promise2, x, resolve, reject) {
	if (promise2 === x) {
		return reject(new TypeError('[TypeError: Chaining cycle detected for promise #<Promise>]----'));
	}
	let called;
	if((typeof x === 'object' && x != null) || typeof x === 'function') {
		try {
			let then = x.then; 
			if (typeof then === 'function') { 
				then.call(x, y => {
					if (called) return;
					called = true;
					resolvePromise(promise2, y, resolve, reject);
				}, r => {
					if (called) return;
					called = true;
					reject(r);
				});
			} else {
				resolve(x);
			}
		} catch(e) {
			if (called) return;
			called = true;
			reject(e);
		}
	} else {
		resolve(x);
	}
}

class myPromise {
	constructor(executor) {
		this.status = PENDING; 
		this.value = undefined; 
		this.reason = undefined; 
		this.onResolvedCallbacks = [];
		this.onRejectedCallbacks = [];
		let resolve = (value) => {
			if (value instanceof Promise) {
				value.then(resolve, reject);
				return;
			}
			if (this.status === PENDING) { 
				this.value = value;
				this.status = RESOLVED;
				this.onResolvedCallbacks.forEach(fn => fn());
			}
		};
		let reject = (reason) => {
			if (this.status === PENDING) {
				this.reason = reason;
				this.status = REJECTED;
				this.onRejectedCallbacks.forEach(fn => fn());
			}
		};
		try {
			executor(resolve, reject); 
		} catch (e) {
			reject(e);
		}
	}
	then(onfulfilled, onrejected) {
		onfulfilled = typeof onfulfilled === 'function' ? onfulfilled : v => v;
		onrejected = typeof onrejected === 'function' ? onrejected : error => { throw error };
		let promise2 = new Promise((resolve, reject) => {
			if (this.status === RESOLVED) {
				setTimeout(() => {
					try {
						let x = onfulfilled(this.value);
						resolvePromise(promise2, x, resolve, reject);
					} catch (e) { 
						console.log(e);
						reject(e);
					}
				}, 0);
			}
			if (this.status === REJECTED) {
				setTimeout(() => {
					try {
						let x = onrejected(this.reason);
						resolvePromise(promise2, x, resolve, reject);
					} catch (e) {
						reject(e);
					}
				}, 0);
			}
			if (this.status === PENDING) {
				this.onResolvedCallbacks.push(() => {
					setTimeout(() => {
						try {
							let x = onfulfilled(this.value);
							resolvePromise(promise2, x, resolve, reject);
						} catch (e) {
							reject(e);
						}
					}, 0);
				});
				this.onRejectedCallbacks.push(() => {
					setTimeout(() => {
						try {
							let x = onrejected(this.reason);
							resolvePromise(promise2, x, resolve, reject);
						} catch (e) {
							reject(e);
						}
					}, 0);
				});
			}
		});

		return promise2;
	}
	catch(errCallback) {
		return this.then(null, errCallback);
	}
}

```