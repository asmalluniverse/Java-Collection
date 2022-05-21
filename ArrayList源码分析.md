### 1.ArrayList特点

1. 底层存储结构：数组，那么数组具有随意访问性，查询的时间复杂度是O（1）。

2. 特点：查询效率高，增删效率低，线程不安全，开发中使用频率高。
因为我们正常使用的场景中，大多数场景用来查询，不会涉及太频繁的增删，如果涉及频繁的增删，数据量不大且符合链表特性的场景可以使用LinkedList。

3. 类的关系
```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable

实现了RandomAccess接口，可以随机访问

实现了Cloneable接口，可以克隆

实现了Serializable接口，可以序列化、反序列化

实现了List接口，是List的实现类之一

实现了Collection接口，是Java Collections Framework成员之一

实现了Iterable接口，可以使用for-each迭代
```


4.属性
```java 
// 序列化版本UID
private static final long
        serialVersionUID = 8683452581122892189L;

/**
 * 默认的初始容量
 */
private static final int
        DEFAULT_CAPACITY = 10;

/**
 * 用于空实例的共享空数组实例
 * new ArrayList(0);
 */
private static final Object[]
        EMPTY_ELEMENTDATA = {};

/**
 * 用于提供默认大小的实例的共享空数组实例
 * new ArrayList();
 */
private static final Object[]
        DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

/**
 * 存储ArrayList元素的数组缓冲区
 * ArrayList的容量，是数组的长度
 * 
 * non-private to simplify nested class access
 */
transient Object[] elementData;

/**
 * ArrayList中元素的数量
 */
private int size;
```

### 2.构造方法分析
- 无参构造和有参构造

1. 无参:构建一个空的数组
```java
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }
```
	
2. 有参： 

```java  
	public ArrayList(int initialCapacity) {
	//如果initialCapacity > 0，就创建一个新的长度是initialCapacity的数组
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
			//如果initialCapacity == 0，就使用EMPTY_ELEMENTDATA
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
			//其他情况，initialCapacity不合法，抛出异常
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }
```
	
通过无参构造方法的方式ArrayList()初始化，则赋值底层数Object[] elementData为一个默认空数组Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {}所以数组容量为0，是一个lazy的过程，
只有真正对数据进行添加add时，才分配默认DEFAULT_CAPACITY = 10的初始容量。

### 3.扩容原理分析

- 场景：
我们现在有一个长度为10的数组，现在我们要新增一个元素，发现已经满了，那ArrayList会怎么做呢？

- 过程分析：
1. 第一步他会重新定义一个长度为10+10/2的数组也就是新增一个长度为15的数组（**1.5倍进行扩容**）。
2. 然后把原数组的数据，原封不动的复制到新数组中，这个时候再把指向原数的地址换到新数组，ArrayList就这样完成了一次改头换面。

注意：扩容方法会在add方法时进行判断是否需要扩容，分析完扩容方法后add方法就不是问题了。
```java
public boolean add(E e) {
	//扩容操作
	ensureCapacityInternal(size + 1);  // Increments modCount!!
	elementData[size++] = e;
	return true;
}

private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}
    
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    // 如果当前的这个数组是DEFAULTCAPACITY_EMPTY_ELEMENTDATA，则直接扩容到DEFAULT_CAPACITY，也就是说，第一次add元素的时候会扩容到10
	//前面说过只有在真正添加元素的时候才会分配容量
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
       return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++;
    // minCapacity：插入元素后的元素个数 
	//elementData.length: 当前的数据长度
	//判断插入了当前元素后是否超过了数组的容量，是的话调用grow方法扩容 
    if (minCapacity - elementData.length > 0)
      grow(minCapacity);
}

private void grow(int minCapacity) {
    // 获取原本数组的大小 
    int oldCapacity = elementData.length;
    // 新的容量为原本的1.5倍 oldCapacity >> 1 即 oldCapacity / 2  
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    // 判断扩容后的容量是否满足当前的插入需求 比如新容量是15 但是当前插入后元素为16 则新容量变为16
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    // 判断新容量是否超过了最大数组容量（Integer.MAX_VALUE - 8）超过了就是用Integer.MAX_VALUE大小，否则使用（Integer.MAX_VALUE - 8）
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // 调用Arrays的进行数组的复制 
    elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0)
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ? Integer.MAX_VALUE : MAX_ARRAY_SIZE;
}


```


### 4.add方法分析
add元素的时候可以指定index进行增加也可以不指定，那么就加入到数组最后
增加元素前会判断是否需要扩容操作。
```java
public boolean add(E e) {
	//扩容操作
	ensureCapacityInternal(size + 1);  // Increments modCount!!
	elementData[size++] = e;
	return true;
}
```	
```java	
	public void add(int index, E element) {
	//判断索引是否有效
	rangeCheckForAdd(index);

	ensureCapacityInternal(size + 1);  // Increments modCount!!
	//调用的是native方法
	System.arraycopy(elementData, index, elementData, index + 1,
					 size - index);
	elementData[index] = element;
	size++;
}
	
```	

- System.arraycopy 方法在做什么呢？
	
	比如有下面这样一个数组我需要在index 5的位置去新增一个元素A
	那从代码里面我们可以看到，他复制了一个数组，是从index 5的位置开始的，然后把它放在了index 5+1的位置
	给我们要新增的元素腾出了位置，然后在index的位置放入元素A就完成了新增的操作了
该方法效率不是很高，小数据量还行，如果数据量很大的情况效力很差。

### 5.删除方法remove
删除方法可以删除数组中的某个元素或者删除指定索引位置的元素

```java
// 参数是下标
// 返回值是被删除的元素值
public E remove(int index) {
		// 首先检查下标是否越界，rangeCheck(index)是个内部私有方法，如果下标越界，将抛出异常
        rangeCheck(index);
		// 当前列表的修改次数加1
		// 这个参数是用于迭代器fail-fast机制的
		// 当在遍历列表时，如果对列表进行删除操作时，将会抛出异常
        modCount++;
        // 根据下标获取对应的值
        E oldValue = elementData(index);
		// 计算要删除位置的元素后还有几个元素，用于后面的操作
        int numMoved = size - index - 1;
        // 如果要删除的元素不是最后一个元素
        if (numMoved > 0)
        	// 这是jdk的一个本地方法，用于将一个数组从指定位置复制到目标数组的指定位置
        	// 其中numMoved就是要复制的个数，也就是被删除元素后面的元素个数
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
         // 把列表大小减1，并把最后一个元素置空，让垃圾收集器把它回收
         // 这里如果不置空，它将会保存着一个引用，那么垃圾收集器将无法回收它，可能会造成内存泄漏
        elementData[--size] = null; // clear to let GC do its work
		// 将被删除的值返回
        return oldValue;
    }

	
// 参数是对象
// 返回值为是否删除成功
public boolean remove(Object o) {
		// 如果对象为空(ArrayList允许元素为空)
        if (o == null) {
            for (int index = 0; index < size; index++)
                if (elementData[index] == null) {
                	// 这里调用了一个私有删除方法
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

//该方法就是指定索引位置删除的，其实就是了解如何实现指定索引位置删除元素
private void fastRemove(int index) {
        modCount++;
        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work
    }

继续打个比方，我们现在要删除下面这个数组中的index5这个位置
1 2 3 4 5 6 7 8 9
新建一个数组
那代码他就复制一个index5+1开始到最后的数组，然后把它放到index开始的位置
1 2 3 4 6 7 8 9
```
	
	


### 6.常见问题总结
1. ArrayList（int initialCapacity）会不会初始化数组大小？
答案是不会的

List list = new ArrayList(10);
System.out.println(list.size());
list.set(5,1);

上面的代码会报错：
```java
0
Exception in thread "main" java.lang.IndexOutOfBoundsException: Index: 5, Size: 0
	at java.util.ArrayList.rangeCheck(ArrayList.java:657)
	at java.util.ArrayList.set(ArrayList.java:448)
```

尽管我们设置了初始化大小但是ArrayList里面的 私有属性 private int size; 并不会设置为10
而且将构造函数与initialCapacity结合使用，然后使用set（）会抛出异常，尽管该数组已创建，但是大小设置不正确。
使用sureCapacity（）也不起作用，因为它基于elementData数组而不是大小。
还有其他副作用，这是因为带有sureCapacity（）的静态DEFAULT_CAPACITY。
进行此工作的唯一方法是在使用构造函数后，根据需要使用add（）多次。
大家可能有点懵，我直接操作一下代码，大家会发现我们虽然对ArrayList设置了初始大小，但是我们打印List大小的时候还是0，我们操作下标set值的时候也会报错，数组下标越界。

那么这样设计的好处，我个人理解是，为了节省内存空间，等到你真正使用的时候才给你初始化大小。从jvm优化的层面考虑，可以理解为代码级别的内存优化设计。


2. ArrayList是线程安全的么？
当然不是，线程安全版本的数组容器是Vector。

Vector的实现很简单，就是把所有的方法统统加上synchronized就完事了。

你也可以不使用Vector，用Collections.synchronizedList把一个普通ArrayList包装成一个线程安全版本的数组容器也可以，原理同Vector是一样的，就是给所有的方法套上一层synchronized。

3. ArrayList用来做队列合适么？

队列一般是FIFO（先入先出）的，如果用ArrayList做队列，就需要在数组尾部追加数据，数组头部删除数组，反过来也可以。
但是无论如何总会有一个操作会涉及到数组的数据搬迁，这个是比较耗费性能的。
结论：ArrayList不适合做队列。

4. 那数组适合用来做队列么？
数组是非常合适的。

比如ArrayBlockingQueue内部实现就是一个环形队列，它是一个定长队列，内部是用一个定长数组来实现的。
另外著名的Disruptor开源Library也是用环形数组来实现的超高性能队列，具体原理不做解释，比较复杂。
简单点说就是使用两个偏移量来标记数组的读位置和写位置，如果超过长度就折回到数组开头，前提是它们是定长数组。

5. ArrayList的遍历和LinkedList遍历性能比较如何？

论遍历ArrayList要比LinkedList快得多，ArrayList遍历最大的优势在于内存的连续性，CPU的内部缓存结构会缓存连续的内存片段，可以大幅降低读取内存的性能开销。	

6. 为什么空实例默认数组有的时候是EMPTY_ELEMENTDATA，而又有的时候是DEFAULTCAPACITY_EMPTY_ELEMENTDATA

看完构造方法、添加方法、扩容方法之后，上文第1个问题就会有答案。new ArrayList()会将elementData 赋值为 DEFAULTCAPACITY_EMPTY_ELEMENTDATA，
new ArrayList(0)会将elementData 赋值为 EMPTY_ELEMENTDATA，EMPTY_ELEMENTDATA添加元素会扩容到容量为1，而DEFAULTCAPACITY_EMPTY_ELEMENTDATA扩容之后容量为10。

7. 为什么elementData要被transient修饰
elementData之所以用transient修饰，是因为JDK不想将整个elementData都序列化或者反序列化，而只是将size和实际存储的元素序列化或反序列化，从而节省空间和时间。


8. ArrayList类注释翻译
类注释还是要看的，能给我们一个整体的了解这个类。我将ArrayList的类注释大概翻译整理了一下：
ArrayList是实现List接口的可自动扩容的数组。实现了所有的List操作，允许所有的元素，包括null值。
ArrayList大致和Vector相同，除了ArrayList是非同步的。
如果多个线程同时访问ArrayList的实例，并且至少一个线程会修改，必须在外部保证ArrayList的同步。
修改包括添加删除扩容等操作，仅仅设置值不包括。这种场景可以用其他的一些封装好的同步的list。
如果不存在这样的Object，ArrayList应该用Collections.synchronizedList包装起来最好在创建的时候就包装起来，来保证同步访问。
iterator()和listIterator(int)方法是fail-fast的，如果在迭代器创建之后，列表进行结构化修改，迭代器会抛出ConcurrentModificationException。
面对并发修改，迭代器快速失败、清理，而不是在未知的时间不确定的情况下冒险。
请注意，快速失败行为不能被保证。通常来讲，不能同步进行的并发修改几乎不可能做任何保证。因此，写依赖这个异常的程序的代码是错误的，快速失败行为应该仅仅用于防止bug。

### 7.总结
1. ArrayList底层的数据结构是数组

2. ArrayList可以自动扩容，不传初始容量或者初始容量是0，都会初始化一个空数组，但是如果添加元素，会自动进行扩容，所以，创建ArrayList的时候，给初始容量是必要的

3. Arrays.asList()方法返回的是的Arrays内部的ArrayList，用的时候需要注意

4. subList()返回内部类，不能序列化，和ArrayList共用同一个数组

5. 迭代删除要用，迭代器的remove方法，或者可以用倒序的for循环

6. ArrayList重写了序列化、反序列化方法，避免序列化、反序列化全部数组，浪费时间和空间

7. elementData不使用private修饰，可以简化内部类的访问
