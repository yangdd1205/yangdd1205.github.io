---
title: 数据结构：List
categories: 数据结构
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
        // 默认每次扩大一倍容量
        data = Arrays.copyOf(data, data.length * 2);
    }
    data[size++] = i;
    return true;
}
```

我们再看看添加元素到指定位置。

假设，我们要把元素 6 加入到元素 2 的位置，也就是加入到下标为 1 的位置。

![添加元素到指定位置](http://p0e1o9bcz.bkt.clouddn.com/list/arraylist-add-index-data-1.png?imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

只需要把元素 2 及其后面的元素，往后挪动一位。

![添加元素到指定位置](http://p0e1o9bcz.bkt.clouddn.com/list/arraylist-add-index-data-2.png?imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

最后，把元素 6 存到下标为 1 的位置。

![添加元素到指定位置](http://p0e1o9bcz.bkt.clouddn.com/list/arraylist-add-index-data-3.png?imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)


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

最后，我们在看看 remove 方法。

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

上面的代码只是很简单地介绍了 ArrayList，还存在很多的 BUG。完整版请点击 MiniArrayList 请看。


## LinkedList

LinkedList 分单向链表、双向链表、循环链表。

### 单向链表

单向链表就是每个结点不仅仅保存数据，还要保存下一个结点的引用。

![单向链表](http://p0e1o9bcz.bkt.clouddn.com/list/singly.png?imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

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
}

```

看看怎么向单向链表中插入元素。

第一种情况，插入元素到链表头。

![插入元素到链表头](http://p0e1o9bcz.bkt.clouddn.com/list/singly-insert-front.png?imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

```Java
public void push(int data) {
  Node newNode = new Node(data);
  newNode.next = head;
  head = newNode;
}

```

第二种情况，插入元素到指定元素后面。

![插入元素到指定元素后面](http://p0e1o9bcz.bkt.clouddn.com/list/singly-insert-after.png?imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

```Java
public void insertAfter(Node prevNode, int data) {
  if ( prevNode == null ) {
    return;
  }
  Node newNode = new Node(data);
  newNode.next = prevNode.next;
  prevNode.next = newNode;
}

```

第三种情况，追加到链表尾部。

![追加元素到链表尾部](http://p0e1o9bcz.bkt.clouddn.com/list/singly-insert-end.png?imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

```Java
public void append(int data) {
     Node newNode = new Node(data);
     if (head == null) {
         head = newNode;
         return;
     }
     Node last = head;
     while (last.next != null) {
         last = last.next;
     }
     last.next = newNode;

 }

```




从上面可以看出，单向链表每次查找元素都需要从 head 结点开始，结点越来越多，遍历时间也就越来越长，特别是寻找靠近链表尾的结点。有什么办法优化呢？

### 双向链表


双向链表就是每个结点不仅仅保存数据和下一个结点的引用，还要保存上一个结点的引用。

![双向链表](http://p0e1o9bcz.bkt.clouddn.com/list/doubly.png?imageView2/0/q/100|watermark/2/text/eWFuZ2Rvbmdkb25nLm9yZw==/font/5a6L5L2T/fontsize/240/fill/IzAwMDAwMA==/dissolve/100/gravity/SouthEast/dx/10/dy/10|imageslim)

```Java

```

### 循环链表


## 对比


链表的插入和删除，只需要修改前面两个元素。而arraylist 则需要移动已存在的元素。
链表理论上是无限大的，而 arraylist 的最大值是 Integer的最大值（知道原因么？）。
