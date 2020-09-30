# LinkedList添加删除元素源码分析

## LinkedList介绍

LinkedList是Java双向链表的实现
- 实现了List和Deque的接口，支持List和deque的特性
- 继承了AbstractSequentialList抽象类：遍历LinkedList推荐使用迭代器
- 实现了Cloneable接口，支持克隆
- 实现了Serializable，支持序列化

## 源码分析

### 节点结构

Node类是LinkedList的内部类，它是真正用来存储数据的结构。除数据item外有next和prev两个引用（指针）指向前后节点。

```java
private static class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

### 成员变量

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
{
    // 节点数量
    transient int size = 0;

    /**
     * Pointer to first node.
     * Invariant: (first == null && last == null) ||
     *            (first.prev == null && first.item != null)
     */
    transient Node<E> first;//链表头部的引用

    /**
     * Pointer to last node.
     * Invariant: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    transient Node<E> last;//链表尾部的引用
    
    ....
}
```

LinkedList中的成员变量包含三个，size是节点数量，first是链表头节点的引用，last是链表尾节点的引用。

### 构造函数

LinkedList包含两个构造函数，一个是无参构造函数，创建一个空的链表。另一个是LinkedList(Collection<? extends E> c) ，作用是将一个集合作为参数传入，构造成链表，使用到了addAll方法，该方法我们后面进行解析。

```java
/**
 * Constructs an empty list.
 */
public LinkedList() {
}

/**
 * Constructs a list containing the elements of the specified
 * collection, in the order they are returned by the collection's
 * iterator.
 *
 * @param  c the collection whose elements are to be placed into this list
 * @throws NullPointerException if the specified collection is null
 */
public LinkedList(Collection<? extends E> c) {
    this();
    addAll(c);
}
```

### 添加节点

#### 将元素添加到节点尾部

下面这个两个方法的实现均是将元素添加到节点的尾部

```java
public boolean add(E e) {
    linkLast(e);
    return true;
}

public void addLast(E e) {
    linkLast(e);
}
```

查看linkLast(E e)方法

```java
/**
 * Links e as last element.
 */
void linkLast(E e) {
    // 当前的尾节点
    final Node<E> l = last;
    // 根据传入的值创建新节点，新节点的前指针指向当前的尾节点
    final Node<E> newNode = new Node<>(l, e, null);
    // 尾节点引用指向新节点
    last = newNode;
    // 尾节点为空的情况，说明链表为空，新加入的节点既是头节点也是尾节点
    if (l == null)
        first = newNode;
    else
        // 尾节点的后指针指向新节点
        l.next = newNode;
    // 更新链表长度
    size++;
    // 更新修改次数
    modCount++;
}
```

#### 在指定位置添加元素

```java
public void add(int index, E element) {
    // 校验index是否超出范围
    checkPositionIndex(index);
	
    if (index == size)
        // 插入位置位于尾部，使用插入尾部的方法
        linkLast(element);
    else
        // 找到index位置的节点，并调用linkBefore方法将新元素插入到该节点前的位置
        linkBefore(element, node(index));
}
```

查看node(index)方法，该方法作用是获取指定下标位置的Node节点，根据下标位置位于链表前一半还是后一半做了遍历方式的优化：

```java
/**
 * Returns the (non-null) Node at the specified element index.
 */
Node<E> node(int index) {
    // assert isElementIndex(index);
   	// 这里判断如果下标小于链表长度的一半，就从头节点开始向后遍历节点，返回index对应的节点
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        // 如果下标大于链表长度的一半，就从尾节点开始向前遍历节点，返回index对应的节点
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

查看linkBefore方法，该方法作用是在指定节点前插入新节点：

```java
/**
 * Inserts element e before non-null Node succ.
 */
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    // 指定节点的前节点
    final Node<E> pred = succ.prev;
    // 创建插入的节点，前指针指向前节点pred， 后指针指向指定节点succ
    final Node<E> newNode = new Node<>(pred, e, succ);
    // 更新指定节点succ的前指针指向插入的节点
    succ.prev = newNode;
    // 如果前节点为空，说明succ节点位于头部，新插入的节点变成头节点
    if (pred == null)
        first = newNode;
    else
        // 前节点的后指针指向插入的节点
        pred.next = newNode;
    // 更新链表长度
    size++;
    // 更新修改次数
    modCount++;
}
```

#### 添加集合元素

```java
/**
 * 添加集合的所有元素到链表尾部
 */
public boolean addAll(Collection<? extends E> c) {
    return addAll(size, c);
}
```

内部调用了addAll(size,c)，查看addAll(size,c)方法：

```java
/**
 * Inserts all of the elements in the specified collection into this
 * list, starting at the specified position.  Shifts the element
 * currently at that position (if any) and any subsequent elements to
 * the right (increases their indices).  The new elements will appear
 * in the list in the order that they are returned by the
 * specified collection's iterator.
 *
 * @param index index at which to insert the first element
 *              from the specified collection
 * @param c collection containing elements to be added to this list
 * @return {@code true} if this list changed as a result of the call
 * @throws IndexOutOfBoundsException {@inheritDoc}
 * @throws NullPointerException if the specified collection is null
 */
// 在指定index位置插入集合的所有元素到链表
public boolean addAll(int index, Collection<? extends E> c) {
    // 校验index下标合法性
    checkPositionIndex(index);

    Object[] a = c.toArray();
    // 要插入的集合元素数量
    int numNew = a.length;
    // 集合为空则返回false
    if (numNew == 0)
        return false;

    // index位置的节点为succ，succ的前节点为pred
    Node<E> pred, succ;
    if (index == size) {
        // 在尾部，则index位置的节点为null，pred为尾部节点
        succ = null;
        pred = last;
    } else {
        // 获取index位置的节点succ
        succ = node(index);
        // succ的前节点为pred
        pred = succ.prev;
    }

    // 将插入的集合元素分别插入到链表中
    for (Object o : a) {
        @SuppressWarnings("unchecked") E e = (E) o;
        // 创建节点，前指针指向pred
        Node<E> newNode = new Node<>(pred, e, null);
        if (pred == null)
            // 如果pred为null说明当前位置为头部，将头节点引用指向该插入的节点
            first = newNode;
        else
            // 设置前节点的后指针指向新节点
            pred.next = newNode;
        // 将pred更新为新节点位置，相当于下次插入的位置后移了一位，在新增的节点之后
        pred = newNode;
    }

    // 此时插入完毕，pred是最后插入的节点
    if (succ == null) {
        // 如果是在尾部，则last引用指向pred
        last = pred;
    } else {
        // pred的后指针指向原index位置的节点succ
        pred.next = succ;
        // 原index位置的节点succ前指针指向新的pred节点
        succ.prev = pred;
    }

    // 更新链表长度
    size += numNew;
    // 更新修改次数
    modCount++;
    return true;
}
```

#### 添加头节点

```java
/**
 * Links e as first element.
 */
private void linkFirst(E e) {
    final Node<E> f = first;
    // 创建新节点作为新的头节点,后指针指向原头节点
    final Node<E> newNode = new Node<>(null, e, f);
    // 更新first引用到新头节点
    first = newNode;
    if (f == null)
        // 原头节点为空,说明是空链表,新的头节点插入后也成为了尾节点
        last = newNode;
    else
        // 原头节点的前指针指向新头节点
        f.prev = newNode;
    // 更新链表长度
    size++;
    // 更新修改次数
    modCount++;
}
```

### 删除节点

删除节点主要有三个方法：

#### 删除指定的非空节点 

E unlink(Node<E> x)

```java
/**
 * 删除非空节点x
 */
E unlink(Node<E> x) {
    // assert x != null;
    final E element = x.item;
    // x的后节点
    final Node<E> next = x.next;
    // x的尾节点
    final Node<E> prev = x.prev;

    // 前节点为空说明x位于链表头部,删除x后后节点成为头节点
    if (prev == null) {
        first = next;
    } else {
        // x前节点的后指针指向x后节点
        prev.next = next;
        // 并将x的前指针修改为null
        x.prev = null;
    }

    // x后节点为null说明x位于链表尾部,删除x后x的前节点成为尾节点
    if (next == null) {
        last = prev;
    } else {
        // 将x的后节点的前指针指向x的前节点
        next.prev = prev;
        // 将x的后指针赋值为null
        x.next = null;
    }

    // 最后把x的item也赋值为null
    x.item = null;
    // 更新链表长度
    size--;
    // 更新修改次数
    modCount++;
    return element;
}
```

#### 删除头节点

E unlinkFirst(Node<E> f)

```java
/**
 * Unlinks non-null first node f.
 */
private E unlinkFirst(Node<E> f) {
    // assert f == first && f != null;
    final E element = f.item;
    // 头节点的后节点
    final Node<E> next = f.next;
    f.item = null;
    f.next = null; // help GC
    // 删除头节点后,后节点成为新的头节点
    first = next;
    if (next == null)
        // 删除头节点后链表为空了,将last引用也设置为null
        last = null;
    else
        // 将新头节点的前指针置为null
        next.prev = null;
    // 更新链表长度
    size--;
    // 更新修改次数
    modCount++;
    return element;
}
```

#### 删除尾节点

E unlinkLast(Node<E> l)

```java
/**
 * Unlinks non-null last node l.
 */
private E unlinkLast(Node<E> l) {
    // assert l == last && l != null;
    final E element = l.item;
    // 尾节点的前节点prev
    final Node<E> prev = l.prev;
    l.item = null;
    l.prev = null; // help GC
    // 删除尾节点后,prev成为新的尾节点
    last = prev;
    if (prev == null)
        // 链表为空,将头节点引用也设置为null
        first = null;
    else
        // 将新的尾节点的后指针设置为null
        prev.next = null;
    // 更新链表长度
    size--;
    // 更新修改次数
    modCount++;
    return element;
}
```



## 总结

LinkedList内部实现了6种主要的新增和删除节点的方法：`void linkFirst(E e)`、`void linkLast(E e)`、`linkBefore(E e, Node<E> succ)`、`E unlinkFirst(Node<E> f)`、`E unlinkLast(Node<E> l)`、`E unlink(Node<E> x)`。其他方法的实现方式大部分都是依赖这几种方法实现的，通过这几个方法就能熟悉LinkedList内部的基本操作原理。
