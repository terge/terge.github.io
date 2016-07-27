---
title: 说一下Handler的原理
date: 2016-07-16 15:33:31
tags:
---

## Part1 : Handler原理概述

####  Handler在解决什么问题?
“handler 是为了解决线程间通讯的问题而设计的”  


对于这个回答我是不满意的：　　

		１．解决线程通讯一个通用的回调就可以，何必整真么多事出来
			线程通讯是终极目的，在设计这个过程中还包括消息队列排队、循环读取消息、阻塞等待的逻辑　　
			学习它如何优雅的设计这个通讯方案，是值得我们去深挖的
		２．handler给我们的印象是更新UI用的，如何与线程通讯联系起来
			因为大多数情况我们是用来是子线程来通知主线程来操作的，而一般是通知主线程是更新UI　　
			所以给我们造成这样一种误解："handler是更新UI"　　
			其实handler可以用来跟随意两个子线程通讯




####  Handler提供的接口
handler主要提供这两个功能（其他一些功能都是围绕这两个转的）


    1. 发消息 	   sendMessageAt()
    2. 处理消息 	 handleMessage()

这里对应就有两个疑问了:

	1. 它的消息发给谁？
	2. 消息又是由谁来消费的?

看看我们熟悉的四基友：Thread - Looper - MessageQueue - Handler
了解他们是如何协同工作的才能回答上述两个疑问

![Handler-flow](http://oaumghlfk.bkt.clouddn.com/handler.png?watermark/2/text/dGVyZ2UubWU=/font/5a6L5L2T/fontsize/500/fill/I0E3MDYwNg==/dissolve/54/gravity/SouthEast/dx/10/dy/10)


1. ThreadA持有handler，此handler是使用线程B的Looper来实例化的，发送的消息是将其入队到ThreadB.Looper.MessageQueue中
2. MessageQueue中的消息会被与之对应的Looper逐个取出来
3. 执行Looper中的消息的主体是与Looper对应的ThreadB
4. ThreadB只是调用message.target.dispatchMessage
5. 这个target兄便是ThreadA的handler,这样消息在人间这么走一遭，逻辑上又会执行到Handler.dispatchMessage
6. 但是这个dispatchMessage是ThreadB来调用的，也就是此msg的逻辑(也就是我们覆写的handlerMessage)是在ThreadB上执行的

他们的关系，Handler实例化需要一个Looper,Looper内部维护一个MQ,hander的消息就是入队到这个Looper的MQ中，一个线程最多只有一个Looper，线程会从Looper中取出消息来执行.
我们一般使用Handler时候也是这样，在主线程实例化Handler，相当于使用主线程的Looper来实例化Handler,将此Handler的引用给到子线程，在需要时候子线程通过handler发消息，就会将消息入队到主线程的Looper中，从而在主线程中执行.
主线程的Looper是在主线程实例化时候就已经准备好的，所以我们不需要调用Looper.prepare(),当我们需要子线程的Looper来实例化Handler的时候，就需要主动调用Looper.prepare().

再来回答那两个问题：
1. 它的消息发给谁？
	Handler实例化时候用了哪个Lopper,消息就会发送到那个looper对应的MessageQueue中  
2. 消息又是由谁来消费的?
	这个Looper对应的线程是哪一个，消息就会被那个线程消费

## Part2 : 源码分析
以上是结论，那么我们现在从源码角度来跟着Message，去人世间走一遭　　
注意：Handler中所有的发消息接口如下图选中的postxxx和sendxxx

![Handler-api](http://oaumghlfk.bkt.clouddn.com/handler-api.png?watermark/2/text/dGVyZ2UubWU=/font/5a6L5L2T/fontsize/500/fill/I0E3MDYwNg==/dissolve/54/gravity/SouthEast/dx/10/dy/10)　　
这些接口都只是在封装Message，最终都是调用到sendMessageAtTime，所以我们只看sendMessageAtTime即可

####   Handler.sendMessageAtTime()
```java
//Handler.sendMessageAtTime 最终将消息入队列
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

//继续看Handler.enqueueMessage
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}

```
OK，Handler.sendMessage 执行的逻辑是MessageQueue.enqueueMessage(msg,time)
那么这个MQ（以下都将MessageQueue简称为MQ）是哪里来的？

```java
//Handler
final MessageQueue mQueue;
final Looper mLooper;
...
public Handler(Callback callback, boolean async) {
		...
    mLooper = Looper.myLooper();
    if (mLooper == null) {
        throw new RuntimeException(
            "Can't create handler inside thread that has not called Looper.prepare()");
    }
    mQueue = mLooper.mQueue;
    ...
}

public Handler(Looper looper, Callback callback, boolean async) {
    mLooper = looper;
    mQueue = looper.mQueue;
    ...
}


```


		呐，还有我们熟悉的异常：
		"Can't create handler inside thread that has not called Looper.prepare()"　　
		Handler 实例化时候是需要一个Looper的，如果没传递进来，就会直接获取当前线程的Looper,如果当前线程未开启Looper,就死给你看  所以出现此问题的处理方式：要么是直接传递Looper进来，要么是在实例化时候调用Looper.prepare(),然而Ｌooper.prepare()只能调用一次，如果已经prepare过了，再调用会抛异常,处理方式如下：
		```java
		if(Looper.myLooper() == null){
			Looper.prepare();
		}
		```

Handler 在实例化时候就会初始化Looper和MQ  
上面Handler.sendMessage,msg最终是入到队列里了，这个MQ就是Looper.mQueue
并且Looper和MQ都是final的，Handler构造完之后就不可改变
到此Handler这一边的逻辑就告一段落，我们继续梳理MQ.equeue之后的逻辑
####   MessageQueue.enqueueMessage()

```java
//MessageQueue.enqueueMessage
boolean enqueueMessage(Message msg, long when) {
	//这个target就是handler引用
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
	//消息不能重复使用，有需要记得clone一份
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }

    synchronized (this) {
	//队列已经退出的逻辑处理
	//mQuitting状态标识是在MQ.quit()方法中被设为true的
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }
	//消息是在这里设置成已经消费了
        msg.markInUse();
	//时间是在入队时候指定的，之前手动给消息赋时间没有意义
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
	//空队列 ，或者非延时消息，或者消息比对头消息还早，直接插队在队头
        if (p == null || when == 0 || when < p.when) {
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
		//MQ中的消息是根据msg.when来排序的
		//根据msg.when给新来的Msg找个合适的位置插进去
            for (;;) {
                prev = p;
                p = p.next;
		//1. p == null，说明找到队列尾部了还是没找到合适位置
		//2. when < p.when 说明找到一个比自己还要晚展示的消息
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
		//插入msg
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }

        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}

```
啥也不说了，都在酒里了，哦不，在注释里
上面说了具体的Message入队的细节，还是没有讲入队之后的事情
那接下来就要去MQ对应的Looper里面看实现了  
####    Lopper.loop()

```java
//Lopper.loop()
public static void loop() {
    final Looper me = myLooper();
    if (me == null) {
        throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
    }
    final MessageQueue queue = me.mQueue;

    Binder.clearCallingIdentity();
    final long ident = Binder.clearCallingIdentity();

    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            return;
        }
				...
        msg.target.dispatchMessage(msg);
				...
        msg.recycleUnchecked();
    }
}

```
Looper.loop执行逻辑是循环读取MQ中的mesage，然后执行message.target.dispatchMessage
而这个message.target就是在enqueue时候制定的handler
那，我们再跟踪到Handler.dispatchMessage去看看  
####    Handler.dispatchMessage()

```java
//Handler.dispatchMessage
public void dispatchMessage(Message msg) {
	//此处的msg.callback也就是postRunanable发送的runable
    if (msg.callback != null) {
        handleCallback(msg);
    } else {
			//注意这个mCallback不要与msg.callback混了，这个是handler的callback.后面在讲HandlerThread时候会讲到这里
        if (mCallback != null) {
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
				//呐，回到了我们覆写的handleMessage方法
        handleMessage(msg);
    }
}

```


loop这里有几点需要注意：
	1. loop方法是个死循环：for (;;) {}
	2. 只有取得的msg为空时候才会退出死循环，那这个消息什么时候为空？（读者自行跟踪下，答对我送你一个惊喜）



## Part3 扩展

####  ThreadLoacal是什么鬼
每个讲Handler都要提到theadLocal,这个与主题没有一毛钱关系，将其理解成Thread一个Map就好，想详细了解的自行Google,个人觉得这东西与Handler原理扯不上关系

####  MessageQueue是怎么组织的？BlockQueue?
具体的实现都在native中 ？暂且当做BlockQueue来理解//todo
enqueueMessage
异常情况(msg.target == null  || msg.isInUse() )
sendMessage会自动设置target,//什么场景下修改会为空？？
msg.isInUse会在入队列时候修改，所以在重复使用时候可以通过Message.obtain(msg)来客隆一份数据一样但是没有使用过的Message来使用

此队列并非先进先出队列，而是一个LinkedQueue,在enqueueMessage,会根据Message.when，将其插入到合适的位置，队列中的消息是按照when排序的


####  Message.when
MQ是按照Message.when来进行排序的，消息入队时候会按照当前消息要展示的是事件来放到合适的位置，如果直接指定sendMessageAt的时间为０，就会直接放在队头，Handler提供的接口 sendMessage() postRunnable() sendMessageAtFrontOfQueue...这些都只是指定了sendMessageAt(msg,0),这个时间０就会赋值给message.when

但是when可以修改么？
when是package访问权限(还好不是final)，是否可以通过新建一个与其包名相同的类来修改？
when值来自 sendMessageAt的那个时间，在enqueue时候在MessageQueue里面被赋值，入队之后这个值就没有意义了，所以我们修改它也没有意义

####  Message.target
Handler.obtain系列会将Message.target赋值为this
而Handler.postRunnable 获取的Message没有赋值target
但是无论哪种方式，最终会在enqueueMessage中将target赋值为Handler.this
也就是是说在使用哪个Handler发送消息，最终一定是进了他的Looper
但是一旦enqueueMessage后，处理逻辑就以及脱离了Handler的控制了，执行的结果都只会发送到Message.target中


####  sendMessage与postRunable有什么区别

handler.sendMessage 和handler.postRunanable其实质上最后都是调用到handler.sendMessageAt(msg,time)，runnable只是赋值给了msg.callback,也是发送一个消息　　
如果你愿意，完全可以实例化一个Ｍessage，将runable赋值给msg.callback，再通过handler.sendMessage来发送runnable  
注:这种方式可以用于统一处理，比如switch-case中，有的case是sendMessage,有的是postRunanable,这样就可以在case里面只处理Message，对于postRunanable的情况也使用message来处理，在switch-case的外层统一调用sendMessage来发送

需要注意的是，在消费Message时候，是优先处理runnable,这个在Part2最后部分已经有注释出来，这里不再重复

#### Handler消费消息时候处理的优先顺序


## Part4 思考
####  如何停止Looper
Looper在loop的时候，如果取到一个Message为null，就会自行跳出，但是MessageQueue只有在队列为空和已经退出时候才会给Looper返回null. 不能通过发送null消息来停止，
同时Looper本身也提供了quite api来退出循环，原理是调用MQ.quite，在下一个next时候MQ返回给Looper一个null,这样Looper就自行结束

在Looper中会调用msg.target.dispatchMessage(msg),这样又将消息返回给handler来执行了，而此时消息执行是looper中调用的，也就是消息执行是在looper所在的线程调用的，这样此消息也就在此线程被消费
同时会调用msg.resycleUnchecked()回收此对象

####  消息执行的优先级
- 会先处理msg.callback也（post进来的runnable就是设置到callback中）
- 如果handler有设置callback，消息会发送到callback的handlermessge()中消费
- 如果没设置就在本handler.handlerMessage()中消费
- 而一般情况下我的处理逻辑都在handlerMessage中，这样消息就又回来了
####  聊一聊HandlerThread
- http://blog.csdn.net/qq_23547831/article/details/50936584

####  MainLooper ActivityThread.man() 如何做到一直待机


####  为什么框架层要提示防止内存泄露？因为Looper会一直循环

####  MessageQueue 的Barrier是什么鬼

####  ActivityThread是个好东西
ActivityThread不是Thread
