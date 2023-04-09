+ 多线程锁  
  1. Synchronized：一次只能被一个线程占有
  2. ReadWriteLock：被多个线程持有，写锁只能被一个线程占有
  3. ReentrantLock：一个线程的多个流程能获取同一把锁，就是可重入锁，即在一个线程中可以被重复的获取
  4. 自旋锁：AtomicXXX开头的锁，当没有获取时就会循环检查

+ 监测api调用  
jconsole可以对Java进程进行监控，可以实时查看Java进程的堆内存，线程数，cpu占用率等。

+ CPU占用过高  
top指令找到占用cpu过高的进程，top -Hp 30896找到占用cpu的线程，`printf '%x\n' 30929`将线程号转为十六进制，然后jstack 30896可以看到当前java进程内占用cpu过高的线程。

+ 线程池  
参考[Java线程池源码解析](https://blog.csdn.net/weixin_42145727/article/details/127624535)

+ 线程同步  
可以使用join()方法控制线程的执行顺序；使用CountDownLatch倒计时门栓用来让一个线程等待其他线程的执行，当其他线程执行完时再执行当前线程；线程a依赖线程b时使用wait()和notify()实现；使用CyclicBarrier循环栅栏保证多线程的执行顺序，当多个线程都完成某个操作时再继续往下执行

+ Runnable和Callable的区别  
Callable有返回值，Runnable没有返回值；Callable可以向上抛出异常，Runnable不支持异常向上抛出；Callable只能通过submit开启线程，Runnable既可以通过submit，也可以通过execute开启线程。

+ 双亲委派机制  
如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到最顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去完成加载。好处是防止类被重复加载，同时可以保护系统中的核心代码的安全性，系统的关键类只能有指定的类加载器加载。

+ HashMap扩容  
参考[HashMap原理](https://blog.csdn.net/weixin_42145727/article/details/129170450)

+ synchronized底层原理  
synchronized锁是通过monitorenter和monitorexit两个字节码指令实现的，在执行monitorenter指令的时候，首先要去尝试获取对象的锁，如果这个对象没被锁定，或者当前线程已经持有了那个对象的锁，就把锁的计数器的值增加一，因此synchronized是可重入锁。执行monitorexit指令时会将锁计数器减一，一旦计数器的值为零，锁随即就被释放了。当出现竞争时通过mutex实现互斥，需要将程序从用户态切换到内核态，cpu消耗很大。

+  获取Class的方法  
   1. 类型.Class
   2. Class.forName`public static Class<?> forName(String className) throws ClassNotFoundException`根据类路径进行加载，如果没有加载成功就会抛出ClassNotFoundException异常
   3. 通过jni本地方法调用`public final native Class<?> getClass();`
   4. 通过类加载器进行加载返回Class`public Class<?> loadClass(String name) throws ClassNotFoundException`

+  sleep()、wait()方法区别  
   1. sleep()是Thread的静态方法，wait()是object类的方法
   2. sleep()方法没有释放锁，而wait()会释放锁
   3. wait()需要notify()/notifyAll()手动唤醒，sleep自动唤醒
   4. sleep()会抛出异常，而wait()不会抛出异常的

+  ReentrantLock实现原理 
参考[ReentrantLock实现原理](https://blog.csdn.net/weixin_42145727/article/details/129229561)

+  AtomicXXX实现原理 
通过Unsafe类的compareAndSwapInt()方法将值写进去`public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);`，首先将将期望值与实际值比较，如果不一样返回false；如果相同则使用`lock cmpxchg`汇编指令将更新值写入寄存器中，由于指令前存在lock指令，因此是线程安全的。

+ Unsafe类  
只能以反射方式获取Unsafe静态单实例，可以通过这个实例对内存管理进行操作。
+ 动态代理
  1. jdk动态代理：代理类为proxy的子类，因此只能对接口进行代理，以反射调用方法
  2. cglib动态代理：既可以对接口又可以对类进行代理，按照索引调用目标方法
+ Java锁的升级
+ jvm频繁垃圾回收
+ HashMap为什么使用尾插法