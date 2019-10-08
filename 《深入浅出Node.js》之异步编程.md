

## 异步编程

### 函数式编程
**高阶函数**
> 即将函数作为请求参数或着返回值的函数。例如sort()排序方法，传入不同的函数排序方式不同，例如map(),forEach()等等都是接受一个函数作为参数。

**偏函数**
> 通过指定部分参数来来产生一个新的定制函数的形式就是偏函数。

### Node异步编程的优势和难点

**优势**
非阻塞I/O模型：使CPU和I/O并不互相依赖等待。

**难点**
1. **异常处理**
普通的try/catch模块只能捕捉到当次*事件循环*的异常，对callback()抛出的异常则无能为力。
> 约定：Node在异常处理方面形成了一种约定，*将异常作为回调函数的第一个实参返回*，如果为空，则代表没有异常。

在我们自行编写的异步方法上，也需要遵循这样的原则：
- 必须执行调用这传入的回调函数；
- 正确传回异常供调用者判断。
2. **函数嵌套过深**
   恶魔金字塔，不过多赘述；
3. **阻塞代码**
    Node没有线程沉睡的方法。如果遇到需要阻塞代码的需求，在统一规划业务逻辑之后，用setTimeout()。
4. **多线程编程**
    通常谈论的JavaScript是单线程的，指的是在浏览器中JavaScript执行线程与UI渲染公用一个线程。在Node中，只是没有UI渲染的部分，模型基本相同。对于服务端而言，如果服务端是多核CPU，单个的Node进程实质上是没有充分利用多核CPU的。Node借鉴了web workers模式。
5. **异步转同步**
    Node仅提供了少量的同步API，但对于异步调用而言，通过良好的流程控制，可以将逻辑梳理成顺序的形式。
    
### 异步编程的解决方案

#### 事件发布/订阅模式

**events模块**
Node中的events模块是事件发布/订阅模式的一个简单实现。
```
var events = require('events')
var eventEmitter = new events.EventEmitter()

//注册一个事件监听器
eventEmitter.on('event1',function(arg){
    console.log(arg)
})

//两秒种后触发
setTimeout(function(){
    eventEmitter.emit('event1')
},2000)
```
**events模块的注意事项**
1. Node的设计者认为如果一个事件的侦听器过多可能会导致内存泄露或出现过多占用CPU的情景，因此对一个事件添加超过10个监听器就会得到一条警告。调用`setMaxListeners(0)`可以将这个限制去掉。
2. 线程运行期间如果触发了error事件，eventEmitter事件会检查是否有对error事件添加了监听器，如果添加了就由监听器处理错误，否则会作为异常抛出。？

1. 继承eventEmitter类
2. 利用事件队列解决雪崩问题once()方法
3. 多异步之间的协作方案,恶魔金字塔的解决方案
   1. 自己写一个偏函数
   2. EventProxy方案：是对事件订阅/发布模式的扩充，可以自由订阅组合事件。


#### Promise/Deferred模式

> Promise/Deferred模式，是对Promise/A,Promise/B,Promise/D这样典型模型的一个统称。Promise/Deferred模式缓解了异步的深度嵌套带来的不愉快的编程体验。接下来主要以Promise/A为代表介绍Promise/Deferred模式。

#### Promise/A

定义：
1. promise操作只会处在3种状态的一种，未完成态，完成态和失败态。
2. promise的状态只会出现未完成态向完成态或失败态转化，不能逆反，完成态和失败态不能相互转化；
3. promise的状态一旦改变，将不能更改。
在API的定义上，Promise/A的提议是，一个Promise对象只要具备then方法即可。
then方法接受funtion对象，在完成态或者错误态出现后调用相应的回调函数；then方法继续返回一个Promise对象，以实现链式调用；

##### Promise/Deferred示例

这里尝试通过继承Node中的events模块来实现一个简单的Promise/deferred模式
```
var events = require('events')
var util = require('util')
var EventEmitter = events.EventEmitter

var Promise = function () {
    EventEmitter.call(this)
}
util.inherits(Promise, EventEmitter)

Promise.prototype.then = function (fulfilledHandler, errorHandler, progressHandler) {
    if (typeof fulfilledHandler === 'function') {
        this.once('success', fulfilledHandler)
    }
    if (typeof errorHandler === 'function') {
        this.once('error', errorHandler)
    }
    if (typeof progressHandler === 'function') {
        this.on('progress', progressHandler)
    }
}
// 延迟对象
var Deferred = function () {
    this.state = 'unfulfilled'
    this.promise = new Promise()
}
Deferred.prototype.resolve = function (obj) {
    this.state = 'fulfilled'
    this.promise.emit('success', obj)
}
Deferred.prototype.reject = function (err) {
    this.state = 'failed'
    this.promise.emit('error', err)
}
Deferred.prototype.progress = function (data) {
    this.progress.emit('progress', data)
}
function readFile(fileUrl) {
    var deferred = new Deferred()
    fs.readFile(fileUrl, 'utf8', function (err, file) {
        if (err) {
            deferred.reject(err)
        } else {
            deferred.resolve(file)
        }
    })
    return deferred.promise
}
readFile('file1.txt').then(file => {
    console.log(file)
})

```
从上述代码块可以看到，Deferred主要用于内部，用于维护异步模型的状态；Promise则作用于外部，通过then方法暴露给外部以添加自定义逻辑。
##### Promise/Deferred的多异步调用（Promise.all的实现）

```
Deferred.prototype.all = function (promises) {
    var count = promises.length
    var that = this
    var results = []
    promises.forEach(function (promise, i) {
        promise.then(function (data) {
            count--
            results[i] = data
            if (count === 0) {
                that.resolve(results)
            }
        }, function (err) {
            that.reject(err)
        })
    })
}

function readFiles() {
    var deferred = new Deferred()
    var promise1 = readFile('file1.txt')
    var promise2 = readFile('file2.txt')
    deferred.all([promise1, promise1])
    return deferred.promise
}
readFiles().then(data => {
    console.log(data)
})

```
##### Promise的链式调用
形成Promise的关键条件是then方法中的回调函数要返回一个Promise对象，当Promise对象检测到返回新的Promise对象时，当前Deferred的引用就改变新的Promise对象，然后继续执行；

```
var fs = require('fs')
// 实现promise的链式调用
var Promise = function () {
    this.queue = []//队列用于存储待执行的回调函数
    this.isPromise = true
}
// then函数的返回值也是一个promise，就是自己本身
Promise.prototype.then = function (fulfilledHandler, errorHandler, progressHandler) {
    var handler = {}//用于存放回调函数
    if (typeof fulfilledHandler === 'function') {
        handler.fulfilled = fulfilledHandler
    }
    if (typeof errorHandler === 'function') {
        handler.error = errorHandler
    }
    if (typeof progressHandler === 'function') {
        handler.progress = progressHandler
    }
    this.queue.push(handler)
    return this
}
var Deferred = function () {
    this.promise = new Promise()
}
Deferred.prototype.resolve = function (obj) {
    var promise = this.promise
    var handler // 回调函数
    while ((handler = promise.queue.shift())) {
        if (handler && handler.fulfilled) {
            var ret = handler.fulfilled(obj) // 执行相应的回调函数，并保存回调函数的返回值
            //如果回调函数的返回值存在并且是一个promise对象
            if (ret && ret.isPromise) {
                // 将当前deferred对象的promise引用改变为新的promise对象
                ret.queue = promise.queue
                // 并将队列中余下的回调转交给它
                this.promise = ret
                return//退出当前循环，等待执行下一个回调函数
            }
        }
    }
}
Deferred.prototype.reject = function (err) {
    var promise = this.promise
    var handler
    while ((handler = promise.queue.shift())) {
        if (handler && handler.error) {
            var ret = handler.error(err)
            if (ret && ret.isPromise) {
                ret.queue = promise.queue
                this.promise = ret
                return
            }
        }
    }
}
// 生成回调函数
Deferred.prototype.callback = function () {
    var that = this
    return function (err, file) {
        if (err) {
            return that.reject(err)
        }
        that.resolve(file)
    }
}
// 以两次文件读取作为例子
var readFile1 = function (file, encoding) {
    var deferred = new Deferred()
    fs.readFile(file, encoding, deferred.callback())
    return deferred.promise
}
var readFile2 = function (file, encoding) {
    var deferred = new Deferred()
    fs.readFile(file, encoding, deferred.callback())
    return deferred.promise
}
readFile1('file1.txt', 'utf8').then(function (file1) {
    return readFile2('file2.txt', 'utf8')
}).then(function (file2) {
    console.log(file2)
})
```


#### 流程控制库
事件发布/订阅模式和Promise/Deferred模式是经典的解决异步编程问题的方案。接下来介绍的非规范的方法，想比之下更为灵活。
##### 中间件Connect
connect是Node.js中的一个中间件框架。
一共有两种类型的中间件。其中一种是过滤器，过滤器处理请求，但并不针对请求进行回应，例如服务器日志。第二种类型是供应器，它会针对请求进行回应，可以根据各自需求使用多个中间件，HTTP请求将会通过每一个中间件知道其中一个中间件对请求进行回应。

**中间件的流式处理**

```
var app = connect()
app.use(connect.staticCache())
app.use(connect.cookieParser())
app.use(connect.session())
...
app.listen(3001)//通过use()方法注册好一系列中间件以后，监听端口上的请求
```
中间件利用了尾触发的机制，一个最简单的中间件：
`function(req,res,next){
    // 中间件
}`
> 每一个中间件传递`请求对象`，`响应对象`和`尾触发函数`，通过队列形成一个处理流。

**中间件的核心实现**

```
function createServer(){
    function app(req,res){ app.handle(req, res); }//创建了HTTP服务器的request事件处理函数
    utils.merge(app, proto)
    utils.merge(app, EventEmitter.prototype)
    app.route = '/'
    app.stack = []//存储中间件的队列
    for(var i=0; i < arguments.length; ++i){
        app.use(arguments[i])
    }
    return app
}
app.use = function(route,fn){
    ...
    this.stack.push({route:route, handle: fn})
    return this
}
// 接下来结合Node原生http模块实现监听即可
app.listen = function(){
    var server = http.createServer(this)
    return server.listen.apply(server,arguments)
}
// 回到app.handle()方法，每一个监听到的网络请求都将从这里开始
app.handle = function(req,res,out){
    ...
    next()
}
```

> 如果把一个http处理过程比作是污水处理，中间件就像是一层层的过滤网。每个中间件在http处理过程中改写成request或response的数据、状态，实现了特定的功能。

### async
1. async提供series()方法实现一组异步任务的串行执行；
2. async提供parallel()方法并行执行异步操作；
3. 之前提到的EventProxy方案做对比；
4. 当串行执行的异步任务，前一个异步任务的结果是后一个异步任务的输入时，async提供了waterfall()方法来满足。
5. async提供auto()方法可以根据依赖关系自动分析，以最佳的顺序执行业务。
6. 相应的用EventProxy实现一下。

### Step

> Step是另一个知名的流程控制库，比async更轻量，在API的暴露上也更具备一致性。它只有一个接口Step。

`Step(task1,task2,task3);`

### wind

> wind用于解决一些特殊场景下的异步方案。
> 异步方法在JavaScript中通常会立即返回，在wind中做到了不阻塞CPU但阻塞代码的目的。

如同Promise/Deferred模式可以让异步编程模型变简单，这种近同步编程的体验需要我们额外或者提前完成的事情是：将异步方法任务化。这种任务化的过程可以看作是Promise/Deferred的封装。但是如果每个方法都一一封装，那将会是一个庞大的工作量，wind提供了两个方法来辅助转换。
`var readFileAsync = Wind.Async.Binding.fromStandard(fs.readFile)`

总结：

### 异步并发控制
#### bagpipe的解决方案
1. 通过一个队列来控制并发量；
2. 如果当前活跃（指调用发起但未执行回调）的异步调用量小于限定值，从队列中取出执行；
3. 如果活跃调用达到限定值，调用暂时存放在队列中；
4. 每个异步调用结束时，从队列中取出新的异步调用执行；
