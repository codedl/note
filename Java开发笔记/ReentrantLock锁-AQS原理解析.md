ReentrantLock是Java中常见的可重入锁，这几天被面试官问疯了，干脆看下他的底层源码，了解他的原理。
先看下简单的用法，就是能够先`lock()`获取锁，如果能成功拿到锁就可以继续往下执行，如果发现有其他线程已经拿到锁就阻塞，等待其他线程释放锁，然后再获取到锁以后往下执行，最后一定要`unlock()`释放锁。
```java
    AtomicInteger integer = new AtomicInteger();

    ReentrantLock lock = new ReentrantLock();
    for (int i = 0; i < 10; i++) {
        new Thread(() -> {
            try {
                lock.lock();
                System.out.println("线程:"+integer.incrementAndGet());
            }finally {
                lock.unlock();
            }
        }).start();
    }
```
现在开始撸源码了，这里的逻辑比较复杂，要花点时间耐心地去看。以非公平锁为例，首先以cas得方式将属性`private volatile int state;`得值从0设置为1，state=0说明当前锁没有被其他线程获取，能够设置成功则可以获取到锁，之后设置当前线程为锁得持有者即可，属性`private transient Thread exclusiveOwnerThread;`表示获取锁得线程实体。如果不能获取到锁，则`acquire(1)`将当前线程添加到队列挂起。
```java
        final void lock() {
            //直接获取锁
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);//没有获取到进一步处理
        }
```
先判断是不是当前线程重复获取锁，然后加入队列挂起。
```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
这里有种情况就是当前线程重复得获取锁，这是在`protected final boolean tryAcquire(int acquires)`方法中实现得。先获取锁得状态，`if (c == 0)`说明锁没有被其他线程获取，此时判断队列前是否有其他线程，没有得话获取锁即可；如果`else if (current == getExclusiveOwnerThread()`，则说明获取锁得线程就是当前线程，则重复获取。
```java
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            //这里会再次获取锁，因此在此期间可能持有锁的线程释放锁了；没有其他线程获取锁
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //获取到锁的正是当前线程自身
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
这里就是判断此线程之前是否还有其他线程在等待获取锁，如果有返回true。
```java
    public final boolean hasQueuedPredecessors() {
        Node t = tail; // 尾节点
        Node h = head; // 头节点
        Node s;
        // 初始化时h == t表示队列为空
        // h != t说明队列中有其他线程
        // h != t && (s = h.next) == null 说明队列不为空且正在有其他线程获取锁成功，当其他线程获取锁成功时h.next=null
        // h != t && s.thread != Thread.currentThread() 队列中第二个节点不是本线程，s是队列中第二个节点
        // 总之返回true说明要么有线程排在自己前面，要么有其他线程已经加锁成功
        return h != t && ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```
无法获取到锁时则将当前线程加入队列。
```java
    private Node addWaiter(Node mode) {
        // 使用node封装当前线程
        Node node = new Node(Thread.currentThread(), mode);
        // 将包含当前线程的节点node加入队列尾
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        //初始化队列
        enq(node);
        return node;
    }

    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            //(t == null)说明队列为空
            if (t == null) { // 初始化时队列中第一个节点为空节点，即获取到锁的节点为空节点
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {//把最新的节点加入队列尾
                node.prev = t; //最尾的节点的prev指向原最后一个节点
                if (compareAndSetTail(t, node)) {
                    t.next = node;//原来最后一个节点指向最新的节点
                    return t;
                }
            }
        }
    }
```
队列中的第二个节点是可以去竞争锁的，如果竞争成功则移除第一个节点，否则将自身挂起。
```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                //获取前一个节点
                final Node p = node.predecessor();
                //如果前一个节点是头节点，说明自身为第二个节点，则竞争锁
                if (p == head && tryAcquire(arg)) {
                    //设置当前节点为队列中第一个节点
                    setHead(node);
                    //将第一个节点next置为null，则第一个节点不会跟其他对象没有引用关系会被gc回收
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                //移除队列中cancel状态的节点，同时找到能够唤醒自身的节点，将自己挂起
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
遍历整个队列，找到waitStatus(`volatile int waitStatus;`)为Node.SIGNAL的节点，然后将当前线程挂起。
```java
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        //waitStatus在初始化和释放锁的时候都会置为0
        int ws = pred.waitStatus;
        //当前一个节点为Node.SIGNAL时返回，之后为Node.SIGNAL的节点会唤醒当前线程
        if (ws == Node.SIGNAL)
            return true;
        if (ws > 0) {
            //一直往前移动，直到pred.waitStatus<=0，如果pred.waitStatus > 0则会从队列中移除
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            //前一个节点设置为Node.SIGNAL，用于唤醒本线程
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
    //挂起当前线程，当其他线程释放锁时会调用` LockSupport.unpark(s.thread);`唤醒自身
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
```
总结下lock()加锁的过程：
1. 直接去获取锁，如果能获取成功则往下执行
2. 再次尝试获取锁，因为有可能其他线程在第一步操作时释放了锁，想的还是很周到的，但是条件时队列中没有其他线程在自己前面并且没有其他线程正在尝试获取锁
3. 判断是否为重入地获取锁，是的话state+1
4. 获取不到锁了就得把当前线程加入队列，首先初始化链条，此时头节点head和尾节点tail都会被初始化空的Node对象，然后用Node封装当前线程加入到队列尾部
5. 判断是否为队列中第二个线程，是的话就去竞争锁，不是的话就将前一个线程节点设置为Node.SIGNAL，用于唤醒自己

再看下unlock()吧，关键就是释放锁，唤醒队列中等待锁的线程。
```java
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
每次调用`tryRelease()`都会将state-1，如果为0则可以释放锁，其他线程就能获取到锁啦。
```java
        protected final boolean tryRelease(int releases) {
            //每次-1
            int c = getState() - releases;
            //必须是获取锁的线程释放锁
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                //释放锁
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            //state为0释放锁成功返回true
            return free;
        }
```
如果锁释放成功则唤醒队列中其他线程，找到waitStatus<0的线程，调用`LockSupport.unpark(s.thread)`唤醒。
```java
    private void unparkSuccessor(Node node) {
        //将当前线程节点waitStatus置为0
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
        //遍历链表，找到离自己最近且waitStatus <= 0的线程，然后唤醒
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        //唤醒
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```