# 2.3.1 一个 Promise 的问题

在开始介绍 async 之前，想先聊一种情况。

基于 Promise 的这一套看似可以让代码“竖着写”，可以很好的解决“callbackHell”回调地狱的窘境，但是上述所有的例子都是简单场景下。在基于 Promise 的 then 链中我们不难发现，虽然一层层往下的 then 链可以向下一层传递本层处理好的数据，但是这种链条并不能跨层使用数据，就是说如果第3层的 then 想直接使用第一层的结果必须有一个前提就是第二层不仅将自己处理好的数据 `return` 给第三层，同时还要把第一层传下来的再一次传给第三层使用。不然还有一种方式，那就是我们从回调地狱陷入另一种地狱 “Promise地狱”。

借用这篇[博客](https://juejin.im/entry/58523b908e450a006c4d0c5b) 的一个操作 mongoDB 场景例子说明：

```js
MongoClient.connect(url + db_name).then(db => {
    return db.collection('blogs');
}).then(coll => {
    return coll.find().toArray();
}).then(blogs => {
    console.log(blogs.length);
}).catch(err => {
    console.log(err);
})
```

如果我想要在最后一个 then 中得到 db 对象用来执行 `db.close()`关闭数据库操作，我只能选择让每一层都传递这个 db 对象直至我使用操作 then 的尽头，像下面这样：

```js
MongoClient.connect(url + db_name).then(db => {
    return {db:db,coll:db.collection('blogs')};
}).then(result => {
    return {db:result.db,blogs:result.coll.find().toArray()};
}).then(result => {
    return result.blogs.then(blogs => {   //注意这里，result.coll.find().toArray()返回的是一个Promise，因此这里需要再解析一层
        return {db:result.db,blogs:blogs}
    })
}).then(result => {
    console.log(result.blogs.length);
    result.db.close();
}).catch(err => {
    console.log(err);
});
```

下面陷入 “Promise地狱”：

```js
MongoClient.connect(url + db_name).then(db => {
    let coll = db.collection('blogs');
    coll.find().toArray().then(blogs => {
        console.log(blogs.length);
        db.close();
    }).catch(err => {
        console.log(err);
    });
}).catch(err => {
    console.log(err);
})
```

看上去不是那么明显，但是**已经出现了 then 里面嵌套 then 了**，操作一多直接一夜回到解放前，再一次丧失了让人想看代码的欲望。OK，用传说中的 async 呢

```js
(async function(){
    let db = await MongoClient.connect(url + db_name);
    let coll = db.collection('blogs');
    let blogs = await coll.find().toArray();
    console.log(blogs.length);
    db.close();
})().catch(err => {
    console.log(err);
});
```
**各种异步写的像同步**了，`async`（异步）关键字声明，告诉读代码的这是一个包含了各种异步操作的函数，`await`（得等它）关键字说明后面的是个异步操作，卡死了等他执行完再往下。这个语义以及视觉确实没法否认这可能是“最好的”异步解决方案了吧。