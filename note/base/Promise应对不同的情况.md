# 2.2.2 Promise应对不同的情况

实际代码逻辑中我们可能会面对各种异步流程控制的情况，像是之前介绍 async 模块一样，一种很常见的情况就是有很多的异步方法是可以同时并发发起请求的，即互相不依赖对方的结果，`async.parallel`的效果那样。Promise 除了封装异步之外还未我们提供了一些原生方法去面对类似这样的情况：

#### 知识准备

* Promise.resolve(value)

它是下面这段代码的语法糖：

```js
new Promise((resolve)=>{
    resolve(value)
})
```
注意点，在 then 调用的时候**即便一个promise对象是立即进入完成状态的，那Promise的 then 调用也是异步的**，这是为了避免同步和异步之间状态出现了模糊。所以你可以认为，**Promise 只能是异步的**，用接下的代码说明：

```js
let promiseA = new Promise((resolve) => {
    console.log("1.构造Promise函数");
    resolve("ray is handsome")
})

promiseA.then((res) => {
    console.log("2.成功态");
    console.log(res);
})

console.log("3.最后书写");
```

上面的代码，打印的结果如下：

```
1.构造Promise函数
3.最后书写
2.成功态
ray is handsome
```

promise 可以链式 then ，每一个 then 之后都会产生一个新的 promise 对象，在 then 链中前一个 then 这种可以通过 `return`的方式想下一个 then 传递值，这个值会自动调用 `promise.resolve()`转化成一个promise对象，代码说明吧：

```js
const fs = require('fs')
let promise = Promise.resolve(1)
promise
    .then((value) => {
            console.log(value)
            return value+1
    })
    .then((value) => {
            console.log(`first那里传下来的${value}`);
            return value+1
    })
    .then((value) => {
            console.log(`second那里传下来的${value}`);
            console.log(value)
    })
    .catch((err) => {
        console.log(err);
    })
```

上面的代码答应的结果：

```
1
first那里传下来的2
second那里传下来的3
3
```

此外 then 链中应该添加 catch 捕获异常，**某一个 then 中出现了错误则执行链会跳过后来的 then 直接进入 catch** 

#### 得到 `async.parallel`同样的效果

Promise 提供了一个**原生方法 Promise.all(arr)**,其中arr是一个由 promise 对象组成的一个数组。该方法可以**实现让传入该方法的数组中的 promise 同时执行，并在所有的 promise 都有了最终的状态之后，才会调用接下来的 then 方法，并且得到的结果和在数组中注册的结果保持一致**。看下面的代码：

```js
const fs = require('fs')
const util = require('util')

let readFile = util.promisify(fs.readFile)

let files = [readFile("../../Test/file1.txt","utf-8"),
            readFile("../../Test/file2.txt","utf-8"),
            readFile("../../Test/file3.txt","utf-8"),]

Promise.all(files)
    .then((res) => {
        console.log(res)
    })
    .catch((err) => {
        console.log(err);
    })
```

上面的代码最终会打印,即是按顺序的三个txt文件里面的内容组成的数组：

```
[‘1’,‘2’,‘3’]
```

对比 `async.parallel`的用法，发现得到相同的结果。

此外，与 `Promise.all`方法相对应的还有一个**Promise.race**，该方法与all用法相同，同样是传入一个由 promise 对象组成的数组，你可以把上面的代码中的 all 直接换成 race 看看是什么效果。没错，对于指导 race 这个英文单词意思的可能已经猜出来了，race 竞争，赛跑，就是只要数组中有一个 promise 到达最终态，该方法的 then 就会执行。所以该代码有可能会出现'1','2','3'中的任何一个字符串。

至此，我们解决了要改造的代码的第一个问题，那就是多异步的同时执行，那么之前 async 模块介绍的其他的的功能在实际运用中也很常见的几个场景，类似顺序执行异步函数，异步集合操作要怎么使用新的方案模拟出来呢？真正的原生 **async**要登场了。