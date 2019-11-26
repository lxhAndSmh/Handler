#### 概述 
`Handler` 是 `Android` 消息机制中的最重要的一部分，常用于主线程与子线程之间的消息通信。
#### 简单的使用场景  
在子线程发送消息，在主线程更新 UI。
```
class SelingMenuActivity : AppCompatActivity(), Handler.Callback {
    override fun handleMessage(msg: Message?): Boolean {
        // 通过回调接口接收消息，如果返回 true，则将消息拦截，不会发送给 override fun handleMessage(msg: Message?) ，具体原因，我会在 Handler 的介绍中分析
        var count = msg?.arg1
        text?.text = "Callback count = $count"
        return true
    }

    private val TAG = "SelingMenuActivity"

    private var demoHandler = DemoHandler(this, this)

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_seling_menu)

        demoHandler.post{ Log.d(TAG ,"--post---")}
        var scheduledExecutorService: ScheduledExecutorService = Executors.newScheduledThreadPool(2)
        scheduledExecutorService.submit{
            var count = 10
            while (count > 0) {
                var message = demoHandler.obtainMessage()
                message.arg1 = count
                demoHandler.sendMessage(message)
                count--
                Thread.sleep(1000)
            }
        }
    }

    private class DemoHandler : Handler {
        // 内部持有 Activity
        private var mReference: WeakReference<SelingMenuActivity>

        constructor(callback: Callback?, activity: SelingMenuActivity) : super(callback) {
            mReference = WeakReference(activity)
        }

        override fun handleMessage(msg: Message?) {
            var count = msg?.arg1
            var activity = mReference.get()
            activity?.text?.text = "count = $count"
        }
    }
}
```

#### 四个主要的类
- `Handler` 用于发送消息和接受消息
- `Looper` 相当于一个轮询器，一直去 `MessageQueue` 获取消息。一个线程只有一个 `Looper`
- `Message` 消息体，相当于信使
- `MessageQueue` 消息队列，用于存放 `Message`
##### 基本原理
`Handler` 发送 `Message`,并将其保存到 `MessageQueue` 中，通过 `Looper` 轮询消息队列，从 `MessageQueue` 取出 `Message`
##### Handler
`Handler` 的构造方法如下,所有的构造方法最终都会调用该方法
```
    public Handler(Callback callback, boolean async) {
        //......省略
        // 获得 Looper 的引用
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        // 获得 MessageQueue 的引用
        mQueue = mLooper.mQueue;
        // Handler 传入 Callback 回调接口
        mCallback = callback;
        mAsynchronous = async;
    }
```
通过 `Handler` 的构造方法，可以知道创建 `Handler` 的时候会持有 `Looper` 和 `MessageQueue` 的引用。`Handler` 发送消息，一般使用 `post(Runnable)` 、`sendMessage(Message)`,发送消息最终都会执行如下该方法。可以看出，这个方法将消息按照一定的顺序放入到消息队列中，其中的 `msg.target = this;` 表明这个消息是由哪个 `Handler` 发送的，最后处理消息的时候就知道哪个 `Handler` 处理该消息。
```
 private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        return queue.enqueueMessage(msg, uptimeMillis);
    }
```
`Handler` 接收消息,`Looper` 通过 `loop()` 方法将轮询到的消息，执行 `              msg.target.dispatchMessage(msg);` 将消息分发给该 `Handler` 处理。通过 `dispatchMessage(Message msg)` 可以看出，当 `Message` 的 `callback`（其实就是一个 `Runnable`）不为空时，则由 `Message` 的 `Runnable` 处理消息，如果它为空且 `Handler` 的回调接口 `mCallback` 不为空时，则由其将消息回调处理，最后才是执行 `Handler` 自己的 `handleMessage` 方法。下面我们就分析一下这三种情况：
   ```
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
    /**
     * Message 自己的 Runnable
     */
    private static void handleCallback(Message message) {
        message.callback.run();
    }
    /**
     * Handler 的回调接口
     */
    public interface Callback {
        /**
         * @param msg A {@link android.os.Message Message} object
         * @return True if no further handling is desired
         * 返回 true 则将消息拦截，flase Handler 自己接受消息 handleMessage(Message msg)
         */
        public boolean handleMessage(Message msg);
    }
    /**
     * Handler 自己接收消息
     * Subclasses must implement this to receive messages.
     */
    public void handleMessage(Message msg) {
    }
   ```
- 方式一：`post(Runnable)` 更新 UI,通过如下源码，可以看到，`post` 传入的 `Runnable` 赋值给了 `Message`。
```
    /**
     * Causes the Runnable r to be added to the message queue.
     * The runnable will be run on the thread to which this handler is 
     * attached. 
     * 该 Runnable 会在 Handler 所在的线程中执行。这就是为什么 post 中传入的 Runnable 可以 
     * 更新 UI 了
     */
    public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }
    
    private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }
```
- 方式二：通过构造方法 `Handler(Callback callback)` 给 `Handler` 回调接口 `Callback` 赋值，此时 `Handler` 中的 `mCallback` 不为空，则将数据回调，处理消息。
```
    /** 构造方法*/
    public Handler(Callback callback) {
        //最终调用构造方法 Handler(Callback callback, boolean async) 
        this(callback, false);
    }
```
- 方式三：`Handler` 自己接收消息，当 `Message` 的 `callback` 且 `Handler` 的回调接口 `mCallback` 为空（或 mCallback 的回调方法返回 `false`）时，` public void handleMessage(Message msg)` 自己接收消息。

#### Looper
`Looper` 只有一个构造方法，如下
```
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
    /**
     * Return the Looper object associated with the current thread.  Returns
     * null if the calling thread is not associated with a Looper.
     */
    public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```
我们在 `Handler` 中看到，通过 `mLooper = Looper.myLooper();` 获取 `Looper` 的引用，但是却看不到 `Looper` 创建，其实 `Android` 在应用的启动入口 `ActivityThread` 类 的 `main` 方法已经帮我们创建好 `Looper` 的对象了，这就是 `Looper` 的入口
```
public static void main(String[] args) {
    //......省略
    // 创建主线程的looper
    Looper.prepareMainLooper();
    //...    
    if (sMainThreadHandler == null) {
        sMainThreadHandler = thread.getHandler();
    }
    // ......省略
    // 开始让轮询器工作起来
    Looper.loop();
}

   // Looper 类的方法
   public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            //可以看出一个线程中只能有一个 Lopper
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        // 创建 Looper 对象并放入 sThreadLocal
        sThreadLocal.set(new Looper(quitAllowed));
    }
```
下面我们再看一下消息轮询 `Looper.loop()`，获取 `MessageQueue` 的对象，可以看到通过死循环，一直获取消息 `queue.next()`，并分发消息 `msg.target.dispatchMessage(msg)`。
```
/**
  * Run the message queue in this thread. Be sure to call
  * {@link #quit()} to end the loop.
  */
public static void loop() {
   final Looper me = myLooper();
   final MessageQueue queue = me.mQueue;
   //...省略
   for (;;) {
      //轮询消息队列中消息
      Message msg = queue.next(); // might block
      //最重要的方法，分发消息，回到了上面 Handler 的 dispatchMessage(msg);
      msg.target.dispatchMessage(msg);
      //...省略
      // 回收消息
      msg.recycleUnchecked();
   }
}
```

#### Message
`Message` 既是一个消息的实体类，又是一个工具类（用于创建消息池）,重要的几个属性
```
// 标记类型的整形参数
int what;
// 整数参数1
int arg1; 
// 整数参数2
int arg2;
// 消息携带的参数对象
Object obj;
// 数据
Bundle data;
// 指向需要被处理的handler对象，或者说管理的handler
Handler target;
// 任务对象
Runnable callback;
// 链表，指向下一个消息
Message next;
```
`Message` 只有一个空的构造方法，在前面，我们可以看到 `Handler` 通过 `handler.obtainMessage()` 调用 `{return Message.obtain(this);}` 获得 `Message` 对象，我们看一下 `Message` 的 `obtain(Handler h)` 方法，如果消息池子 `sPool` 中有消息，直接拿来用， 没有的时候才去创建新的对象，避免创建多余的对象，获取 `Message` 对象后，并将需要被处理的 `Handler` 对象赋值给 `Message` 的 `target`
```
     public Message() {}
     public static Message obtain(Handler h) {
        // 或许 Message 对象
        Message m = obtain();
        // 将要被处理的 Handler 对象，赋值给 Message 的 target,
        m.target = h;

        return m;
    }
    
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
在 `Looper.loop()` 方法中，可以看到消息被 `Handler` 处理后，会通过 `msg.recycleUnchecked();` 回收消息，重复使用 `Message` 对象。`next = sPool;sPool = this;` 其实是一个链表对象，sPool 是一个头指针，指向首个消息对象，第一个消息的 `next` 属性又指向了下一个消息对象，消息池的最大个数是50个，消息在使用完之后，就返回给了消息池；这里所说的消息池，其实就是每个消息对象都被自己的下一个消息所引用，所以不会被内存回收，达到了复用的目的。
```
    /**
     * Recycles a Message that may be in-use.
     * Used internally by the MessageQueue and Looper when disposing of queued Messages.
     */
    void recycleUnchecked() {
        // Mark the message as in use while it remains in the recycled object pool.
        // Clear out all other details.
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = -1;
        when = 0;
        target = null;
        callback = null;
        data = null;
        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
```

#### MessageQueue

`MessageQueue` 的构造方法如下，唯一一个参数，用来控制是否允许终止获取消息。
```
    MessageQueue(boolean quitAllowed) {
        mQuitAllowed = quitAllowed;
        mPtr = nativeInit();
    }
```
主要的方法 `enqueueMessage` ,在 `Handler` 发送消息的时候，通过 `MessageQueue` 的对象调用该方法，将消息存放到消息队列中，从下面的主要代码中，可以明白，利用咱们上面分析过的 `Message` 消息池的方式，将消息存放起来。
```
    boolean enqueueMessage(Message msg, long when) {
        // ...
        synchronized (this) {
        //...
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
        //...
        }
        return true;
    }
```
另一个比较重要的方法 `Message next() {}`，在执行 `Loop.loop()` 轮询消息时，就是通过 `MessageQueue` 的 `next()` 方法读取消息的。