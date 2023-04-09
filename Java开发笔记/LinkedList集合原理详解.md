LinkedList底层是用链表保存数据的，先看下关键的属性和数据结构。
```java
    transient Node<E> first;//链表头节点
    transient Node<E> last;//链表尾节点

    private static class Node<E> {
        E item; //数据对象
        Node<E> next;//前一个节点
        Node<E> prev;//后一个节点

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```
看下add()方法添加数据到集合的源码，其实就是用Node对数据进行封装，然后加入到链表尾部。
```java
    public boolean add(E e) {
        linkLast(e);
        return true;
    }
    void linkLast(E e) {
        final Node<E> l = last;//找到链表中最后一个节点
        final Node<E> newNode = new Node<>(l, e, null);//新加入的节点指向最后一个节点
        last = newNode;//让链表中最后一个节点指向自己
        if (l == null) //如果最后一个节点为空，则first和last都指向这个节点
            first = newNode;
        else
            l.next = newNode; //链表已经存在就让最后一个节点的next指向新加入的节点
        size++;
        modCount++;
    }
```
再看怎么通过get()获取数据，先检查数据容量，然后再获取数据。
```java
    public E get(int index) {
        checkElementIndex(index);
        return node(index).item;
    }
```
数据的索引index必须为非负的，且不能超过集合容量。
```java
    private void checkElementIndex(int index) {
        if (!isElementIndex(index))
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
    private boolean isElementIndex(int index) {
        return index >= 0 && index < size;
    }
```
发现宝藏了，Java为了加快数据的查找就用了二分法，首先判断index是否在链表的前半段，如果在就从前往后遍历，否则就从后往前遍历。
```java
    Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```