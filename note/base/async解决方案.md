# 2.3.3 async 解决方案

前文我们通过 `Promise.all()`解决了 `async.paralle()`的功能，现在我们来看看用 Promise 配合原生 async 来达到“async”模块的其他功能。

* 实现 async.series 顺序执行异步函数

```js
//源代码
async.series([
        function(callback) {
            if (version.other_parameters != otherParams) { // 更新其他参数
                var newVersion = {
                    id: version.id,
                    other_parameters: otherParams,
                };
                CVersion.update(newVersion, callback);
            } else {
                callback(null, null);
            }
        },
        function(callback) {
            cVersionModel.removeParams(version.id, toBeRemovedParams, callback);
        },
        function(callback) {
            cVersionModel.addParams(version.id, toBeAddedParams, callback);
        },
        function(callback) {
            CVersion.get(version.id, callback);
        },
    ], function(err, results) {
        if (err) {
            logger.error("更新电路图参数失败！");
            logger.error(version);
            logger.error(tagNames);
            logger.error(err);
            callback(err);
        } else {
            callback(null, results[3].parameters);
        }
    });


//新代码

(async function(){
    if (version.other_parameters != otherParams) { // 更新其参数
        var newVersion = {
            id: version.id,
            other_parameters: otherParams,
        };
        await  CVersion.update(newVersion, callback);
    } else {
        return null
    }
    await cVersionModel.removeParams(version.id, toBeRemovedParams)
    await cVersionModel.addParams(version.id, toBeAddedParams)
    let result = await CVersion.get(version.id)
    return result
})()
..catch((err)=>{
	logger.error("更新参数失败！");
    logger.error(version);
    logger.error(tagNames);
    logger.error(err);
})
```

* 实现 async.each 的遍历集合每一个元素实现异步操作功能：

```js
//源代码
Notification.newNotifications= function(notifications, callback) {
    function iterator(notification, callback) {
        Notification.newNotification(notification, function(err, results) {
            logger.error(err);
            callback(err);
        });
    }

    async.each(notifications, iterator, function(err) {
        callback(err, null);
    });
}
```

新代码：

```js
//新代码
Notification.newNotifications= function(notifications）{
  notifications.forEach(async function(notification){
      try{
           await Notification.newNotification(notification)//异步操作
      } catch (err) {
           logger.error(err);
           return err;
        }    
  });
}
```

上述代码需要说明的情况是，在forEach 体内的每一个元素的 await 都是并发执行的，因为这正好满足了 `async.each` 的特点，如果你希望的是**数组元素继发执行异步操作**，也就是前文所提到的 `async.eachSeries` 的功能，你需要协程一个 for 循环而不是 forEach 的形式，类似如下代码：

```js
async function dbFuc(db) {
  let docs = [{}, {}, {}];

  for (let doc of docs) {
    await db.post(doc);//异步数据库操作
  }
}
```

如果你觉得上述并发集合操作使用 forEach 的方式依旧不太直观，也可以改为配合`Promise.all`的形式：

```js
async function dbFuc(db) {
  let docs = [{}, {}, {}];
  let promises = docs.map((doc) => db.post(doc));

  let results = await Promise.all(promises);
  console.log(results);
}
```
上述代码现先对数组元素进行遍历，将传入了数组元素参数的一步操作封装成为一个数组，通过`await Promise.all(promises)`的形式进行并发操作。**Tips：** `Promise.all` 有自动将数组的每个元素变成Promise对象的能力。