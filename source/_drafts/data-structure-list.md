---
title: 数据结构：List
categories: 数据结构
date: 2017-12-07 22:09:49
tags:
  - 数据结构
  - List
---


List 是我们编程常用的数据结构。下面我们就来看看 ArrayList 和 LinkedList。

<!-- more -->

## ArrayList

ArrayList 其本质就是容量（length）可变的一个数组。

先来一版最简单的了解原理。

```Java
public class MiniArrayList {

    /**
     * 存储元素的数组，默认大小为 10
     */
    private Integer[] data = new Integer[10];

    /**
     * 当前 list 中元素的个数
     */
    private int size;

    public MiniArrayList() {
    }


    public boolean add(Integer i) {
        data[size++] = i;
        return true;
    }

    public Integer get(int index) {
        return data[index];
    }

    public int size() {
        return size;
    }
}
```

如果元素超过 10 个，数组怎么进行扩容呢？来看看 Java 给我们提供的方法。

```Java
// java.util.Array
/**
* @param original 原数组
* @param newLength 新的数组长度
*/
public static int[] copyOf(int[] original, int newLength) {
    int[] copy = new int[newLength];
    System.arraycopy(original, 0, copy, 0,
                     Math.min(original.length, newLength));
    return copy;
}
```
再看看 System.arraycopy 这个方法的的源码。

```Java
// java.lang.System
/**
* 这个是一个 native 方法
*
* @param src      原数组
* @param srcPos   原数组中需要被复制的元素的起始下标
* @param dest     目标数组
* @param destPos  被复制的元素，存储在目标数组中的起始起始下标
* @param length   需要复制的长度
*/
public static native void arraycopy(Object src,  int  srcPos, Object dest, int destPos, int length);
```

所以，我们修改一下 add 方法。

```Java
public boolean add(Integer i) {
    if (size == data.length) {
        // 每次扩大一倍容量
        data = Arrays.copyOf(data, data.length * 2);
    }
    data[size++] = i;
    return true;
}
```

我们再看看添加元素到指定位置。

假设，我们要把元素 6 加入到元素 2 的位置，也就是加入到下标为 1 的位置。

![添加元素到指定位置](http://p0e1o9bcz.bkt.clouddn.com/list/arraylist-add-index-data-1.png?v1&imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

第一步，把下标大于等于 1 的元素，往后移动一位。

![添加元素到指定位置](http://p0e1o9bcz.bkt.clouddn.com/list/arraylist-add-index-data-2.png?v1&imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

第二步，把元素 6 存到下标为 1 的位置。

![添加元素到指定位置](http://p0e1o9bcz.bkt.clouddn.com/list/arraylist-add-index-data-3.png?v1&imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)


```Java
public boolean add(int index, Integer i) {
    //需要检查下标的范围

    if (size == data.length) {
        // 默认每次扩大一倍容量
        data = Arrays.copyOf(data, data.length * 2);
    }
    //移动元素
    System.arraycopy(data, index, data, index + 1, size - index);

    data[index] = i;
    size++;
    return true;
}
```

最后，我们在看看 remove 方法。其实也是移动元素，只不过是往前移动。

```Java
public Integer remove(int index) {
    //需要检查下标的范围

    Integer value = data[index];
    //计算需要移动元素个数
    int numMoved = size - index - 1;
    //如果不是最后元素
    if (numMoved > 0) {
      //移动元素 从下标为 index+1 的元素开始，移动到位置 index 的位置，移动个数是 numMoved
      System.arraycopy(data, index + 1, data, index, numMoved);
    }
    data[--size] = null;
    return value;
}
```

上面的代码只是很简单地介绍了 ArrayList，还存在很多的 BUG。


## LinkedList

LinkedList 分单向链表、双向链表、循环链表。

### 单向链表

单向链表就是每个结点不仅仅保存数据，还要保存下一个结点的引用。

![单向链表](http://p0e1o9bcz.bkt.clouddn.com/list/singly.png?v1&imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

来看看怎么构建一个单向链表。

```Java
public class SinglyLinkedList {

    /**
     * 链表头
     */
    private Node head;

    /**
     * 数据结点
     */
    private static class Node {
        int data;
        Node next;

        public Node(int data) {
            this.data = data;
        }
    }

    /**
     * 链表中元素的个数
     */
    private int size;

    public int size() {
        return this.size;
    }
}

```

看看怎么向单向链表中插入元素。

第一种情况，插入元素到链表头。

![插入元素到链表头](http://p0e1o9bcz.bkt.clouddn.com/list/singly-insert-front.png?v1&imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

```Java
public void push(int data) {
  Node newNode = new Node(data);
  newNode.next = head;
  head = newNode;
  size++;
}

```

第二种情况，插入元素到指定元素后面。

![插入元素到指定元素后面](http://p0e1o9bcz.bkt.clouddn.com/list/singly-insert-after.png?v1&imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

```Java
public void insertAfter(Node prevNode, int data) {
  if ( prevNode == null ) {
    return;
  }
  Node newNode = new Node(data);
  newNode.next = prevNode.next;
  prevNode.next = newNode;
  size++;
}

```

第三种情况，追加到链表尾部。

![追加元素到链表尾部](http://p0e1o9bcz.bkt.clouddn.com/list/singly-insert-end.png?v1&imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

```Java
public void append(int data) {
    Node newNode = new Node(data);
    if (head == null) {
        head = newNode;
        size++;
        return;
    }
    Node last = head;
    while (last.next != null) {
        last = last.next;
    }
    last.next = newNode;
    size++;
 }
```

再来看看删除元素。

![删除元素](http://p0e1o9bcz.bkt.clouddn.com/list/singly-delete.png?v1&imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

```Java
public void remove(int key) {
    //判断是否是删除 head
    Node del = head;
    if (del != null && del.data == key) {
        head = del.next;
        size--;
        return;
    }
    Node prev = null;
    while (del != null && del.data != key ) {
        prev = del;
        del = del.next;
    }
    if (del == null) {
        return;
    }
    prev.next = del.next;
    size--;
}
```

从上面可以看出，单向链表每次查找元素都需要从 head 结点开始，结点越来越多，遍历时间也就越来越长。有什么办法优化呢？

### 双向链表


双向链表就是每个结点不仅仅保存数据和下一个结点的引用，还要保存上一个结点的引用。

![双向链表](http://p0e1o9bcz.bkt.clouddn.com/list/doubly.png?v2&imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

构造一个双向链表。

```Java
public class DoublyLinkedList {

    /**
     * 链表头
     */
    private Node first;

    /**
     * 链表尾
     */
    private Node last;

    /**
     * 数据结点
     */
    private static class Node {
        Node prev;
        int data;
        Node next;

        public Node(int data) {
            this.data = data;
        }
    }

    /**
     * 链表中元素的个数
     */
    private int size;


    public int size() {
      return this.size;
    }
}
```
往双向链表中插入结点。

第一种情况，插入元素到链表头。

```Java
public void addFirst(int data) {
  Node f = first;
  Node newNode = new Node(data);
  newNode.next = head;
  newNode.prev = null;
  first = newNode;
  if (f == null){
    last = newNode;
  } else {
    f.prev = newNode;
  }
  size++;
}
```

第二种情况，插入元素到指定位置。

```Java
public void add(int index,int data){
  //检查index的范围

  if(index == size){
    //追加到链表尾
    addLast(data);
  } else {
    //找到index的结点
    Node prev = node(index);
    linkBefore(data, node(index));
  }
}
void linkBefore(E e, Node<E> succ) {
  // assert succ != null;
  final Node<E> pred = succ.prev;
  final Node<E> newNode = new Node<>(pred, e, succ);
  succ.prev = newNode;
  if (pred == null)
    first = newNode;
  else
    pred.next = newNode;
  size++;
}
Node node(int index) {
  //双向链表的好处 可以判断index靠近哪一端，就从哪一段开始遍历
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

第三种情况，插入元素到链表尾。


```Java
public void addLast(int data) {
  Node l = last;
  Node newNode = new Node(data);
  last = newNode;
  if (l == null) {
    first = newNode;
  } else {
    l.next = newNode;
  }
  size++;
}
```

获取元素。
```Java
public int get(int index) {
  //检查index范围
  return node(index).data;
}
```

删除元素。

```Java
public int remove(int index) {
  //检查index范围
  return unlink(node(index));
}
E unlink(Node x) {
  // assert x != null;
  final int element = x.item;
  final Node next = x.next;
  final Node prev = x.prev;

  if (prev == null) {
      first = next;
  } else {
      prev.next = next;
      x.prev = null;
  }
  if (next == null) {
      last = prev;
  } else {
      next.prev = prev;
      x.next = null;
  }
  x.item = null;
  size--;
  return element;
}

```

更加详细地操作可以看，Java 中的 LinkedList。

### 循环链表

![循环链表](http://p0e1o9bcz.bkt.clouddn.com/list/circular.png?imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

所有的结点连成一个环，最后一个结点不指向 NULL，这样的话任何结点都可以成为起点，我们可以从任何一个结点遍历整个列表。循环链表可以是单循环链表或双循环链表。

## 总结

ArrayList 的优点就是支持随机访问元素，但是添加和删除元素时，有时需要移动元素，所以如果知道数据的大小，最好在初始化的时候指定初始容量，避免多次移动元素。而 LinkedList 恰好与 ArrayList 相反，不支持随机访问元素，每次都要从链表头或尾依次访问，但是在添加和删除元素时，不需要移动元素，只需要改变引用即可。所以，当需要频繁的添加和删除元素时，推荐使用 LinkedList。

LinkedList 理论上是无限大的。而 ArrayList 是有大小限制的，知道最大值是多少么？

ArrayList 只需要保存 data 数据，而 LinkedList 还需要额外空间保存引用数据。


## 参考资料
Linked List Data Structure：http://www.geeksforgeeks.org/data-structures/linked-list
