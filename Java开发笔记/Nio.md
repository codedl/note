---
title: Nio
tags:
notebook: Java开发笔记
---
+ ButeBuffer
1. 通过静态方法allocate分配缓冲区`ByteBuffer buffer = ByteBuffer.allocate(1024);`，分配一个数组，容量capacity为allocate的参数，limit=capacity，position=0，mark=-1;
2. 写入数据时调用`buffer.put(byte[]);`，写入数据后position+=写入缓冲区的字节数
    1. 判断要写入的数组是否存在越界问题
    2. 判断写入数据到缓冲区后是否会导致缓冲区越界
    3. 按字节逐个写入缓冲区
3. 读取数据时先调用`flip()`反转
   1. limit = position，设置成缓冲区已有数据大小
   2. position = 0，表示从第一个字节开始操作
   3. mark = -1 ，清除标记
4. 调用get方法读取数据`buffer.get(byte[]);`
   1. 判断指定的数组是否存在越界问题
   2. 判断当前缓冲区是否有足够的字节数据可以读取
   3. 按字节从缓冲中读取数据
+ FileChannel
1. 通过FileInputStream或FileOutputStream的getChannel()方法返回FileChannel，`fileInputStream.getChannel()`
2. 调用[read](#read)从通道读取数据到缓冲区`inChannel.read(buffer)`，本质时linux的系统调用`read(fd, buf, len)`
   1. 判断缓冲区是否为直接缓冲区，如果是直接缓冲区则直接将数据读到直接缓冲区；直接缓冲区位于操作系统可见的内存
   2. 如果是堆缓冲区，则先将数据读到临时直接缓冲区再读到堆缓冲区；堆缓冲区的虚拟内存地址只对Java虚拟机可见，由于gc的原因，堆缓冲区的地址一直变化，无法被操作系统访问
3. 调用write从缓冲区将数据写入通道`outChannel.write(buffer);`，本质时linux的系统调用`write(fd, buf, len)`
+ ServerSocketChannel
1. 调用ServerSocketChannel类的静态方法open()创建一个ServerSocketChannel`serverSocketChannel = ServerSocketChannel.open();`
   1. 对ServerSocketChannelImpl类进行实例化
   2. 调用socket创建套接字`socket(domain, type, 0)`
2. 设置channel为非阻塞模式`serverSocketChannel.configureBlocking(false);`
   1. 本质上是调用fcntl给fd设置相关标志位`fcntl(fd, F_SETFL, newflags);`
3. 将channel跟ip端口绑定`serverSocketChannel.bind(new InetSocketAddress(8000));`
   1. 调用bind将socket套接字跟ip端口绑定`int bind(int sockfd, struct sockaddr* addr, socklen_t addrlen)`
   2. 调用listen使进程可以接受其他进程的请求`int listen(int sockfd, int backlog)`
4. 将通道及其事件注册到selector上，当通道上注册的事件就绪时，selector方法就从阻塞中返回，返回的是监听到的事件的数量 `serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);`
5. 监听来自客户端的连接`SocketChannel socketChannel = serverSocketChannel.accept();`
   1. 本质上是调用accept接受来自客户端的连接`int accept(int sockfd,struct sockaddr *addr, socklen_t *addrlen);`
+ Selector
1. 先调用静态方法open()创建Selector`Selector selector = Selector.open();`
2. 监听通道的各种事件
   1. 调用select()对注册的通道进行监听，会一直阻塞，当通道中出现事件就会停止阻塞`selector.select()`
   2. 调用selectNow()对注册的通道进行监听，不会阻塞`selector.selectNow()`
   + 本质是调用epoll机制
   1. 通过epoll_create创建要监听的文件描述符，之后可以通过这个文件描述符对相应的文件进行监听`int epoll_create(int size);`
   2. 通过epoll_ctl注册要监听的文件及事件 `int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);`
      1. epfd 是epoll_create返回得文件描述符
      2. op 表示对应得操作
      3. fd 表示要监听得文件
      4. event 表示要监听得文件上得事件
   3. 通过epoll_wait对epfd进行监听`int epoll_wait(int epfd, struct epoll_event *events,int maxevents, int timeout);`
      1. epfd 为epoll_create返回得文件描述符
      2. events 是个出参，返回事件数组
      3. maxevents events数组大小
      4. timeout 超时时间
3. 可以调用wakeup()让select()方法立即返回，停止阻塞
4. 获取通道中返回的事件，不同事件进行不同的处理`selector.selectedKeys()`
# allocate
```
//Buffer类中的四个属性：
// Invariants: mark <= position <= limit <= capacity
private int mark = -1; //索引，标记操作数据时数据所在的位置
private int position = 0; //下次操作数据的索引，下次操作数据时从position开始
private int limit; //能够操作的字节数，写的时候为缓冲区大小，读的时候为缓冲区已有的字节数
private int capacity; //标识当前buffer缓冲区大小，即能存储的字节数

public static ByteBuffer allocate(int capacity) {
    if (capacity < 0)
        throw new IllegalArgumentException();
    return new HeapByteBuffer(capacity, capacity);
}
HeapByteBuffer(int cap, int lim) {            // package-private
    super(-1, 0, lim, cap, new byte[cap], 0);
}
ByteBuffer(int mark, int pos, int lim, int cap, byte[] hb, int offset)
{
    super(mark, pos, lim, cap);
    this.hb = hb;
    this.offset = offset;
}
Buffer(int mark, int pos, int lim, int cap) {       // package-private
    if (cap < 0)
        throw new IllegalArgumentException("Negative capacity: " + cap);
    this.capacity = cap;
    limit(lim);
    position(pos);
    if (mark >= 0) {
        if (mark > pos)
            throw new IllegalArgumentException("mark > position: ("
                                               + mark + " > " + pos + ")");
        this.mark = mark;
    }
}
```
# put
```
public ByteBuffer put(byte[] src, int offset, int length) {
    //检查off、len是否小于0，(off + len)是否超出int范围，size是否小于(off + len)即是否越界
    checkBounds(offset, length, src.length);
    //判断写入缓冲区数据的长度是否超出缓冲区剩余的字节数，即是否会越界
    if (length > remaining())
        throw new BufferOverflowException();
    int end = offset + length;
    //将数据按字节写入
    for (int i = offset; i < end; i++)
        this.put(src[i]);
    return this;
}
public ByteBuffer put(byte x) {
    hb[ix(nextPutIndex())] = x;
    return this;
}
final int nextPutIndex() {                          // package-private
    int p = position;
    if (p >= limit)
        throw new BufferOverflowException();
    position = p + 1;
    return p;
}
```
# flip
```
public final Buffer flip() {
    limit = position;
    position = 0;
    mark = -1;
    return this;
}
```
```
public ByteBuffer get(byte[] dst, int offset, int length) {
    checkBounds(offset, length, dst.length);
    if (length > remaining())
        throw new BufferUnderflowException();
    int end = offset + length;
    for (int i = offset; i < end; i++)
        dst[i] = get();
    return this;
}
public byte get() {
    return hb[ix(nextGetIndex())];
}
```
# read
```
public int read(ByteBuffer var1) throws IOException {
......
   var3 = IOUtil.read(this.fd, var1, -1L, this.nd);
......
}
static int read(FileDescriptor var0, ByteBuffer var1, long var2, NativeDispatcher var4) throws IOException {
     if (var1.isReadOnly()) {
         throw new IllegalArgumentException("Read-only buffer");
     } else if (var1 instanceof DirectBuffer) {
         return readIntoNativeBuffer(var0, var1, var2, var4);
     } else {
         ByteBuffer var5 = Util.getTemporaryDirectBuffer(var1.remaining());

         int var7;
         try {
             int var6 = readIntoNativeBuffer(var0, var5, var2, var4);
             var5.flip();
             if (var6 > 0) {
                 var1.put(var5);
             }

             var7 = var6;
         } finally {
             Util.offerFirstTemporaryDirectBuffer(var5);
         }

         return var7;
     }
 }
class FileDispatcherImpl extends FileDispatcher {
    static native int read0(FileDescriptor var0, long var1, int var3) throws IOException;
}
```
# open
```
ServerSocketChannelImpl(SelectorProvider var1) throws IOException {
  super(var1);
  this.fd = Net.serverSocket(true);
  this.fdVal = IOUtil.fdVal(this.fd);
  this.state = 0;
}
static FileDescriptor serverSocket(boolean var0) {
  return IOUtil.newFD(socket0(isIPv6Available(), var0, true, fastLoopback));
}
//socket0 => socket(domain, type, 0);
```