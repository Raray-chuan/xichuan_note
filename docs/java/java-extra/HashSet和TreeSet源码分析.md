## 一.概述
 Set集合与List一样，都是继承自Collection接口，常用的实现类有HashSet和TreeSet。值得注意的是，HashSet是通过HashMap来实现的而TreeSet是通过TreeMap来实现的，所以HashSet和TreeSet都没有自己的数据结构，具体可以归纳如下：
        1. Set集合中的元素不能重复，即元素唯一
        2. HashSet按元素的哈希值存储，所以是无序的，并且最多允许一个null对象
        3. TreeSet按元素的大小存储，所以是有序的，并且不允许null对象
        4. Set集合没有get方法，所以只能通过迭代器（Iterator）来遍历元素，不能随机访问



## 二. HashSet源码分析
**基础属性**
```
static final long serialVersionUID = -5024744406713321676L;

private transient HashMap<E,Object> map;

// Dummy value to associate with an Object in the backing Map
//Object类型的引用PRESENT则是对应于map中的value的一个虚拟值，没有实际意义。
private static final Object PRESENT = new Object();
```
    观察源码，我们知道HashSet的数据是存储在HashMap的实例对象map中的，并且对应于map中的key，而Object类型的引用PRESENT则是对应于map中的value的一个虚拟值，没有实际意义。联想到HashMap的一些特性：无序存储、key值唯一等等，我们就可以很自然地理解Set集合元素不能重复以及HashSet无序存储的特性了。


 **构造器（四种）**
```

//  HashSet() 空的构造器，初始化一个空的HashMap
public HashSet() {
    map = new HashMap<>();
}

//HashSet(Collection<? extends E> c) 传入一个子集c，用于初始化HashMap
public HashSet(Collection<? extends E> c) {
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    addAll(c);
}

// HashSet(int initialCapacity, float loadFactor) 初始化一个空的HashMap，并指定初始容量和加载因子
public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
}

//  HashSet(int initialCapacity) 初始化一个空的HashMap，并指定初始容量
public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
}
```



**插入元素:add(E e) 插入指定元素**
调用HashMap的put方法实现

```
public boolean add(E e) {
    //为true的话，表名元素不重复，否则重复。
    return map.put(e, PRESENT)==null;
}
```



**查找元素:contains(Object o) 判断集合中是否包含指定的元素**
调用HashMap的containsKey方法实现

```
public boolean contains(Object o) {
    return map.containsKey(o);
}
```



**iterator()**
由于HashSet的实现类中没有get方法，所以只能通过迭代器依次遍历，而不能随机访问（调用HashMap中keySet的迭代器实现）

```
public Iterator<E> iterator() {
    return map.keySet().iterator();
}
```

 

**修改元素:**
由于HashMap中的key值不能修改，所以HashSet不能进行修改元素的操作




 **remove(Object o)** 
 remove(Object o) 删除指定元素（调用HashMap中的remove方法实现，返回值为true或者false）
```
public boolean remove(Object o) {
	return map.remove(o)==PRESENT;
}
```

 

**clear()**
clear() 清空元素（调用HashMap中的clear方法实现，没有返回值）

```
public void clear() {
   map.clear();
}
```









## 三. TrssSet源码分析
**基础属性**
    TreeSet是SortedSet接口的唯一实现类。前面说过，TreeSet没有自己的数据结构而是通过TreeMap实现的，所以`TreeSet也是基于红黑二叉树的一种存储结构，所以TreeSet不允许null对象，并且是有序存储的`（默认升序）。

```
private transient NavigableMap<E,Object> m;

// Dummy value to associate with an Object in the backing Map

private static final Object PRESENT = new Object();

```




 **构造器（四种）**
```
//TreeSet() 空的构造器，初始化一个空的TreeMap，默认升序排列
public TreeSet() {
    this(new TreeMap<E,Object>());
}
//TreeSet(Comparator<? super E> comparator) 传入一个自定义的比较器，常常用于实现降序排列
public TreeSet(Comparator<? super E> comparator) {
    this(new TreeMap<>(comparator));
}
//TreeSet(Collection<? extends E> c) 传入一个子集c，用于初始化TreeMap对象，默认升序
public TreeSet(Collection<? extends E> c) {
    this();
    addAll(c);
}
//TreeSet(SortedSet<E> s) 传入一个有序的子集s，用于初始化TreeMap对象，采用子集的比较器
public TreeSet(SortedSet<E> s) {
    this(s.comparator());
    addAll(s);
}
```

示例
```
//自定义一个比较器，实现降序排列
  Set<Integer> treeSet = new TreeSet<Integer>(new Comparator<Integer>() {
   @Override
   public int compare(Integer o1, Integer o2) {
//    return 0;  //默认升序
    return o2.compareTo(o1);//降序
   }
  });
  treeSet.add(200);
  treeSet.add(120);
  treeSet.add(150);
  treeSet.add(110);
  for (Iterator iterator = treeSet.iterator(); iterator.hasNext();) {
   Integer integer = (Integer) iterator.next();
   System.out.print(integer+" ");
  } //200 150 120 110

ArrayList<Integer> list = new ArrayList<Integer>();
  list.add(300);
  list.add(120);
  list.add(100);
  list.add(150);
  System.out.println(list); //[300, 120, 100, 150]

  //传入一个子集，默认升序排列
  TreeSet<Integer> treeSet = new TreeSet<Integer>(list);
  for (Iterator iterator = treeSet.iterator(); iterator.hasNext();) {
   Integer integer = (Integer) iterator.next();
   System.out.print(integer+" ");
  }//100 120 150 300

/*
   * 传入一个有序的子集，采用子集的比较器
   * 注意子集的类型必须是SortedSet及其子类或者实现类，否则将采用默认的比较器
   * 所以此处subSet的类型也可以是TreeSet。
        */

  SortedSet<Integer> subSet = new TreeSet<Integer>(new Comparator<Integer>() {
   @Override
   public int compare(Integer o1, Integer o2) {
//    return 0;  //默认升序
    return o2.compareTo(o1);//降序
   }
  });
  subSet.add(200);
  subSet.add(120);
  subSet.add(150);
  subSet.add(110);
  for (Iterator iterator = subSet.iterator(); iterator.hasNext();) {
   Integer integer = (Integer) iterator.next();
   System.out.print(integer+" ");
  } //200 150 120 110

  System.out.println();
  Set<Integer> treeSet = new TreeSet<Integer>(subSet);
  for (Iterator iterator = treeSet.iterator(); iterator.hasNext();) {
   Integer integer = (Integer) iterator.next();
   System.out.print(integer+" ");
  }//200 150 120 110

  System.out.println();
  treeSet.add(500);
  treeSet.add(100);
  treeSet.add(105);
  for (Iterator iterator = treeSet.iterator(); iterator.hasNext();) {
   Integer integer = (Integer) iterator.next();
   System.out.print(integer+" ");
  }//500 200 150 120 110 105 100
```



**add(E e)  /   addAll(Collection<? extends E> c)** 
add(E e) 插入指定的元素（调用TreeMap的put方法实现）
addAll(Collection<? extends E> c) 插入一个子集c

```
public boolean add(E e) {
    ////为true的话，表名元素不重复，否则重复。
    return m.put(e, PRESENT)==null;
}

public  boolean addAll(Collection<? extends E> c) {
    // Use linear-time version if applicable
    if (m.size()==0 && c.size() > 0 &&
        c instanceof SortedSet &&
        m instanceof TreeMap) {
        SortedSet<? extends E> set = (SortedSet<? extends E>) c;
        TreeMap<E,Object> map = (TreeMap<E, Object>) m;
        Comparator<?> cc = set.comparator();
        Comparator<? super E> mc = map.comparator();
        if (cc==mc || (cc != null && cc.equals(mc))) {
            map.addAllForTreeSet(set, PRESENT);
            return true;
        }
    }
    return super.addAll(c);
}
```

示例
```
ArrayList<Integer> list = new ArrayList<Integer>();
  list.add(300);
  list.add(120);
  list.add(100);
  list.add(150);
  System.out.println(list); //[300, 120, 100, 150]
  Set<Integer> treeSet = new TreeSet<Integer>();
  //插入一个子集，默认升序
  treeSet.addAll(list);
  for (Iterator iterator = treeSet.iterator(); iterator.hasNext();) {
   Integer integer = (Integer) iterator.next();
   System.out.print(integer+" ");
  }//100 120 150 300
```

 

**contains(Object o)** 
 	contains(Object o) 判断集合中是否包含指定对象（调用TreeMap的containsKey方法实现）
    与HashSet一样，TreeSet的实现类中没有get方法，所以只能通过迭代器依次遍历，而不能随机访问（调用TreeMap中keySet的迭代器实现）。

```
public boolean contains(Object o) {
    return m.containsKey(o);
}
```



**修改元素**
 TreeSet不能进行修改元素的操作，原因与HashSet一样。




**remove(Object o)** 
remove(Object o) 删除指定元素（调用TreeMap中的remove方法实现，返回true或者false）
```
public boolean remove(Object o) {
return m.remove(o)==PRESENT;
}
```


**clear()** 
清空元素（调用TreeMap中的clear方法实现，无返回值）
```
public void clear() {
    m.clear();
}
```