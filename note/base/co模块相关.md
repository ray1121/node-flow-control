# 2.3.2 不得不提的 co 模块

众所周知的是 async 函数式 generator 的语法糖，generator 在异步流程控制中的执行依赖于执行器，co 模块就是一个 generator 的执行器，在真正介绍和使用 async 解决法案之前有必要简单了解一下大名鼎鼎的 co 模块。

**什么是 generator**，详细请参考[Ecmascript6 入门](http://es6.ruanyifeng.com/#docs/generator)

```js
var fs = require('fs');

var readFile = function (fileName){
  return new Promise(function (resolve, reject){
    fs.readFile(fileName, function(error, data){
      if (error) reject(error);
      resolve(data);
    });
  });
};

var gen = function* (){
  var f1 = yield readFile('/etc/fstab');
  var f2 = yield readFile('/etc/shells');
  console.log(f1.toString());
  console.log(f2.toString());
};
// 执行生成器，返回一个生成器内部的指针
var g = gen();
//手动 generator 执行器
g.next().value.then(function(data){
  g.next(data).value.then(function(data){
    g.next(data);
  });
})

```

上述代码采用 generator 的方式在 yeild 关键字后面封装了异步操作并通过 `next()`去手动执行它。调用 g.next() 是去执行 yield 后面的异步，这个方案就是经典的异步的“协程”（多个线程互相协作，完成异步任务）处理方案。

协程执行步骤：

1. 协程A开始执行。
2. 协程A执行到一半，进入暂停，执行权转移到协程B。
3. （一段时间后）协程B交还执行权。
4. 协程A恢复执行。

**协程遇到 yield 命令就暂停** 等到执行权返回，再从暂停的地方继续往后执行。

翻译上述代码：

* `gen()`执行后返回一个生成器的内部执行指针，gen 生成器就是一个协程。
* `gen.next()`让生成器内部开始执行代码到遇到 yield 执行 yield 后，就暂停该协程，并且交出执行权，此时执行权落到了JS主线程的手里，即开始执行 Promise 的 then 解析。
* then 的回调里取得了该异步数据结果，调用`g.next(data)`通过网`next()`函数传参的形式，将结果返回给生成器的`f1`变量。
* 依次回调类推。

说明：
* `g.next()`返回一个对象，形如`{ value: 一个Promise, done: false }`到生成器内部代码执行完毕返回`{ value: undefined, done: true }`

引出一个问题: 我们不能每一次用 generator 处理异步都要手写  generator 的 then 回调执行器，该格式相同，每次都是调用`.next()`,所以可以用递归函数封装成一个函数：

```js
function run(gen){
  var g = gen();

  function next(data){
    var result = g.next(data);
    if (result.done) return result.value;
    result.value.then(function(data){
      next(data);
    });
  }

  next();
}

run(gen);
```

上述执行器的函数编写 co 模块考虑周全的写好了，[co模块源码](https://github.com/tj/co/blob/master/index.js)

你只需要：

```js
const co = require('co')
co(function* () {
  var res = yield [
    Promise.resolve(1),
    Promise.resolve(2)
  ];
  console.log(res); 
}).catch(onerror);
```
`yield` 后面的是并发。

此时我们来对比 async 写法:)

```js
async function(){
    var res = await [
    Promise.resolve(1),
    Promise.resolve(2)
    ]
    console.log(res);
}().catch(onerror);
```
async 函数就是将 Generator 函数的星号（*）替换成 async，将 yield 替换成 await，仅此而已。并且它不需要额外的执行器，因为它**自带 Generator 执行器**

本质上其实并没有脱离“协程”异步的处理方式

```js
const fs = require('fs')
const util = require('util')


let readFile = util.promisify(fs.readFile);

(async function fn() {
	var a = await readFile('./test1.txt',"utf-8")
	var b = await readFile('./test2.txt',"utf-8")
	console.log(a)
	console.log(b)
})()
.catch((e)=>{
	console.log("出错了")
})



console.log('主线程')
```

打印结果会先输出“主线程”。