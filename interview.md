1. 模块的加载机制
```js
在 Node.js 中模块加载一般会经历 3 个步骤，路径分析、文件定位、编译执行。

按照模块的分类，按照以下顺序进行优先加载：

系统缓存：模块被执行之后会会进行缓存，首先是先进行缓存加载，判断缓存中是否有值。
系统模块：也就是原生模块，这个优先级仅次于缓存加载，部分核心模块已经被编译成二进制，省略了 路径分析、文件定位，直接加载到了内存中，系统模块定义在 Node.js 源码的 lib 目录下，可以去查看。
文件模块：优先加载 .、..、/ 开头的，如果文件没有加上扩展名，会依次按照 .js、.json、.node 进行扩展名补足尝试，那么在尝试的过程中也是以同步阻塞模式来判断文件是否存在，从性能优化的角度来看待，.json、.node最好还是加上文件的扩展名。
目录做为模块：这种情况发生在文件模块加载过程中，也没有找到，但是发现是一个目录的情况，这个时候会将这个目录当作一个 包 来处理，Node 这块采用了 Commonjs 规范，先会在项目根目录查找 package.json 文件，取出文件中定义的 main 属性 ("main": "lib/hello.js") 描述的入口文件进行加载，也没加载到，则会抛出默认错误: Error: Cannot find module 'lib/hello.js'
node_modules 目录加载：对于系统模块、路径文件模块都找不到，Node.js 会从当前模块的父目录进行查找，直到系统的根目录。
```

2. 循环引用

3. 对象引用关系考察，module.exports与exports的区别
   
exports 相当于 module.exports 的快捷方式
```js
const exports = modules.exports;
```
但是要注意不能改变 exports 的指向，我们可以通过 exports.test = 'a' 这样来导出一个对象, 但是不能向下面示例直接赋值，这样会改变 exports 的指向
```js
// 错误的写法 将会得到 undefined
exports = {
  'a': 1,
  'b': 2
}

// 正确的写法
modules.exports = {
  'a': 1,
  'b': 2
}
```

4. 对buffer的理解
  
Buffer 用于读取或操作二进制数据流，做为 Node.js API 的一部分使用时无需 require，用于操作网络协议、数据库、图片和文件 I/O 等一些需要大量二进制数据的场景。Buffer 在创建时大小已经被确定且是无法调整的。

5. child_process和cluster的区别

The child_process module provides the ability to spawn subprocesses in a manner that is similar, but not identical, to popen(3). This capability is primarily provided by the child_process.spawn() function:

The cluster module allows easy creation of child processes that all share server ports.

```js
集群模式实现通常有两种方案：

方案一：1 个 Node 实例开启多个端口，通过反向代理服务器向各端口服务进行转发
方案二：1 个 Node 实例开启多个进程监听同一个端口，通过负载均衡技术分配请求（Master->Worker）
首先第一种方案存在的一个问题是占用多个端口，造成资源浪费，由于多个实例是独立运行的，进程间通信不太好做，好处是稳定性高，各实例之间无影响。

第二个方案多个 Node 进程去监听同一个端口，好处是进程间通信相对简单、减少了端口的资源浪费，但是这个时候就要保证服务进程的稳定性了，特别是对 Master 进程稳定性要求会更高，编码也会复杂。

在 Nodejs 中自带的 Cluster 模块正是采用的第二种方案。
```

```js
child_process.spawn()：适用于返回大量数据，例如图像处理，二进制数据处理。
child_process.exec()：适用于小量数据，maxBuffer 默认值为 200 * 1024 超出这个默认值将会导致程序崩溃，数据量过大可采用 spawn。
child_process.execFile()：类似 child_process.exec()，区别是不能通过 shell 来执行，不支持像 I/O 重定向和文件查找这样的行为
child_process.fork()： 衍生新的进程，进程之间是相互独立的，每个进程都有自己的 V8 实例、内存，系统资源是有限的，不建议衍生太多的子进程出来，通长根据系统 CPU 核心数设置。
```

6. console.log是异步的吗

并没有什么规范或一组需求指定console.* 方法族如何工作——它们并不是JavaScript 正式的一部分，而是由宿主环境（请参考本书的“类型和语法”部分）添加到JavaScript 中的。因此，不同的浏览器和JavaScript 环境可以按照自己的意愿来实现，有时候这会引起混淆。
尤其要提出的是，在某些条件下，某些浏览器的console.log(..) 并不会把传入的内容立即输出。出现这种情况的主要原因是，在许多程序（不只是JavaScript）中，I/O 是非常低速的阻塞部分。所以，（从页面/UI 的角度来说）浏览器在后台异步处理控制台I/O 能够提高性能，这时用户甚至可能根本意识不到其发生。

如果遇到这种少见的情况，最好的选择是在JavaScript 调试器中使用断点，而不要依赖控制台输出。次优的方案是把对象序列化到一个字符串中，以强制执行一次“快照”，比如通过JSON.stringify(..)。

console.log打印出来的内容并不是一定百分百可信的内容。一般对于基本类型number、string、boolean、null、undefined的输出是可信的。但对于Object等引用类型来说，则就会出现上述异常打印输出。

所以对于一般基本类型的调试，调试时使用console.log来输出内容时，不会存在坑。但调试对象时，最好还是使用打断点(debugger)这样的方式来调试更好。

7. __dirname，__filename，process.cwd()区别，为什么可以直接调用
- __dirname：    获得当前执行文件所在目录的完整目录名
- __filename：   获得当前执行文件的带有完整绝对路径的文件名
- process.cwd()：获得当前执行node命令时候的文件夹目录名 
- ./：           文件所在目录

8. node查看内存
process.memoryUsage()

9. node内存泄漏产生

内存泄漏（Memory Leak）是指程序中己动态分配的堆内存由于某种原因程序未释放或无法释放，造成系统内存的浪费，导致程序运行速度减慢甚至系统崩溃等严重后果。

- 全局变量

未声明的变量或挂在全局 global 下的变量不会自动回收，将会常驻内存直到进程退出才会被释放，除非通过 delete 或 重新赋值为 undefined/null 解决之间的引用关系，才会被回收。关于全局变量上面举的几个例子中也有说明。

- 闭包

这个也是一个常见的内存泄漏情况，闭包会引用父级函数中的变量，如果闭包得不到释放，闭包引用的父级变量也不会释放从而导致内存泄漏。

- 慎将内存做为缓存

通过内存来做缓存这可能是我们想到的最快的实现方式，另外业务中缓存还是很常用的，但是了解了 Node.js 中的内存模型和垃圾回收机制之后在使用的时候就要谨慎了，为什么呢？缓存中存储的键越多，长期存活的对象也就越多,垃圾回收时将会对这些对对象做无用功。

10.   如何预防xss攻击

根据攻击的来源，XSS 攻击可分为存储型、反射型和 DOM 型三种。

[美团技术团队-如何防止XSS攻击？](https://tech.meituan.com/2018/09/27/fe-security.html)

11.   websocket、http、tcp的区别

12.   sleep()如何实现

13.   express、koa、egg的区别

14.   描述下egg的洋葱模型

15.   进程间通信方式
- 管道
- 信号量
- 消息队列
- 共享内存
- socket

16. setTimeout、setInterval和setImmediate的区别  

[Node.js 事件循环，定时器和 process.nextTick()](https://nodejs.org/zh-cn/docs/guides/event-loop-timers-and-nexttick/)

