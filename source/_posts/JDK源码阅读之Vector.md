---
title: JDK源码阅读之Vector
date: 2016-07-04 09:31:16
tags: [Java,multi-thread]
categories:
- Java
- JDK
---

> JDK源码阅读系列文章：
> https://zhum.in/blog/categories/Java/JDK/

- - -

# 概述
Vector实现了一个增长型的Object数组，可以包含任何类型的元素，包括null。

像数组一样，它的元素可以使用下标索引值进行访问。为了容纳添加或删除后的元素，Vector的容量可以增长或收缩。每个Vector实例通过维护容量大小和容量增长因子来优化存储管理。Vector容量的大小要大于等于Vector中元素的实际个数，如果添加元素时需要扩容，则会按照增长因子的大小进行扩容。当需要添加大量的元素到Vector中，需要给它进行合适的扩容，这样可以减少增量式的内存再分配次数。

像ArrayList一样，通过Vector的iterator方法和listIterator方法返回iterators时被设计成是fail-fast，当调用这两个方法的时候，如果Vector实例被从结构上进行了修改(指添加或删除一个或多个元素的操作，除了ListIterator的remove方法或add方法，或者显式调整底层数组的大小，仅仅设置元素的值不是结构上的修改)，将会抛出ConcurrentModificationException，这样的设计避免了某个不确定的时间因为修改而出现的不可预知的问题。

<!-- more -->

需要注意的是这个类的elements方法并不是fail-fast的。Vector和ArrayList大体上的相同的，不同的是ArrayList是非同步的，而Vector是同步的，其实就是在实现的方法上使用synchronized进行同步。Vector具有同步的特性，是线程安全的，所以它的性能相对于ArrayList来说是低效的，因为同步操作需要消耗时间，针对于不需要使用线程安全的情况来说，推荐使用ArrayList代替Vector。Vector实现了RandomAccess接口可以进行快速的随机访问。Vector实现了Cloneable接口可进行clone操作。Vector实现了java.io.Serializable接口，可进行序列化操作。

# 数据结构
Vector类的声明代码如下，
```java
public class Vector<E>
    extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```
由上面的代码段可以得出如下所示的类图：

{% asset_img vector.png vector %}

Vector中声明了如下一些属性：
```java
protected Object[] elementData;
protected int elementCount;
protected int capacityIncrement;
protected transient int modCount = 0;
```

- elementData数组用于存储元素；
- elementCount为Vector中实际元素个数；
- capacityIncrement为Vector扩容的增长因子；
- modCount从AbstractList继承而来，用于记录从结构上修改此Vector的次数；

# 构造方法
提供了四个构造函数可供使用。
```java
//根据指定初始容量大小、容量增长因子构造一个空的Object数组
public Vector(int initialCapacity, int capacityIncrement) {
    super();
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    this.elementData = new Object[initialCapacity];
    this.capacityIncrement = capacityIncrement;
}

//根据指定初始容量大小、容量增长因子为零构造一个空的Object数组
public Vector(int initialCapacity) {
    this(initialCapacity, 0);
}

//初始容量大小为10、容量增长因子为0构造一个空的Object数组
public Vector() {
    this(10);
}

//将Collection中的元素按照迭代次序存入elementData中
public Vector(Collection<? extends E> c) {
    elementData = c.toArray();
    elementCount = elementData.length;
    // c.toArray might (incorrectly) not return Object[] (see 6260652)
    if (elementData.getClass() != Object[].class)
        elementData = Arrays.copyOf(elementData, elementCount, Object[].class);
}
```

# 扩容
当添加元素的时候，检查数组容量是否需要扩容。
```java
//使用synchronized进行同步
public synchronized void ensureCapacity(int minCapacity) {
    if (minCapacity > 0) {
        modCount++;
        ensureCapacityHelper(minCapacity);
    }
}

//指定的扩容大小大于elementData数组的长度才进行扩容
private void ensureCapacityHelper(int minCapacity) {
    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

//可分配的最大数组大小，因为一些VM需要在数组中预留字节头，所以需要减8
//尝试去分配的数组大小超过了VM的限制会报OutOfMemoryError
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

//从这里可以得出尝试扩容的大小可能是原容量加上增长因子的大小或原容量的二倍
//扩容后最大容量大小是Integer.MAX_VALUE
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                     capacityIncrement : oldCapacity);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}
```

# setSize方法
```java
public synchronized void setSize(int newSize) {
    modCount++;
    if (newSize > elementCount) {
        ensureCapacityHelper(newSize);
    } else {
        for (int i = newSize ; i < elementCount ; i++) {
            elementData[i] = null;
        }
    }
    elementCount = newSize;
}
```

设置容量大小为newSize。如果newSize大于当前的容量，则会进行扩容；否则，从newSize开始到末尾的所有元素会被丢弃，被设置为null。

# capacity方法
```java
public synchronized int capacity() {
    return elementData.length;
}
```
返回当前Vector的容量大小，是线程安全的。

# size方法
```java
public synchronized int size() {
    return elementCount;
}
```
返回当前Vector中元素的实际个数。

# isEmpty方法
```java
public synchronized boolean isEmpty() {
    return elementCount == 0;
}
```
判断当前Vector中元素的书籍个数是否等于零。

# elements方法
```java
public Enumeration<E> elements() {
    return new Enumeration<E>() {
        int count = 0;

        public boolean hasMoreElements() {
            return count < elementCount;
        }

        public E nextElement() {
            synchronized (Vector.this) {
                if (count < elementCount) {
                    return elementData(count++);
                }
            }
            throw new NoSuchElementException("Vector Enumeration");
        }
    };
}
```
返回一个Enumeration实例，用于遍历Vector。

# contains方法
```java
public boolean contains(Object o) {
    return indexOf(o, 0) >= 0;
}

public synchronized int indexOf(Object o, int index) {
    if (o == null) {
        for (int i = index ; i < elementCount ; i++)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = index ; i < elementCount ; i++)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```
从前向后遍历elementData数组，匹配第一个和o相等的元素，如果存在返回对应元素的下标序号，否则返回-1。这里需要注意的是o.equals(elementData[i])，这里是Objetc的equals方法，比较的是引用。

# indexOf方法
```java
public int indexOf(Object o) {
    return indexOf(o, 0);
}
```
从前向后查找匹配的第一个元素，若存在返回对应的索引下标，否则返回-1。

# lastIndexOf方法
```java
public synchronized int lastIndexOf(Object o) {
    return lastIndexOf(o, elementCount-1);
}

public synchronized int lastIndexOf(Object o, int index) {
    if (index >= elementCount)
        throw new IndexOutOfBoundsException(index + " >= "+ elementCount);

    if (o == null) {
        for (int i = index; i >= 0; i--)
            if (elementData[i]==null)
                return i;
    } else {
        for (int i = index; i >= 0; i--)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```
从后向前查找匹配的第一个元素，若存在返回对应的索引下标，否则返回-1。

# elementAt方法
```java
public synchronized E elementAt(int index) {
    if (index >= elementCount) {
        throw new ArrayIndexOutOfBoundsException(index + " >= " + elementCount);
    }

    return elementData(index);
}
```
返回指定索引下标位置的元素。

# firstElement方法
```java
public synchronized E firstElement() {
    if (elementCount == 0) {
        throw new NoSuchElementException();
    }
    return elementData(0);
}
```
返回第一个元素。

# lastElement方法
```java
public synchronized E lastElement() {
    if (elementCount == 0) {
        throw new NoSuchElementException();
    }
    return elementData(elementCount - 1);
}
```
返回最后一个元素。

# setElementAt方法
```java
public synchronized void setElementAt(E obj, int index) {
    if (index >= elementCount) {
        throw new ArrayIndexOutOfBoundsException(index + " >= " +
                                                 elementCount);
    }
    elementData[index] = obj;
}
```

# removeElementAt方法
```java
public synchronized void removeElementAt(int index) {
    modCount++;
    if (index >= elementCount) {
        throw new ArrayIndexOutOfBoundsException(index + " >= " +
                                                 elementCount);
    }
    else if (index < 0) {
        throw new ArrayIndexOutOfBoundsException(index);
    }
    int j = elementCount - index - 1;
    if (j > 0) {
        System.arraycopy(elementData, index + 1, elementData, index, j);
    }
    elementCount--;
    elementData[elementCount] = null;//GC回收
}
```

# insertElementAt方法
```java
public synchronized void insertElementAt(E obj, int index) {
    modCount++;
    if (index > elementCount) {
        throw new ArrayIndexOutOfBoundsException(index
                                                 + " > " + elementCount);
    }
    //扩容判断及操作
    ensureCapacityHelper(elementCount + 1);
    //数据移动，其实是数组拷贝
    System.arraycopy(elementData, index, elementData, index + 1, elementCount - index);
    elementData[index] = obj;
    elementCount++;
}
```

# addElement方法
```java
public synchronized void addElement(E obj) {
    modCount++;
    //扩容判断及操作
    ensureCapacityHelper(elementCount + 1);
    elementData[elementCount++] = obj;
}
```

# removeElement方法和removeAllElements方法
```java
public synchronized boolean removeElement(Object obj) {
    modCount++;
    int i = indexOf(obj);
    if (i >= 0) {
        removeElementAt(i);
        return true;
    }
    return false;
}

public synchronized void removeAllElements() {
    modCount++;
    // Let gc do its work
    for (int i = 0; i < elementCount; i++)
        elementData[i] = null;

    elementCount = 0;
}
```

# toArray方法
```java
public synchronized Object[] toArray() {
    return Arrays.copyOf(elementData, elementCount);
}
```
该方法返回一个新的数组，数组中的元素按照Vector中的第一个到最后一个顺序排列，修改返回的数组不会影响原Vector中的数据。

# get方法
```java
E elementData(int index) {
    return (E) elementData[index];
}

public synchronized E get(int index) {
    if (index >= elementCount)
        throw new ArrayIndexOutOfBoundsException(index);

    return elementData(index);
}
```

# set方法
```java
public synchronized E set(int index, E element) {
    if (index >= elementCount)
        throw new ArrayIndexOutOfBoundsException(index);

    E oldValue = elementData(index);
    elementData[index] = element;
    return oldValue;
}
```
set方法就是给数组的指定下标成员重新赋值。

# add方法
```java
//首先进行扩容判断
//给数组的elementCount++位置赋值为e
public synchronized boolean add(E e) {
    modCount++;
    ensureCapacityHelper(elementCount + 1);
    elementData[elementCount++] = e;
    return true;
}

//给index下标成员重新赋值
public void add(int index, E element) {
    insertElementAt(element, index);
}
```

# remove方法
```java
//删除指定位置的元素
//原理：将index+1位置开始到末尾的元素向前移动一位，最后一位赋值为null，GC进行回收
public synchronized E remove(int index) {
    modCount++;
    if (index >= elementCount)
        throw new ArrayIndexOutOfBoundsException(index);
    E oldValue = elementData(index);

    int numMoved = elementCount - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                         numMoved);
    elementData[--elementCount] = null; // Let gc do its work

    return oldValue;
}

//删除指定内容的元素，删除匹配的第一个元素
//原理：将匹配的第一个元素的位置下标加1开始到末尾的元素向前移动一位，最后一位赋值为null，GC进行回收
public boolean remove(Object o) {
    return removeElement(o);
}
```

# clear方法
```java
public void clear() {
    removeAllElements();
}
```
将数组所有元素引用设置了null，Vector的元素个数置为0；

# iterator方法和listIterator方法
这两个方法同ArrayList中两个方法，不同点是进行了同步操作。在执行iterator或listIterator方法时，如果有线程从结构上做了修改(指任何添加或删除一个或多个元素的操作，或者显式调整底层数组的大小，仅仅设置元素的值不是结构上的修改)，这两个方法会fail-fast，将会抛出ConcurrentModificationException。
迭代器的快速失败行为无法得到保证，因为一般来说，不可能对是否出现不同步并发修改做出任何硬性保证。快速失败操作会尽最大努力抛出ConcurrentModificationException。
因此，为提高此类操作的正确性而编写一个依赖于此异常的程序是错误的做法，正确做法是：ConcurrentModificationException应该仅用于检测bug。抛异常就是为了避免数据问题而存在的，其实就是检查modCount是否是期望值，如果不是，则说明Vector数组被从结构上修改了。这里是经常会遇到的问题，使用Java的For Each方式遍历元素时删除匹配的元素，会抛出ConcurrentModificationException。Java中的For Each实际上使用的是iterator进行处理的，而iterator是不允许集合在iterator使用期间删除的。

# 小结
1. Vector底层也是基于动态数组的数据结构。
2. Vector在每次增加元素时，都要进行扩容判断，扩容时都要确保足够的容量。当容量不足以容纳当前的元素个数时，就设置新容量为旧的容量的2倍或旧容量加增长因子大小，如果新容量还不够，则新容量为最大容量。从中可以看出，当容量不够时，都要将原来的元素拷贝到一个新的数组中，耗时而且还需要重新分配内存，因此建议在事先能确定元素数量的情况下，明确指明容量大小。
3. Vector基于数组实现，可以通过下标索引直接查找到指定位置的元素，因此查找效率高，但每次插入或删除元素，就要大量地移动元素，效率低。
4. Vector是线程安全的，方法会进行同步，相对于ArrayList来说效率相对低，建议在非同步的情况下使用ArrayList。
