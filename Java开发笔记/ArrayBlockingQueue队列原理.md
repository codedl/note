面试的时候又被人问到了，怎么实现一个延时操作，那其实就是通过队列实现的，调用带参数的poll()方法就行了，老规矩先看一些重要的属性。
```java
    int count;//队列长度
    final Object[] items;//存储数据的数组
    int putIndex;//元素的位置
    private final Condition notEmpty;//实现不同线程通信的信号量，如果队列中有数据就会signal()唤醒等待取数据的线程
```    
现在开始看对集合进行操作的方法，首先是保存数据的offer()方法，先加锁，然后将数据加入队列，最后指向finally代码块中的unlock()释放锁。
```java
    public boolean offer(E e) {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            if (count == items.length)
                return false;
            else {
                enqueue(e);
                return true;
            }
        } finally {
            lock.unlock();
        }
    }
    private void enqueue(E x) {
        final Object[] items = this.items;//找到存储数据的数组
        items[putIndex] = x;//保存数据
        if (++putIndex == items.length)//如果数组满了，则从头开始，防止数据溢出
            putIndex = 0;
        count++;//计数
        notEmpty.signal();//唤醒await()的线程
    }
```
看下带超时时间作为参数的poll()方法，调用条件变量awaitNanos()延时阻塞，然后再获取元素。
```java
    public E poll(long timeout, TimeUnit unit) throws InterruptedException {
        long nanos = unit.toNanos(timeout);//转化为ns
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0) {
                if (nanos <= 0)
                    return null;//如果是负数直接返回
                nanos = notEmpty.awaitNanos(nanos);//阻塞nanos个ns
            }
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
    private E dequeue() {
        final Object[] items = this.items;//存储数据的数组
        @SuppressWarnings("unchecked")
        E x = (E) items[takeIndex];//取出数据
        items[takeIndex] = null;//置为null
        if (++takeIndex == items.length)//如果数组满了则从头开始
            takeIndex = 0;
        count--;
        if (itrs != null)
            itrs.elementDequeued();
        notFull.signal();//释放集合没有满的信号
        return x;
    }
```
take()相比下就好看多了，直接阻塞，直到有数据进入集合位置。
```java
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0)
                notEmpty.await();
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
```