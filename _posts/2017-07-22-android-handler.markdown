---
layout:     post
title:      "Android消息机制"
subtitle:   "学习总结"
date:       2017-07-22 12:00:00
author:     "溜大虾"
header-img: "img/bg-handler.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Android
    

---

## 1.消息机制

 Android消息机制，其实也就是Handler机制，主要用于UI线程和子线程之间交互。众所周知，一般情况下，出于安全的考虑，所有与UI控件的操作都要放在主线程及UI线程，而一些耗时操作应当放在子线程中。当在子线程中完成耗时操作并要对UI控件进行操作时，就要用Handler来控制了。在这一套消息机制中，首先要明确这样几个概念：  

- Handler：消息的控制器  
- Message：消息的载体  
- MessageQueue:存放消息
- Looper：控制消息队列的循环   

（经评论区小伙伴指正，MessageQueue严格意义上说并不是一个存放消息的队列，Message本身通过next一个一个的连在一起，通过单链表形成了一个队列，MessageQueue只是可以对这个队列进行部分操作，比如入队）

下面一段简单的代码就展示了Handler的用法：  

```
private Handler handler = new Handler(){
    @Override
    public void handleMessage(Message msg) {
         super.handleMessage(msg);
         textView.setText("对UI进行操作");
    }
};
@Override
protected void onCreate(Bundle savedInstanceState){
       super.onCreate(savedInstanceState);
       setContentView(R.layout.activity_main);
       textView = (TextView) findViewById(R.id.mytv);
       new Thread(new Runnable() {
           @Override
           public void run() {
               //模拟耗时操作
               SystemClock.sleep(3000);
               handler.sendMessage(new Message());
           }
       }).start();

   }


```

 可以看到，在子线程中通过发送一个消息 Message，然后再由Handler处理接收到的消息，下面我将一步一步看sdk的源码了解他的原理。  

## 2.发送消息：sendMessage

 跟踪sendMessage()/sendEmptyMessage()：  

```
 public final boolean sendMessage(Message msg){
        return sendMessageDelayed(msg, 0);
  }

  public final boolean sendEmptyMessage(int what){
        return sendEmptyMessageDelayed(what, 0);
  }

public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
    Message msg = Message.obtain();
    msg.what = what;
    return sendMessageDelayed(msg, delayMillis);
}


```

 可以看到，无论是sendMessage() 还是sendEmptyMessage()，最后都会调用sendMessageDelayed()方法。不同之处在于，sendMessage()方法接受的是一个Message对象，然后将这个对象传给sendMessageDelayed(),而sendEmptyMessage()需要的是一个int值what，然后通过Message.obtain()方法得到一个Mesage对象，再将what值赋给他，最后传给sendMessageDelayed()。类似的还有sendMessageAtFrontOfQueue()和sendEmptyMessageAtTime()等方法，总之就是需要一个Message对象并将他传给sendMessageDelayed();    这里有两个点需要注意一下，第一点，what值是干什么的？第二点，new出来的Message对象和调用Message.obtain()方法得到的对象有什么区别呢？  这是对what的描述：

```
 /**
     * User-defined message code so that the recipient can identify
     * what this message is about. Each {@link Handler} has its own name-space
     * for message codes, so you do not need to worry about yours conflicting
     * with other handlers.
     */
    public int what;


```

 可见，what就是一条消息的消息代码，由于不同的handler都有自己的命名空间，所以我们不必担心会引起冲突。  再来看看obtain():

```
 /**
 * Return a new Message instance from the global pool. Allows us to
 * avoid allocating new objects in many cases.
 */
public static Message obtain() {
    synchronized (sPoolSync) {
        if (sPool != null) {
            Message m = sPool;
            sPool = m.next;
            m.next = null;
            m.flags = 0; // clear in-use flag
            sPoolSize--;
            return m;
        }
    }
    return new Message();
}


```

原来android已经为我们定义好了一个全局的Message池，这个池是一个链表型数据结构，通过obtain()方法可以从链表头取出一个Message对象。这两个小问题解决完了，继续看 sendMessageDelayed():

```
public final boolean sendMessageDelayed(Message msg, long delayMillis){
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}


```

可以看到，对消息的时间做了一下修正，然后传给了sendMessageAtTime()，这里为时间加上了一个SystemClock.uptimeMillis()，也就是从这里开始，采用了系统的准确时刻而不是之前的延时多久。接下来看sendMessageAtTime():   

```
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


```

在这里首先获取了Handler中的MessageQueue对象，若不为空，说明一切正常，接下来就要将这个Message插入到MessageQueue中。  

```
private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }


```

这里将这个message的目标(target)指向了该handler自己(this)，然后调用MessageQueue的enqueueMessage()方法进行了消息的插入操作。  

```
boolean enqueueMessage(Message msg, long when) {
        if (msg.target == null) {
            throw new IllegalArgumentException("Message must have a target.");
        }
        if (msg.isInUse()) {
            throw new IllegalStateException(msg + " This message is already in use.");
        }

        synchronized (this) {
            if (mQuitting) {
                IllegalStateException e = new IllegalStateException(
                        msg.target + " sending message to a Handler on a dead thread");
                Log.w(TAG, e.getMessage(), e);
                msg.recycle();
                return false;
            }

            msg.markInUse();
            msg.when = when;
            Message p = mMessages;
            boolean needWake;
            if (p == null || when == 0 || when < p.when) {
                // New head, wake up the event queue if blocked.
                msg.next = p;
                mMessages = msg;
                needWake = mBlocked;
            } else {
                // Inserted within the middle of the queue.  Usually we don't have to wake
                // up the event queue unless there is a barrier at the head of the queue
                // and the message is the earliest asynchronous message in the queue.
                needWake = mBlocked && p.target == null && msg.isAsynchronous();
                Message prev;
                for (;;) {
                    prev = p;
                    p = p.next;
                    if (p == null || when < p.when) {
                        break;
                    }
                    if (needWake && p.isAsynchronous()) {
                        needWake = false;
                    }
                }
                msg.next = p; // invariant: p == prev.next
                prev.next = msg;
            }

            // We can assume mPtr != 0 because mQuitting is false.
            if (needWake) {
                nativeWake(mPtr);
            }
        }
        return true;
    }


```

如果熟悉数据结构的话，可以很清楚的看到，这个所谓的消息队列MessageQueue是一个链表，将消息插入消息队列就是一个简单的对链表进行插入的过程。首先会对链表头指针做判断，如果为空，那么就把当前消息插入到链表头部，如果链表不为空，那么比较一下当前消息的执行时间，若时间小于头指针所存储的消息，那么也要将他插入到链表头部。若以上条件都不满足，那么就要对链表进行一个遍历，找到适当的位置并插入。  

## 3.取出消息：Looper

Looper负责取出消息然后把消息交给目标handler处理。那么他是怎么工作的呢，来看看他的源码：首先，Looper的入口是prepare()方法：

```
public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }


```

调用prepare()方法，会new 一个Looper对象把他传给sThreadLocal.set()方法，那么先来看看这个方法是何用：  

```
/**
     * Sets the current thread's copy of this thread-local variable
     * to the specified value.  Most subclasses will have no need to
     * override this method, relying solely on the {@link #initialValue}
     * method to set the values of thread-locals.
     *
     * @param value the value to be stored in the current thread's copy of
     *        this thread-local.
     */
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }


```

他将一个数据保存在了当前线程中。那么刚才就是将一个Looper对象保存在了调用方法的当前线程中。再来看看Looper的构造方法：  

```
private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }


```

还记得默认是给构造方法传了一个值为true的boolean。在这个构造方法中，先创建了一个消息队列，保存起来，然后又获取了当前线程，并保存起来。综合一下，就是在创建Looper的时候将当前线程、一个消息队列和该Looper对象关联起来了。创建好了Looper，接下来就是开启了。开启方法是loop():  

```
**
     * Run the message queue in this thread. Be sure to call
     * {@link #quit()} to end the loop.
     */
    public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;

        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            final Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }

            final long traceTag = me.mTraceTag;
            if (traceTag != 0 && Trace.isTagEnabled(traceTag)) {
                Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
            }
            try {
                msg.target.dispatchMessage(msg);
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }

            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }

            msg.recycleUnchecked();
        }
    }


```

代码太长，我只看关键部分：  

```
final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;


```

首先通过myLooer()从当前线程中获取到了刚刚保存起来的Looper对象，然后检查是否为空。如果为空，直接抛出异常。因此，我们要想使用Looper，就要先调用prepare()方法创建一个Looper对象保存在当前线程，然后才能在loop()方法中获取到。之后进入了一个死循环中：  

```
for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
            //......
            try {
                msg.target.dispatchMessage(msg);
            } finally {
                if (traceTag != 0) {
                    Trace.traceEnd(traceTag);
                }
            }
            //......
            msg.recycleUnchecked();
        }


```

在这个循环中，会不断的从queue中获取msg，然后调用msg的target的dispatchMessage()方法，queue通过名字可以看出来是一个队列，即消息队列，暂不深究。这里有一个问题，msg的target是什么？dispatchMessage()做了什么？    

```
/*package*/ Handler target;


```

跟踪进来可以看到，target其实就是一个Handler对象，那么dispatchMessage()也即Handler的方法了：  

```
/**
     * Handle system messages here.
     */
    public void dispatchMessage(Message msg) {
        if (msg.callback != null) {
            handleCallback(msg);
        } else {
            if (mCallback != null) {
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);
        }
    }


```

首先，若msg的callback不为空就调用handleCallback()方法：  

```
private static void handleCallback(Message message) {
        message.callback.run();
    }


```

否则，先判断自己的callback若不为空，则将msg传给mCallback的handleMessage()：  

```
public interface Callback {
        public boolean handleMessage(Message msg);
    }


```

最后实在不行才回去调用自己的handleMessage方法：  

```
/**
     * Subclasses must implement this to receive messages.
     */
    public void handleMessage(Message msg) {
    }


```

这个方法是要自己覆盖的（不然一个空方法调用个锤子）。那么现在很请除了，每一条消息关联了自己的Handler对象，然后把自己交给他去处理。还记得前面发送消息时有一行代码是Handler将target指向了自己吗？对，就是在那里进行了关联。一切都分析完了（好像很简单的样子？），总结一下：  

- Message是消息对象，表示要具体做些什么  
- 创建Message对象建议用obtain()方法，这样是从一个消息池中不断的取出消息来使用，避免过多的内存分配    
- Handler首先通过sendMessage()方法把消息发送出去   
- Handler发送消息最终会由MessageQueue进行一个入队的操作（消息队列即链表），与此同时会将该消息的target指向该Handler，Handler和Message的联系就在这里建立起来  
- Looper负责不断的从消息队列中取出消息来处理  
- 使用Looper首先要调用prepare()方法将创建的Looper对象保存在当前线程中，之后才能通过Loop()方法取出，Looper和线程、消息队列的联系在这里建立  
- 对于消息的处理，还是要交给Handler来做，即取出消息的target所指向的Handler，交给他处理  
- 主线程即UI线程在一开始创建时就已经创建并开启了Looper，所以我们在主线程中使用Handler时就已经和主线程、消息队列有了联系，就不用再手动调用loop()了    

以上就是我在学习Android消息机制及Handler时阅读SDK源码的过程，欢迎多多交流