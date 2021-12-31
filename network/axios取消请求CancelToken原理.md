<!-- toc -->

# axios取消请求方法

我们先来看一下官方提供的取消请求写法：

```javascript
const CancelToken = axios.CancelToken;
const source = CancelToken.source();

axios.get('/user/12345', {
    cancelToken: source.token
}).catch(function (thrown) {
    if (axios.isCancel(thrown)) {
        console.log('Request canceled', thrown.message);
    } else {
        // 处理错误
    }
});

axios.post('/user/12345', {
    name: 'new name'
}, {
    cancelToken: source.token
})

// 取消请求（message 参数是可选的）
source.cancel('Operation canceled by the user.');
```

可以看到引入了CancelToken的source方法进行取消请求。

# CancelToken.js

在看 CancelToken.js 之前我们先引入一个知识，如何在promise外面控制他的resolve/reject方法？看一个例子：

```javascript
let resolveHandle;
new Promise((resolve) => {
    resolveHandle = resolve;
}).then((val) => {
    console.log('resolve', val);
});
resolveHandle('ok');
```

可以看到，我们用resolveHandle获取了Promise的resolve方法控制权，这样就可以在promise外部控制他的成功了。

接下来回归主题，我们找到axios的[源码](https://github.com/axios/axios) ，在 /lib/cancel/ 下找到CancelToken.js

先看他的source方法：

```javascript
CancelToken.source = function source() {
    var cancel;
    var token = new CancelToken(function executor(c) {
        cancel = c;
    });
    return {
        token: token,
        cancel: cancel
    };
};
```

在source中，返回了一个token和一个cancel。

token是CancelToken类的一个实例

cancel则是拿到了CancelToken实例的c方法，这个c方法是什么，继续往上看源码：

```javascript
function CancelToken(executor) {
......

    var resolvePromise;

    this.promise = new Promise(function promiseExecutor(resolve) {
        resolvePromise = resolve;
    });

......

    executor(function cancel(message) {
        if (token.reason) {
            // Cancellation has already been requested
            return;
        }

        token.reason = new Cancel(message);
        resolvePromise(token.reason);
    });
}
```

可以看到c方法就是CancelToken中的cancel方法，在cancel方法中，通过上面示例的方式，把Promise的控制权放在了外层。

那么调用cancel方法就能让promise直接resolve。

之后我们去 xhr.js 中，看resolve之后发生了什么。

# xhr.js

在 /lib/adapters/ 下找到xhr.js。关键代码：

```javascript
if (config.cancelToken || config.signal) {
    // Handle cancellation
    // eslint-disable-next-line func-names
    onCanceled = function (cancel) {
        if (!request) {
            return;
        }
        reject(!cancel || (cancel && cancel.type) ? new Cancel('canceled') : cancel);
        request.abort();
        request = null;
    };

    config.cancelToken && config.cancelToken.subscribe(onCanceled);
    if (config.signal) {
        config.signal.aborted ? onCanceled() : config.signal.addEventListener('abort', onCanceled);
    }
}
```

resolve后，会触发promise的then方法，立即执行abort方法取消请求，同时调用reject让外层的promise失败。


至此，成功取消请求，并返回失败。
