首先看ArrayList集合，ArrayList集合实现了List和List接口继承的Collection接口。
+ 扩容：在add()方法向集合中添加元素时触发，首先计算数组需要的容量，在默认值10和加入到数组的元素个数跟当前数组容量的和之间取最大值，然后判断是否需要进行扩容，即这个值是否大于当前数组容量，如果需要扩容时扩大0.5倍，同时通过Arrays.copyOf()方法将原来数组内容拷贝过来。
```java
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }
    private static int calculateCapacity(Object[] elementData, int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            return Math.max(DEFAULT_CAPACITY, minCapacity);
        }
        return minCapacity;
    }
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    private void grow(int minCapacity) {
        // overflow-conscious code
        int oldCapacity = elementData.length;
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```
+ 遍历问题：构造迭代器时会记录当前的ArrayList的修改次数，每次迭代next()时都会进行检查，如果发现其他线程修改了ArrayList，即`if (modCount != expectedModCount)`，就会抛出异常。
```java
// ArrayList$Itr
        int expectedModCount = modCount;
        public E next() {
            checkForComodification();
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }
        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
```
+ remove问题：remove时会将当前位置的元素删除，然后将后面的元素向前复制，即每个元素都前进了一个位置，此时遍历就会丢失，因此应该逆序遍历。
```java
    public boolean remove(Object o) {
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                    fastRemove(index);
                    return true;
                }
        } else {
            for (int index = 0; index < size; index++)
                if (o.equals(elementData[index])) {
                    fastRemove(index);
                    return true;
                }
        }
        return false;
    }
    private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }
```
+ 线程安全的list集合CopyOnWriteArrayList：保持数据的数组array上存在volatile关键字，多线程添加数据时需要先获取锁，会将当前数组拷贝到新数组中，再将新的数据添加到新的数组中，然后setArray通知其他线程，这样其他线程获取到的array就是修改后的array。
```java
// CopyOnWriteArrayList
    private transient volatile Object[] array;
    public boolean add(E e) {
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
```