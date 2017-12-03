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

最后，我们在看看 remove 方法。

```Java
public Integer remove(int index) {
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

LinkedList 分单链表和双链表。

### 单链表

单链表就是每个结点不仅仅保存数据，还要下一个结点的引用。

![单链表](http://p0e1o9bcz.bkt.clouddn.com/list/singly.png)




### 双链表

双链表就是每个结点不仅仅保存数据和下一个结点的引用，还要保存上一个结点的引用。

![双链表](http://p0e1o9bcz.bkt.clouddn.com/list/double.png)



## 对比
