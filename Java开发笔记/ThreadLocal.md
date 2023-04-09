首先看下ThreadLocal的set()方法存数据的过程，首先获取当前的线程中保持的ThreadLocalMap，每个线程的ThreadLocalMap都是不一样的，因此存储的值是不同的。
```java
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            map.set(this, value);
        } else {
            createMap(t, value);
        }
    }
```
如果在一个线程中首次使用ThreadLocal保持数据，则需要创建ThreadLocalMap，ThreadLocalMap中保存数据的实体是Entry，保存数据的过程就是先计算这个ThreadLocal对象的hashcode，根据hashcode计算在Entry数组中的位置，然后将创建的Entry保存在这个位置。
```java
    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
```
如果在第一次之后使用ThreadLocal的话，则根据ThreadLocal计算hashcode，再根据hashcode计算Entry数组的索引，根据索引找到这个线程对应的Entry，如果是当前线程使用的ThreadLocal`if (k == key)`，则将对象设置进来，即写到存储数据的Entry中。
```java
        private void set(ThreadLocal<?> key, Object value) {

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();

                if (k == key) {
                    e.value = value;
                    return;
                }

                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            tab[i] = new Entry(key, value);
            int sz = ++size;
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```
当通过get()方法获取数据时，首先找到当前的线程对象，获取线程对象内部的ThreadLocalMap，然后根据ThreadLocal对象计算Entry的索引，找到本线程存储数据的Entry，获取Entry中的数据。
```java
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
                return getEntryAfterMiss(key, i, e);
        }
```
+ ThreadLocal内存泄漏的问题  
可以看到Entry是指向ThreadLocal的弱引用，弱引用不会阻止gc的垃圾回收，如果这个ThreadLocal对象置为null，指向ThreadLocal对象的弱引用不会阻止gc的垃圾回收，此时ThreadLocal对象会被gc回收，通过get()方法获取value时需要计算ThreadLocal对象的hashcode，在ThreadLocal对象被回收的情况就无法计算hashcode，也就无法访问这个value引用的对象，于是value就成了有引用链但是无法被访问的内存，即造成内存泄漏了。
```java
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```
解决方法：
1. 将ThreadLocal变量定义成private static，这样就一直存在ThreadLocal的强引用，可以通过ThreadLocal对象访问到保存的数据，不会造成内存泄漏
2. 调用remove()方法清除内存
```java
     public void remove() {
         ThreadLocalMap m = getMap(Thread.currentThread());
         if (m != null) {
             m.remove(this);
         }
     }
        private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.get() == key) {
                    e.clear();
                    expungeStaleEntry(i);
                    return;
                }
            }
        }
```