---
title: 说一下Handler的原理
date: 2016-07-16 15:33:31
tags:
---

#### Handler在解决什么问题?
handler 是为了解决线程间通讯的问题而设计的


#### 如果让你设计你要怎么做？
那么，如果没有handler，如何做到线程A与线程B进行通讯？
回调？监听？这样的话，要么每次都要定义自己所写回调接口，双方必须定义好回调参数格式，要么定义一个通用方法，而handler其实就相当于这个通用的方法
egg:主线程 启动了子线程A去完成一项任务，并对A说"你拿着这个handler，等你做完了用它告诉我就可以"，等到A做完了，通过handler.sendMessage来发出暗号，这时候主线程收到这个消息，就知道A已经做完了，然后去处理对应的逻辑。


#### Handler是怎么设计来解决此问题的
对于这个功能，handler要定义成通用逻辑，所以handler提供了如下功能来满足不同业务场景需要（同性质的功能就不一一列出了）


    1. obtainMessage()                //获取一个空消息
    2. sendMessage(Message msg)       //发送消息
    3. handleMessage(Message msg)     //收到消息
    4. post(Runnable r)               //抛出一个runnable
    5. removeCallbacks(Runnable r)    //移除runnable


以上就构成了一个通用的回调逻辑，实际上我的很多的回调接口都可以通过此方式进行，用好系统提供的现成功能，一般情况下都好过于自己单独开发，除非你很清楚它并不能满足自己的需求


好了，线程通讯的问题解决了，看起来并没有什么特殊的，就是定义了一个通用的回调方案，然而我们都知道那个Looper还没提到

#### Handler的消息发到哪里了
handler.sendMessage 和handler.postRunanable其实质上最后都是调用到handler.sendMessageAt(msg,time)，runnable只是赋值给了msg.callback,也是一个消息，如果你愿意，完全可以Message.obtain(handler,runable)，再通过handler.sendMessage来发送runnable

那么handler.sendMessageAt(msg,time) 的逻辑做的事情就是将msg入到消息队列里面去
queue.enqueueMessage(msg,time)

#### ThreadLoacal是什么鬼
为什么你们都要提到theadLocal,这个与主题没有一毛钱关系，将其理解成Thread一个Map就很好理解了

#### MessageQueue是怎么组织的？BlockQueue?
具体的实现都在native中 ？暂且当做BlockQueue来理解//todo
enqueueMessage
异常情况(msg.target == null  || msg.isInUse() )
sendMessage会自动设置target,//什么场景下修改会为空？？
msg.isInUse会在入队列时候修改，所以在重复使用时候可以通过Message.obtain(msg)来客隆一份数据一样但是没有使用过的Message来使用
sendMessageAtFrontOfQueue

此队列并非先进先出队列，而是一个LinkedQueue,在enqueueMessage,会根据Message.when，将其插入到合适的位置，队列中的消息是按照when排序的


#### Message可以插队么？Message.when
在发消息时候发送一个when 比 当期MessageQueue时间还小的when即可插队到对头，你也许会说，我哪里知道当前对头的消息when是多少，所以直接设置when为0即可插队到对头，这可以应用在优先级高的处理

但是when可以修改么？
when是package访问权限(还好不是final)，是否可以通过新建一个与其包名相同的类来修改？
when值来自 sendMessageAt的那个时间，在enqueue时候在MessageQueue里面被赋值，不要煞费苦心修改了，平时直接发送的sendMessage的when就是0
如果同时发送的消息如果都是0，还在排队处理的话，后来的消息会插在队列的前面，都是立即处理的话，处理谁都一样


#### Message可以发给不同的线程执行么？Message.target
Handler.obtain系列会将Message.target赋值为this
而Handler.postRunnable 获取的Message没有赋值target
但是无论哪种方式，最终会在enqueueMessage中将target赋值为Handler.this
也就是是说在使用哪个Handler发送消息，最终一定是进了他的Looper

但是一旦enqueueMessage后，处理逻辑就以及脱离了Handler的控制了，执行的结果都只会发送到Message.target中

#### MainLooper ActivityThread.man() 如何做到一直待机

#### sendMessage与postRunable有什么区别
最终都是调用setMessageAt(msg,time),不同的是获取Message方式不同，postRunable是将unable赋值给message.callback

#### 如何停止Looper
Looper在loop的时候，如果取到一个Message为null，就会自行跳出，但是MessageQueue只有在队列为空和已经退出时候才会给Looper返回null. 不能通过发送null消息来停止，
同时Looper本身也提供了quite api来退出循环，原理是调用MQ.quite，在下一个next时候MQ返回给Looper一个null,这样Looper就自行结束

在Looper中会调用msg.target.dispatchMessage(msg),这样又将消息返回给handler来执行了，而此时消息执行是looper中调用的，也就是消息执行是在looper所在的线程调用的，这样此消息也就在此线程被消费
同时会调用msg.resycleUnchecked()回收此对象

#### 消息执行的优先级
- 会先处理msg.callback也（post进来的runnable就是设置到callback中）
- 如果handler有设置callback，消息会发送到callback的handlermessge()中消费
- 如果没设置就在本handler.handlerMessage()中消费
- 而一般情况下我的处理逻辑都在handlerMessage中，这样消息就又回来了

#### HandlerThread替我们做了哪些事情
- http://blog.csdn.net/qq_23547831/article/details/50936584

#### 为什么框架层要提示防止内存泄露？因为Looper会一直循环

#### MessageQueue 的Barrier是什么鬼

#### ActivityThread是个好东西
