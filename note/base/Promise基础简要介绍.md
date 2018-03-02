# 2.2.1 Promise基础介绍

一个简单清晰的例子：

```js
const fs = require('fs')

fs.readFile("./Test/file1.txt", "utf-8", (err, content) => {
    if (err) {
        console.log(err);
    } else {
        console.log(content);
    }
})

let readFile = () => {
    return new Promise((resolve, reject) => {
        fs.readFile("./Test/file2.txt", "utf-8", (err, content) => {
            if (err) {
                reject(err)
            } else {
                resolve(content);
            }
        })
    })
}

readFile()
    .then((res) => {
        console.log(res);
    })
    .catch((err) => {
        console.log(err);
    })
```

只是比原生的callback形式的异步函数多了一步封装包裹的过程。Promise是一个对象，可以把它看做是一个包含着异步函数可能出现的结果（成功或者失败（err））的**“异步状态小球”**。得到了这个小球你就能用 then 去弄他，用 catch 去捕获它的失败。简单的概括，也仅此而已。基于这个小球，我们就能得到所谓的“现代异步处理方案”了，后话。

前端 Promisify Ajax请求：

```js
let btn = document.getElementById("btn")
let getData = (api) => {
    return new Promise((resolve,reject)=>{
        let req = new XMLHttpRequest();
        req.open("GET",api,true)
        
        req.onload = () => {
              if (req.status === 200) {
                resolve(req.responseText)
              } else {
                reject(new Error(req.statusText))
              }
            }
        
        req.onerror = () => {
              reject(new Erro(req.statusText))
            }
            req.send()
          })
        }

btn.onclick = function(e) {
    getData('/api')
        .then((res) => {
            let content=JSON.parse(res).msg
            document.getElementById("content").innerText = content
            })
        .catch((err) => {
            console.log(err);
            })
        }
```

Node提供的原生模块的API基本上都是基于一个 callback 形式的函数，我们想用 Promise ，难不成甚至原生的这些最原始的函数都要我们手动去进行 return 一个 Promise 对象的改造？其实不是这样的，Node 风格的 callback 都遵从着“错误优先”的回到函数方案，即形如`(err,res)=>{}`，并且回调函数都是最后一个参数，他们的形式都是一致的。所以Node的原生[util](http://nodejs.cn/api/util.html#util_util_promisify_original)模块提供了一个方便我们将函数 Promisfy 的工具——`util.promisfy(origin)`

```js
let readFileSeccond = util.promisify(fs.readFile)

readFileSeccond("./Test/file3.txt","utf-8")
    .then((res) => {
        console.log(res);
    })
    .catch((err) => {
        console.log(err);
    })
```

**注意，这个原生工具会对原生回调的结果进行封装，如果在最后的回调函数中除了 err 参数之外，还有不止一个结果的情况，那么 util.promisify 会将结果都统一封装进一个对象之中。**