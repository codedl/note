ReentrantLock是Java中常见的可重入锁，通过底层实现源码可以了解其原理。
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
现在开始撸源码了，这里的逻辑比较复杂，要花点时间耐心地去看。
+ 非公平锁  
以非公平锁为例，首先以cas得方式将属性`private volatile int state;`得值从0设置为1，state=0说明当前锁没有被其他线程获取，能够设置成功则可以获取到锁，之后设置当前线程为锁得持有者即可，属性`private transient Thread exclusiveOwnerThread;`表示获取锁得线程实体。如果不能获取到锁，则`acquire(1)`将当前线程添加到队列挂起，可以看出这里是直接跟其他线程竞争锁的。
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
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
这里有种情况就是当前线程重复得获取锁，这是在`protected final boolean tryAcquire(int acquires)`方法中实现得。先获取锁得状态，`if (c == 0)`说明锁没有被其他线程获取，此时判断队列前是否有其他线程，没有得话获取锁即可；如果`else if (current == getExclusiveOwnerThread()`，则说明获取锁得线程就是当前线程，则重复获取。
```java
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                //获取锁本质是cas将state属性置为1，记录当前获取锁的线程
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //重复获取，如果当前线程重复调用lock则state+1
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
无法获取到锁时则将当前线程加入队列，例如原本队列为:[HEAD] <=> [T1] <=> [T2]，现在新的线程调用lock获取锁失败时为:[HEAD] <=> [T1] <=> [T2] <=> [T3]
```java
    private Node addWaiter(Node mode) {
        // 使用node封装当前线程
        Node node = new Node(Thread.currentThread(), mode);
        // 将包含当前线程的节点node加入队列尾
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            //判断tail尾节点是否为pred值，是的话设置为node
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
            //队列中第一个节点为空节点,调用lock方法失败说明必有一个线程成功lock到锁，此时队列中应该有两个节点
            //队列中第一个节点为持有锁的节点，第二个节点为有资格争夺锁的节点
            //初始化时添加一个空的node节点，在持有锁的线程调用release方法释放时才能唤醒lock失败的当前线程
            //release中释放锁会判断(if (h != null && h.waitStatus != 0))
            if (t == null) { 
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
队列中的第二个节点是可以去竞争锁的，例如[HEAD] <=> [T1] <=> [T2] <=> [T3]，当线程位于HEAD节点之后，即队列中第二个节点时则可以竞争锁，如果竞争成功则移除第一个节点，将自身变化头节点，如[HEAD(T1)] <=> [T2] <=> [T3]， 否则将自身挂起。
```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                //获取前一个节点
                final Node p = node.predecessor();
                //如果前一个节点是头节点空节点，说明自身为第二个节点，则竞争锁
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
遍历整个链表，找到waitStatus(`volatile int waitStatus;`)为Node.SIGNAL的节点，然后将当前线程挂起。正常情况下每个线程节点的状态都是SIGNAL，如[HEAD] <=> [T1(SIGNAL)] <=> [T2(SIGNAL)] <=> [T3(SIGNAL)]，此时当前线程直接休眠；如果发现ws > 0即线程因为异常状态为CANCELLED时，如[HEAD] <=> [T1(SIGNAL)] <=> [T2(CANCELLED)] <=> [T3(SIGNAL)]，后面唤醒线程时从队列尾向队列前移动时就只能找到T3，但是T3不在HEAD后面导致线程没法被唤醒，所以需要移除CANCELLED的线程，变为：[HEAD] <=> [T1(SIGNAL)] <=> [T3(SIGNAL)]，在其他线程unlock释放锁，就能找到离HEAD最近的T1线程唤醒。
```java
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        //waitStatus在初始化和释放锁的时候都会置为0
        int ws = pred.waitStatus;
        //前一个节点为Node.SIGNAL时返回，之后waitStatus为Node.SIGNAL的节点会唤醒当前线程
        if (ws == Node.SIGNAL)
            return true;
        if (ws > 0) {
            //一直往前移动，直到pred.waitStatus<=0，如果pred.waitStatus > 0则会从队列中移除
            //pred.waitStatus > 0意味着线程发生异常不用被唤醒
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            //前一个节点设置为Node.SIGNAL，用于唤醒本线程
            //后面沿队列尾向头遍历时就能找到离头最近的node节点if (t.waitStatus <= 0)
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
+ 公平锁   
这里就是判断此线程之前是否还有其他线程在等待获取锁，如果有返回true。
```java
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }
    public final boolean hasQueuedPredecessors() {
        Node t = tail; // 尾节点
        Node h = head; // 头节点
        Node s;
        // 初始化时h == t表示队列为空
        // h != t说明队列中有其他线程
        //(s = h.next) == nul代码的点睛之笔,意思如果在acquireQueued()方法中获取到锁，执行到p.next = null，
        // 此时链表非空，头、尾节点指向不同，但是 h.next为null
        // 总之返回true说明要么有线程排在自己前面，要么有其他线程已经加锁成功
        // h != t && (s = h.next) == null -> 有线程正在获取锁并且已经获取
        // h != t &&  h.next.thread != Thread.currentThread() -> 有线程排在自己前面先一步lock获取锁
        return h != t && ((s = h.next) == null || s.thread != Thread.currentThread());
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
如果锁释放成功则唤醒队列中其他线程，找到waitStatus<0的线程，调用`LockSupport.unpark(s.thread)`唤醒。这里分三种情况：
1. 正常情况  
[HEAD] <=> [T1(SIGNAL)] <=> [T2(SIGNAL)] <=> [T3(SIGNAL)]，此时直接唤醒T1线程，T1发现自己位于HEAD之后就会获取锁
2. 线程异常  
[HEAD] <=> [T1(CANCELLED)] <=> [T2(SIGNAL)] <=> [T3(SIGNAL)]，此时从队列尾向前遍历，找到离HEAD最近的T2节点，T2被唤醒后在acquireQueued方法的for循环中发现自己不是队列中第二个节点，就会在shouldParkAfterFailedAcquire方法中从后往前移除掉CANCELLED线程，链表变为[HEAD]  <=> [T2(SIGNAL)] <=> [T3(SIGNAL)]，此时T2为HEAD后一个节点，就能获取到锁
3. 极端情况
s == null成立为极端情况了，一个线程调用unlock释放锁时队列为[HEAD(TAIL)]，此时另外一个线程调用lock获取锁执行到addWaiter中的pred.next = node语句，会将链表改为[HEAD] <- [T1(SIGNAL)]，即向后的next指向还未建立，代码继续往下执行，此时[HEAD] <=> [T1(SIGNAL)]，从队列尾向前遍历就能找到要唤醒的T1线程。
```java
    private void unparkSuccessor(Node node) {
        //将当前线程节点waitStatus置为0
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);
        //遍历链表，唤醒调用lock等待锁的线程
        Node s = node.next;
        //优先唤醒当前队列中释放的线程的next下一个线程
        //s == null可能会出现，一个线程调用unlock释放锁的同时执行到此处，另外一个线程lock获取锁时执行到addWaiter中的pred.next = node;
        //此时，链表存在，但是头节点的下个节点为null，只要遍历一次找出即可
        if (s == null || s.waitStatus > 0) {
            s = null;
            //从队列尾向前遍历找到离头节点最近的能够被唤醒状态不为cancelled的线程节点
            //acquireQueued方法中for死循环不断尝试获取锁，
            //被唤醒线程节点的前一个节点不是head后，会将node.prev = pred = pred.prev;状态为cancelled的线程清理掉，直到自己成为head的下个节点
            //1.尾部为刚插入的节点不会轻易因为异常变为cancelled 
            //2.next= null 出现的概率大
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        //唤醒
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```