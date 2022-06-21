### 概述
ArrayList 是 java 集合框架中比较常用的数据结构了。继承自 AbstractList，实现了 List 接口。底层基于数组实现容量大小动态变化。允许 null 的存在。同时还实现了 RandomAccess、Cloneable、Serializable 接口，所以ArrayList 是支持快速访问、复制、序列化的。

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151727520.png)



### 基础属性
```
//初始化容量
private static final int DEFAULT_CAPACITY = 10;
//空实例数组
private static final Object[] EMPTY_ELEMENTDATA = {};
//默认大小的空实例数组，在第一次调用ensureCapacityInternal时会初始化长度为18
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
//存放元数据数组
transient Object[] elementData;
//数组当前的元素数量
private int size;

//带容量参数的构造函数
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                initialCapacity);
    }
}

//不带容量参数则使用默认大小的空实例数组
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```
一些基本属性，参考下图理解。

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151727198.png)




###  get方法
```
@SuppressWarnings("unchecked")
//直接根据索引位置返回元素
E elementData(int index) {
    return (E) elementData[index];
}

//根据索引获取元素
public E get(int index) {
    //校验索引是否越界
    rangeCheck(index);
    ////校验索引是否越界（底层elementData是数组）
    return elementData(index);
}
```
很简单，由于底层是数组实现的，先检查下索引是否越界，然后直接返回对应索引位置的元素即可。



### set方法
```
//用指定的元素(element)替换指定位置( index)的元素
public E set(int index, E element) {
    rangeCheck(index);  //校验索引是否越界

    E oldValue = elementData(index); //1根据index获取指定位置的元素
    elementData[index] = element; //用传入的element替换index位置的元素
    return oldValue; //返回index位置原来的元素
}
```
* 校验索引是否越界

* 根据index获取指定位置的元素

* 用传入的element替换index位置的元素

* 返回index位置原来的元素

  ​


### add方法
```
//增加一个元素
public boolean add(E e) {
    //将modCount+1,并校验添加元素是否需要扩容
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    //在数组尾部添加元素，并将size+1
    elementData[size++] = e;
    return true;
}

//将指定的元素(element)插入此列表中的指定位置(index)。
//将index位置及后面的所有元素(如果有的话)向右移动一个位置
public void add(int index, E element) {
    //校验索引是否越界
    rangeCheckForAdd(index);
    //将modCount+1,并校验添加元素是否需要扩容
    ensureCapacityInternal(size + 1);
    ////将index位置及之后的所有元素向右移动-个位 置(为要添加的元素腾出1个位置)
    System.arraycopy(elementData, index, elementData, index + 1,
            size - index);
    elementData[index] = element;  //index位置填入要添加的元素
    size++;  //元素数量+1
}
```

#### add(E e)：
调用ensureCapacityInternal方法（下文有详解），如果数组还没初始化，则进行初始化；如果已经初始化了，则将modCount+1，并校验添加元素后是否需要扩容。
在elementData数组尾部添加元素即可（size位置）。

#### add(int index, E element)：
* 检查索引是否越界，再调用ensureCapacityInternal方法，将modCount+1，并校验添加元素后是否需要扩容。


* 将index位置及之后的所有元素向右移动一个位置（为要添加的元素腾出1个位置）。


* 将index位置设置为element元素，并将size+1。

  ​

add(int index, E element)的过程如下图所示。

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151728631.png)





### remove方法

```

//删除列表中index位置的元素，将index位置后面的所有元素向左移一个位置
public E remove(int index) {

    //校验索引是否越界
    rangeCheck(index);
    //修改次数+1
    modCount++;
    // index位置的元素，也就是将要被移除的元素
    E oldValue = elementData(index);
    
    //计算需要移动的元素个数，例口: size为10， index为9， 此时numMoved为0，则无需移动元素,
    //因为此时index为9的元素刚好是最后一个元素，直接执行下面的代码，将索引为9的元素赋值为空即可
    int numMoved = size - index - 1;
    //如果需要移动元素
    if (numMoved > 0)
        //将index+1位置及之后的所有元素，向左移动一个位置
        System.arraycopy(elementData, index+1, elementData, index,
                numMoved);
    //将size-1位置的元素赋值为空(因为. 上面将元素左移了，所以size-1位 置的元素为重复的，将其移除)
    elementData[--size] = null;
    //返回index位置原来的元素
    return oldValue;
}

//如果存在与入参相同的元素，则从该列表中删除指定元素的第一个匹配项;如果列表不包含元素，则不变。
public boolean remove(Object o) {
    //如果入参元素为空，则遍历数组查找是否存在元素为空，
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                //如果存在则调用fastRemove将该元素移除，
                fastRemove(index);
                //并返回true表示移除成功
                return true;
            }
        //如果入参元素不为空，则遍历数组查找是否存在元素与入参元素使用equals比较返回true
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                //如果存在则调用fastRemove将该元素移除，
                fastRemove(index);
                //并返回true表示移除成功
                return true;
            }
    }
    //不存在目标元素，返回false
    return false;
}

//供上面的remove方法调用，直接删除掉index位置的元素；与remove(int index)方法一样
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                numMoved);
    elementData[--size] = null; // clear to let GC do its work
}
```

#### remove(int index)：
* 检查索引是否越界，将modCount+1，拿到索引位置index的原元素。
* 计算需要移动的元素个数。
* 如果需要移动，将index+1位置及之后的所有元素，向左移动一个位置。
* 将size-1位置的元素赋值为空（因为上面将元素左移了，所以size-1位置的元素为重复的，将其移除）。

#### remove(Object o)：
* 如果入参元素为空，则遍历数组查找是否存在元素为空，如果存在则调用fastRemove将该元素移除，并返回true表示移除成功。
* 如果入参元素不为空，则遍历数组查找是否存在元素与入参元素使用equals比较返回true，如果存在则调用fastRemove将该元素移除，并返回true表示移除成功。
  *否则，不存在目标元素，则返回false。



`调用remove方法， 会， 且只会 删除第一个与传入对象通过equals方法判断相等的元素。如果传入null， 则删除掉第一个null元素`
`所以， 如果自定义类想要使用remove方法从列表删除某个指定值对象， 还需要实现该类型自己的equals方法才行！`
`且删除只能从后往前删，从前往后删，删不干净`


#### fastRemove(int index)：跟remove(int index)类似
* 将modCount+1，并计算需要移动的元素个数。
* 如果需要移动，将index+1位置及之后的所有元素，向左移动一个位置。
* 将size-1位置的元素赋值为空（因为上面将元素左移了，所以size-1位置的元素为重复的，将其移除）。

#### remove(int index)方法的过程如下图所示。

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151729969.jpeg)





### clear方法
```
//删除此列表中的所有元素。
public void clear() {
    //修改次数+1
    modCount++;
    
    //遍历数组将所有元素清空
    for (int i = 0; i < size; i++)
        elementData[i] = null;
    //元素数量赋0
    size = 0;
}
```
遍历数组将所有元素清空即可。



### 扩容
上文add方法在添加元素之前会先调用ensureCapacityInternal方法，主要是有两个目的：1.如果没初始化则进行初始化；2.校验添加元素后是否需要扩容。
```
private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

private static int calculateCapacity(Object[] elementData, int minCapacity) {
    //校验当前数组是否为DEFAULTCAPACITY_EMPTY_ ELEMENTDATA,
    //如果是则将minCapacity设为DEFAULT_CAPACITY,
    //主要是给DEFAULTCAPACITY_EMPTY_ELEMENTDATA设置初始容量
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;//修改次数+1

    //如果添加该元素后的大小超过数组大小，则进行扩容
    if (minCapacity - elementData.length > 0)
        //进行扩容
        grow(minCapacity);
}

//数组允许的最大长度
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;

//数组扩容
private void grow(int minCapacity) {
    //原来的容量
    int oldCapacity = elementData.length;
    //新容量 = 老容量 + 老容量/2
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    //如果新容量比minCapacity小，
    if (newCapacity - minCapacity < 0)
        //则将新容量设为minCapacity,兼容初始化情况
        newCapacity = minCapacity;
    //如果新容量比最大允许容量大，
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        //则调用hugeCapacity方法设置个合适的容量
        newCapacity = hugeCapacity(minCapacity);
    //将原数组元素拷贝到一个容量为newCapacity的新数组(使用System.arraycopy)，
    //并且将elementData设置为新数组
    elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError(); //越界
    //如果minCapacity大于MAX_ARRAY_SIZE, 则返回Integer.MAX_VALUE, 否则返回MAX_ARRAY_SIZE
    return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
}
```

**ensureCapacityInternal(int minCapacity)**：此方法就是为默认大小的空实例数组DEFAULTCAPACITY_EMPTY_ELEMENTDATA而写的，判断数组是否为DEFAULTCAPACITY_EMPTY_ELEMENTDATA，如果是则将minCapacity设置为DEFAULT_CAPACITY。

**ensureExplicitCapacity(int minCapacity)**：将modCount+1，并校验minCapacity是否大于当前数组的容量，如果大于则调用grow方法进行扩容。

**grow(int minCapacity)**：
将数组新容量设置为老容量的1.5倍。
两个if判断，第一个是对DEFAULTCAPACITY_EMPTY_ELEMENTDATA初始化的兼容，第二个是对超过MAX_ARRAY_SIZE的兼容。
调用Arrays.copyOf方法创建长度为newCapacity的新数组，并将老数组的数据复制给新数组，并将elementData赋值为新数组。

**grow(int minCapacity)的过程如下图所示。**

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151730959.jpeg)



#### Iterator()
```
 public Iterator<E> iterator() {
        return new Itr();
} 
```

在调用list.iterator()的时候返回的是一个Itr对象，它是ArrayList中的一个内部类。
主要是实现了Iterator的三个方法：

#### hasNext()、next()、remove()
```
private class Itr implements Iterator<E> {
        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        //expectedModCount的初值为modCount
        //设置此直的原因是：
        //在使用迭代器遍历集合的时候同时修改集合元素。因为ArrayList被设计成非同步的，所以理所当然会抛异常。
        int expectedModCount = modCount;
    
        public boolean hasNext() {
            //hasNext的判断条件为cursor!=size，就是当前迭代的位置不是数组的最大容量值就返回true
            return cursor != size;
        }
    
        @SuppressWarnings("unchecked")
        public E next() {
            //会先调用checkForComodification来检查expectedModCount和modCount是否相等
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
    
        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();
    
            try {
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }
    
        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
```




### 坑一：删除一个null元素，只会删除第一个null元素；如果删除自定义元素，需要自己实现equals方法
```
//如果存在与入参相同的元素，则从该列表中删除指定元素的第一个匹配项;如果列表不包含元素，则不变。
public boolean remove(Object o) {
    //如果入参元素为空，则遍历数组查找是否存在元素为空，
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                //如果存在则调用fastRemove将该元素移除，
                fastRemove(index);
                //并返回true表示移除成功
                return true;
            }
        //如果入参元素不为空，则遍历数组查找是否存在元素与入参元素使用equals比较返回true
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                //如果存在则调用fastRemove将该元素移除，
                fastRemove(index);
                //并返回true表示移除成功
                return true;
            }
    }
    //不存在目标元素，返回false
```
调用remove方法， 会， 且只会 删除第一个与传入对象通过equals方法判断相等的元素。
如果传入null， 则删除掉第一个null元素。
所以， 如果自定义类想要使用remove方法从列表删除某个指定值对象， 还需要实现该类型自己的equals方法才行！






### 坑二：ArrayList是可以顺序删除节点的，但是！如果使用普通for循环，必须是从后往前删。不能从前往后删。
我们先来看一下【错误示范】：
```
ArrayList list=new ArrayList();
list.add("a");
list.add("b");
list.add("c");

System.out.println("删除前："+list.toString());

//顺序删除节点错误示范：从前往后删----会删不干净
for (int i=0;i<list.size();i++){
    list.remove(i);
}
System.out.println("删除后："+list.toString());
```
删除后结果
```
删除前：[a, b, c]
删除后：[b]
```

**【出错原因分析】**：
要顺序删除ArrayList的全部节点，如果我们从前往后的顺序删除，先删除【0】位置的数据，但是由于删除的时候是从后往前挪一位进行删除的，所以【0】的位置又会被下一个位置的数据覆盖上，实际上【0】还是有数据的。再画一张图来方便大家理解：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151731551.png)

**【正确的做法】**：
要想顺序删除ArrayList的所有节点，且采用普通的for循环，那只能从后往前删，这样就不会出问题。
```
//通过一般for循环，必须从后往前删除！
for (int i=(arraylist.size()-1);i>=0;i--){
    arraylist.remove(i);
}
```






坑三：每当我们使用迭代器遍历元素时，如果使用迭代器以外的方法修改了元素内容（如删除元素），那就会抛出ConcurrentModificationException的异常。

错误示例：
		ArrayList arrayList = new ArrayList();
	    arrayList.add("a");
	    arrayList.add("b");
	    arrayList.add("c");
	
	    System.out.println("移除前：" + arrayList);
	    Iterator<String> iterator = arrayList.iterator();
	    while (iterator.hasNext()) {
	        if ("c".equals(iterator.next())) {
	            arrayList.remove("c");
	        }
	    }
	    System.out.println("移除后：" + arrayList);
	
	    //注意增强for使用的也是迭代器
	    //所以一下这种操作也会报ConcurrentModificationException
	    //for (Object o : arrayList) {
	    //    arrayList.remove(o);
	    //}
	    //System.out.println("移除后2：" + arrayList);


报错
Exception in thread "main" java.util.ConcurrentModificationException
	at java.util.ArrayList$Itr.checkForComodification(ArrayList.java:909)
	at java.util.ArrayList$Itr.next(ArrayList.java:859)
	at com.xichuan.spring.dev.TestJava.main(TestJava.java:21)

先分析一下报错原因：
在我们使用 ArrayLis 的 iterator() 方法获取到迭代器进行遍历时，会把 ArrayList 当前状态下的 modCount 赋值给 ArrayListIterator类的 expectedModCount 属性。
如果我们在迭代过程中，使用了 ArrayList 的 remove()方法，这时 modCount 就会加 1 ，但是迭代器中的expectedModCount 并没有变化，当我们再使用迭代器的next()方法时，它就会报ConcurrentModificationException的错。



来自：[ArrayList详解](https://mp.weixin.qq.com/s/-zVj4f14xYISufeZEsSunw)



