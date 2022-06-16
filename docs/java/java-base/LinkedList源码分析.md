
### 概述
LinkedList也是我们经常使用的集合，本文就LinkedList的几个主要方法展开介绍，并结合几个图片来介绍几个重要操作。

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202206151745243.png)



### 基础属性
```
transient int size = 0;  //节点数量

/**
 * Pointer to first node.
 * Invariant: (first == null && last == null) ||
 *            (first.prev == null && first.item != null)
 */
transient Node<E> first;  //第一个节点（头节点）

/**
 * Pointer to last node.
 * Invariant: (first == null && last == null) ||
 *            (last.next == null && last.item != null)
 */
transient Node<E> last;//最后一个节点（尾节点）

//Node的数据结构
private static class Node<E> {
    E item;  //存放的对象
    Node<E> next; //下一个节点
    Node<E> prev; //上一个节点

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```
基本数据结构图如下：

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202206151745258.png)




### add方法
```
public boolean add(E e) {
    //调用linkLast方法，将节点添加到尾部
    linkLast(e);
    return true;
}

//在index位置插入节点， 节点值为element
public void add(int index, E element) {
    //校验index是否越界
    checkPositionIndex(index);
    //如果索引为size,即将element插 入链表尾部
    if (index == size)
        //调用linkLast将节点插入链表尾部
        linkLast(element);
     //否则，将element插入原index位置节点的前面，
    //即:将element插入index位置，将原index位置节点移到index+1的位置
    else
        //调用linkBefore插入index位置
        linkBefore(element, node(index));
}
```
**add(E e)**：调用linkLast方法将元素添加到尾部（linkLast方法详解见下文）
**add(int index, E element)**：

* 检查index是否越界
* 比较index与size，如果index==size，则代表插入位置为链表尾部，调用linkLast方法（linkLast方法详解见下文），否则调用linkBefore方法（LinkBefore方法详解见下文）



### get方法
```
public E get(int index) {
    //校验index是否越界
    checkElementIndex(index);
    //根据index,调用node方法寻找目标节点，寻找目标节点的item
    return node(index).item;
}
```
根据index，调用node方法（见下文node方法详解）寻找目标节点，返回目标节点的item。



### node方法
```
//根据index位置寻找node
Node<E> node(int index) {
    //如果index < size/2, 则代表index在链表的前半部分，从头结点开始遍历
    if (index < (size >> 1)) {
        Node<E> x = first;
        //从first节点遍历，直到index位置
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
        //否则，index在链表的后半部分，从尾节点开始遍历
    } else {
        Node<E> x = last;
        //从last节 点遍历，直到index位 置
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```
如果index在链表的前半部，则从头节点开始遍历；否则从尾节点开始遍历。



### set方法
```
//替换index位置节点的值为element
public E set(int index, E element) {
    //检查index是否越界
    checkElementIndex(index);
    //根据index, 调用node方法寻找到目标节点
    Node<E> x = node(index);
    //节点的原值
    E oldVal = x.item;
    //将节点的item属性替换为element
    x.item = element;
    //返回节点原值
    return oldVal;
}
```
检查index是否越界
调用node方法寻找目标节点（见上文node方法详解）
将目标节点的item属性设为element



### remove方法
```
public boolean remove(Object o) {
    //如果o为空，则遍历链表寻找item属性为空的节点
    if (o == null) {
        for (Node<E> x = first; x != null; x = x.next) {
            //如果目标节点存在
            if (x.item == null) {
                //则调用unlink方法将该节点移除
                unlink(x);
                return true;
            }
        }
        //如果o不为空， 则遍历链表寻找item属性跟o相同的节点
    } else {
        for (Node<E> x = first; x != null; x = x.next) {
            //如果目标节点存在
            if (o.equals(x.item)) {
                //则调用unlink方法将该节点移除
                unlink(x);
                return true;
            }
        }
    }
    return false;
}

//移除index位置的节点
public E remove(int index) {
    //检查index是否越界
    checkElementIndex(index);
    //移除index位置的节点
    return unlink(node(index));
}
```

**remove(Object o)**：

* 判断o是否为null，如果o为null，则遍历链表寻找item属性为空的节点，并调用unlink方法将该节点移除（unlink方法详解见下文）
* 如果o不为null, 则遍历链表寻找item属性跟o相同的节点，并调用unlink方法将该节点移除（unlink方法详解见下文）

**remove(int index)**：

* 检查index是否越界
* 调用unlink方法，移除index位置的节点（unlink方法详解见下文）



### clear方法
```
//清除链表的所有节点
public void clear() {
    // Clearing all of the links between nodes is "unnecessary", but:
    // - helps a generational GC if the discarded nodes inhabit
    //   more than one generation
    // - is sure to free memory even if there is a reachable Iterator
    //从头节点开始遍历，将所有节点的属性清空
    for (Node<E> x = first; x != null; ) {
        Node<E> next = x.next;
        x.item = null;
        x.next = null;
        x.prev = null;
        x = next;
    }
    //将头节点和尾节点设置为null
    first = last = null;
    //size清零
    size = 0;
    modCount++;
}
```
从first节点开始，遍历将所有节点的属性清空
将first节点和last节点设为null



### linkLast方法
```
//将e放到链表的最后一个节点
  void linkLast(E e) {
      //拿到当前的尾节点l节点
      final Node<E> l = last;
      //使用e创建一个新的节点newNode, prev属性为l节点，next 属性为null
      final Node<E> newNode = new Node<>(l, e, null);
      //将当前尾节点设置为上面新创建的节点newNode
      last = newNode;
      //如果l节点为空则代表当前链表为空，将newNode设置为头结点
      if (l == null)
          first = newNode;
      //否则将l节点的next属性设置为newtNode
      else
          l.next = newNode;
      size++;
      modCount++;
  }
```
* 拿到当前的尾节点 l 节点
* 使用e创建一个新的节点newNode，prev属性为l节点，next属性为null
* 将当前尾节点设置为上面新创建的节点newNode
* 如果l节点为空则代表当前链表为空, 将newNode设置为头结点，否则将l节点的next属性设置为newNode

**过程如图所示**

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202206151746289.png)




### linkBefore方法
```
// 将节点e插入节点succ前面
void linkBefore(E e, Node<E> succ) {
    // assert succ != null;
    //拿到succ节点的prev节点，赋值给pred节点
    final Node<E> pred = succ.prev;
    //使用e创建一个新的节点newNode, 其中prev属性为pred节点，next属性为succ节点
    final Node<E> newNode = new Node<>(pred, e, succ);
    //将succ 节点的prev属性设置为newNode
    succ.prev = newNode;
    //如果pred节点为null,则代表succ 节点为头结点，
    //要把e插入succ前面，只 需将first设置为newNode
    if (pred == null)
        first = newNode;
     //否则将pred节点的next属性设为newNode
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```
* 拿到succ节点的prev节点
* 使用e创建一个新的节点newNode，其中prev属性为pred节点，next属性为succ节点
* 将succ节点的prev属性设置为newNode
* 如果pred节点为null，则代表succ节点为头结点，要把e插入succ前面，因此将first设置为newNode，否则将pred节点的next属性设为newNode

**过程如图所示**

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202206151747408.png)



### unlink方法

```
//移除链表上的x节点
E unlink(Node<E> x) {
    // x节点的值
    final E element = x.item;
    // x节点的下一个节点next节点
    final Node<E> next = x.next;
    // x节点的上一个节点prev节点
    final Node<E> prev = x.prev;
    //如果prev为空， 则代表x节点为头结点，则将first指向next即可
    if (prev == null) {
        first = next;
        //否则，x节点不为头结点，
    } else {
        //将prev节点的next属性指向x节点的next属性
        prev.next = next;
        //将x的prev属性清空
        x.prev = null;
    }

    ////如果next为空，则代表x节点为尾节点，则将last指向prev即可
    if (next == null) {
        last = prev;
        //否则，节点不为尾节点
    } else {
        //将next节点的prev属性指向x节点的prev属性
        next.prev = prev;
        //将x的next属性清空
        x.next = null;
    }
    //将x的值清空，以便垃圾收集器回收x对象
    x.item = null;
    size--;
    modCount++;
    return element;
}
```
* 定义element为x节点的值，next为x节点的下一个节点，prev为x节点的上一个节点
* 如果prev为空，则代表x节点为头结点，则将first指向next即可；否则，x节点不为头结点，将prev节点的next属性指向x节点的next属性，并将x的prev属性清空
* 如果next为空，则代表x节点为尾节点，则将last指向prev即可；否则，x节点不为尾节点，将next节点的prev属性指向x节点的prev属性，并将x的next属性清空
* 将x的item属性清空，以便垃圾收集器回收x对象

**过程如图所示**

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202206151749290.png)





来自：[LinkedList详解](https://mp.weixin.qq.com/s/tXK9sC1JENJYHjXAzCFDLg)