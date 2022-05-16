### HashMap源码分析
##### 1. 概述
1. 在JDK1.8之前，HashMap使用数组+链表实现，即使用链表处理冲突，同一hash值的节点都存储在一个链表里。但是当位于一个桶中的元素较多，即hash值相等的元素较多时，通过key值依次查找的效率较低。而JDK1.8中，HashMap采用数组+链表+红黑树实现，当链表长度超过阈值（8）时且数组中的元素个数大于64，会将链表转换为红黑树，这样大大减少了查找时间。

2. 类的继承关系
```java
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable
```
可以看到HashMap继承自父类（AbstractMap），实现了Map、Cloneable、Serializable接口。其中，Map接口定义了一组通用的操作；Cloneable接口则表示可以进行拷贝，在HashMap中，实现的是浅层次拷贝，即对拷贝对象的改变会影响被拷贝的对象；Serializable接口表示HashMap实现了序列化，即可以将HashMap对象保存至本地，之后可以恢复状态。

3. 类的核心属性
```java
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V>, Cloneable, Serializable {
    // 序列号
    private static final long serialVersionUID = 362498820763181265L;    
    // 默认的初始容量是16
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;   
    // 最大容量
    static final int MAXIMUM_CAPACITY = 1 << 30; 
    // 默认的填充因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    // 当桶(bucket)上的结点数大于这个值时,以及对应的table的最小大小为64，即MIN_TREEIFY_CAPACITY ；这两个条件都满足，会链表会转红黑树
    static final int TREEIFY_THRESHOLD = 8; 
    // 当桶(bucket)上的结点数小于这个值时树转链表
    static final int UNTREEIFY_THRESHOLD = 6;
    // 桶中结构转化为红黑树对应的table的最小大小
    static final int MIN_TREEIFY_CAPACITY = 64;
    // 存储元素的数组，总是2的幂次倍
    transient Node<k,v>[] table; 
    // 存放具体元素的集
    transient Set<map.entry<k,v>> entrySet;
    // 存放元素的个数，注意这个不等于数组的长度。
    transient int size;
    // 每次扩容和更改map结构的计数器
    transient int modCount;   
    // 临界值 当实际大小(容量*填充因子)超过临界值时，会进行扩容
    int threshold;
    // 填充因子
    final float loadFactor;
}
```
4. 构造方法
```java
public HashMap(int initialCapacity, float loadFactor) {
    // 初始容量不能小于0，否则报错
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                            initialCapacity);
    // 初始容量不能大于最大值，否则为最大值
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    // 填充因子不能小于或等于0，不能为非数字
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                            loadFactor);
    // 初始化填充因子                                        
    this.loadFactor = loadFactor;
    // 初始化threshold大小,返回大于initialCapacity的最小的二次幂数值
    this.threshold = tableSizeFor(initialCapacity);    
}
```
5. put方法分析
首先我们先对整个流程进行核心的讲解，然后跟着这个思路再去理解源码：

- 整体流程：
1. 首先判断存在键值对的数组table[i]是否为空或为null，否则执行resize()进行扩容；
2. 根据key计算得到key.hash = (h = k.hashCode()) ^ (h >>> 16) 值；
3. 根据key.hash计算得到桶数组的索引index = key.hash & (table.length - 1)，这样就找到该key的存放位置了：
① 如果该位置没有数据，用该数据新生成一个节点保存新数据，返回null；
② 如果该位置有数据是一个红黑树，那么执行相应的插入 / 更新操作；
③ 如果该位置有数据是一个链表，分两种情况一是该链表没有这个节点，另一个是该链表上有这个节点，注意这里判断的依据是key.hash是否一样：
如果该链表没有这个节点，那么采用尾插法新增节点保存新数据，返回null；如果该链表已经有这个节点了，那么找到该节点并更新新数据，返回老数据。
    这里还会有一个判断，如果当前链表长度超过阈值8调用treeifyBin(tab, hash);方法转化为红黑树。
4.插入成功后，判断实际存在的键值对数量size是否超多了最大容量threshold，如果超过，进行扩容。

- 源码解析

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
// 步骤①：tab为空则创建 
// table未初始化或者长度为0，进行扩容
if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;

// 步骤②：计算index，并对null做处理  
// (n - 1) & hash 确定元素存放在哪个桶中，桶为空，新生成结点放入桶中(此时，这个结点是放在数组中)
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);

    // 桶中已经存在元素
    else {
        Node<K,V> e; K k;

        // 步骤③：节点key存在，直接覆盖value 
        // 比较桶中第一个元素(数组中的结点)的hash值相等，key相等
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
              // 将第一个元素赋值给e，用e来记录
               //如果相等直接替换 e = p;
            e = p;

        //步骤④：判断该链为红黑树
        else if (p instanceof TreeNode)
            //如果是红黑树，放入树中
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);

// 步骤⑤：该链为链表       
 else {
            // 在链表最末插入结点，遍历链表，判断链表中是否存在该key，如果存在替换，否则加入链表尾部
            for (int binCount = 0; ; ++binCount) {
                // 在尾部插入新结点
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 结点数量达到阈值，转化为红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 判断链表中结点的key值与插入的元素的key值是否相等
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                //跳出循环
                    break;
                // 用于遍历桶中的链表，与前面的e = p.next组合，可以遍历链表                
                p = e;
            }
        }

        // 表示在桶中找到key值、hash值与插入元素相等的结点
        if (e != null) { // existing mapping for key
            // 记录e的value
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                //用新值替换旧值
                e.value = value;
            // 访问后回调
            afterNodeAccess(e);

            // 返回旧值
            return oldValue;
        }
    }

    ++modCount;
    if (++size > threshold)
        //进行扩容操作
        resize();
    afterNodeInsertion(evict);
    return null;
}
```
  ①.判断键值对数组table[i]是否为空或为null，否则执行resize()进行扩容；
  ②.根据键值key计算hash值得到插入的数组索引i，如果table[i]==null，直接新建节点添加，转向⑥，如果table[i]不为空，转向③；
  ③.判断table[i]的首个元素是否和key一样，如果相同直接覆盖value，否则转向④，这里的相同指的是hashCode以及equals；
  ④.判断table[i] 是否为treeNode，即table[i] 是否是红黑树，如果是红黑树，则直接在树中插入键值对，否则转向⑤；
  ⑤.遍历table[i]，判断链表长度是否大于8（且），大于8的话（且数组的长度大于64）把链表转换为红黑树，在红黑树中执行插入操作，否则进行链表的插入操作；遍历过程中若发现key已经存在直接覆盖value即可；
  ⑥.插入成功后，判断实际存在的键值对数量size是否超多了最大容量threshold，如果超过，进行扩容。
  
6. get方法分析
  get方法相对比较简单，就是获取数组中的元素：
  
- 整体流程  
1.判断数组是否为空，如果为空直接返回null
2.如果数组不为空，计算当前要查询的元素的下标，如果第一个元素就像等，返回第一个元素，否则从红黑树中查询或者从链表中遍历查找。

- 源码解析
  ```java
 public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
      // table已经初始化，长度大于0，根据hash寻找table中的项也不为空
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 桶中第一项(数组元素)相等
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        // 桶中不止一个结点
        if ((e = first.next) != null) {
            // 为红黑树结点
            if (first instanceof TreeNode)
                  // 在红黑树中查找
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 否则，在链表中查找
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
  ```
7. resize方法
该方法是一个比较核心的方法，主要涉及到两个核心思想：
1.扩容后的大小为原数组的两倍
2.扩容是采用高低位进行扩容的，也就是扩展后Node对象的位置要么在原位置，要么移动到原偏移量加上旧的数组长度的位置。

在分析扩容源码之前，我们先通过一个例子分析什么是高低位迁移
举个例子，假设table原长度是16，扩容后长度32，那么一个hash值在扩容前后的table下标是这么计算的：
我们知道计算下标位置是通过当前key的hash值与上数组长度减一。那么扩容后变化的就是数组的长度，而key的hash值是不变的。
原数组长度16，那么数组长度减一为15，扩容后的数组长度32，数组长度减一为31
他们的二进制分别为：
```xml
原数组：   0 1 1 1 1 1 
扩容后：   1 1 1 1 1 1
key hash :a b c d e f
```
hash值的每个二进制位用abcde来表示，那么，hash和新旧table按位与的结果，最后4位显然是相同的，唯一可能出现的区别就在第5位，也就是hash值的b所在的那一位，如果b所在的那一位是0，那么新table按位与的结果和旧table的结果就相同，反之如果b所在的那一位是1，则新table按位与的结果就比旧table的结果多了10000（二进制），而这个二进制10000就是旧table的长度16。
换言之，hash值的新散列下标是不是需要加上旧table长度，只需要看看hash值第5位是不是1就行了，位运算的方法就是hash值和10000（也就是旧table长度）来按位与，其结果只可能是10000或者00000。
知道这一点我们再通过源码去理解什么是高低位迁移：
```java
final Node<K,V>[] resize() {
//定义一个oldTab记录老数组
    Node<K,V>[] oldTab = table;
//计算老数组的长度
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
//老数组的扩容阈值
    int oldThr = threshold;
//newCap：新数组的长度 newThr 新数组的阈值
    int newCap, newThr = 0;
    if (oldCap > 0) {
//如果老数组的容量已经大于最大容量，修改数组阈值为最大整数，并返回老数组
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
//如果当前hash桶数组的长度在扩容后仍然小于最大容量 并且oldCap大于默认值16，则进行扩容操作，扩大为原来的两倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
//设置新数组的扩容阈值
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
//数组初始化会走该逻辑
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
//新建hash桶数组
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
//将新数组的值复制给旧的hash桶数组
    table = newTab;
    if (oldTab != null) {
//进行扩容操作，复制Node对象值到新的hash桶数组
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
//如果旧的hash桶数组在j结点处不为空，复制给e
            if ((e = oldTab[j]) != null) {
//将旧的hash桶数组在j结点处设置为空，方便gc
                oldTab[j] = null;
//如果e后面没有Node结点
                if (e.next == null)
//直接对e的hash值对新的数组长度求模获得存储位置
                    newTab[e.hash & (newCap - 1)] = e;
//如果e是红黑树的类型，那么添加到红黑树中
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order

// loHead，下标不变情况下的链表头
// loTail，下标不变情况下的链表尾
// hiHead，下标改变情况下的链表头
// hiTail，下标改变情况下的链表尾
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
//高低位迁移过程，见下面的注解
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```
正常情况下，计算节点在table中的下标的方法是：hash&(oldTable.length-1)，扩容之后，table长度翻倍，计算table下标的方法是hash&(newTable.length-1)，也就是hash&(oldTable.length*2-1)，于是我们有了这样的结论：这新旧两次计算下标的结果，要不然就相同，要不然就是新下标等于旧下标加上旧数组的长度。

- 流程总结：
1.首先计算新老数组的数组长度和扩容阈值
2.如果是初始化扩容那么新数组的长度是16扩容阈值是16*0.75=12
3.根据旧的数组是否为null进行判断是初始化操作还是扩容操作，如果为null直接返回新数组，否则进行扩容操作
4.扩容操作，遍历数组中的元数，如果当前数组下标存在元数，先进行赋值操作，把该位置的值赋值给新建的节点e,然后清空该位置的值方便gc，如果e后面没有节点，计算e的新数组下边，e的hash值于上新数组长度减一，如果e后面存在节点，是红黑树添加到红黑树中，否在对链表进行迁移，这里得迁移过程采用的是高低位迁移，提升效率，不用计算每个元素对应的索引位置，因为扩容后的元数不是在当前位置就是在当前数组下标加上旧数组长度的位置。


8. 常见面试题总结
- jdk1.7使用的头插法，为什么1.8修改为尾插法呢？
这种逆序的扩容方式在多线程时有可能出现环形链表，出现环形链表的原因大概是这样的：线程1准备处理节点，线程二把HashMap扩容成功，链表已经逆向排序，那么线程1在处理节点时就可能出现环形链表。
假设现在数组长度为2，下标1的位置出现了链表 3-> 7 -> 5，经过扩容后元数据3,7移动到了新的位置3，5还是在下标1处。

线程       数组下标
thread 1   0 :  (e)     (next)
           1 : 3:A --->  7:B ---> 5:C
           
thread 2   0 : 
           1 : 5:C
           2 :
           3 : 7:B ---> 3:A
线程1的e指向了key3,而next指向的值key7，而线程2rehash后指向了新的数组形成的链表。此时线程1再次进行执行，key3的next指向了key7，而线程2已经使用头插法把key7的next设置为key3，环出现了。

使用头插会改变链表的上的顺序，但是如果使用尾插，在扩容时会保持链表元素原本的顺序，就不会出现链表成环的问题了。

- 数组长度为什么是2的幂次方
假设现在数组的长度 length 可能是偶数也可能是奇数
length 为偶数时，length-1 为奇数，奇数的二进制最后一位是 1，这样便保证了 hash &(length-1) 的最后一位可能为 0，也可能为 1（这取决于 h 的值），即 & 运算后的结果可能为偶数，也可能为奇数，这样便可以保证散列的均匀性。
例如 length = 4，length - 1 = 3, 3 的 二进制是 11
若此时的 hash 是 2，也就是 10，那么 10 & 11 = 10（偶数位置）
hash = 3，即 11 & 11 = 11 （奇数位置）
而如果 length 为奇数的话，很明显 length-1 为偶数，它的最后一位是 0，这样 hash & (length-1) 的最后一位肯定为 0，即只能为偶数，这样任何 hash 值都只会被散列到数组的偶数下标位置上，这便浪费了近一半的空间
length = 3， 3 - 1 = 2，他的二进制是 10
10 无论与什么树进行 & 运算，结果都是偶数
因此，length 取 2 的整数次幂，是为了使不同 hash 值发生碰撞的概率较小，这样就能使元素在哈希表中均匀地散列。
           
- 1.7 vs 1.8
（1）JDK1.7用的是头插法，而JDK1.8及之后使用的都是尾插法，那么他们为什么要这样做呢？因为JDK1.7是用单链表进行的纵向延伸，当采用头插法时会容易出现逆序且环形链表死循环问题。但是在JDK1.8之后是因为加入了红黑树使用尾插法，能够避免出现逆序且链表死循环的问题。

（2）扩容后数据存储位置的计算方式也不一样：1. 在JDK1.7的时候是直接用hash值和需要扩容的二进制数进行&（这里就是为什么扩容的时候为啥一定必须是2的多少次幂的原因所在，因为如果只有2的n次幂的情况时最后一位二进制数才一定是1，这样能最大程度减少hash碰撞）（hash值 & length-1）

（3）在JDK1.7的时候是先扩容后插入的，这样就会导致无论这一次插入是不是发生hash冲突都需要进行扩容，如果这次插入的并没有发生Hash冲突的话，那么就会造成一次无效扩容，但是在1.8的时候是先插入再扩容的，优点其实是因为为了减少这一次无效的扩容，原因就是如果这次插入没有发生Hash冲突的话，那么其实就不会造成扩容，但是在1.7的时候就会急造成扩容

（4）而在JDK1.8的时候直接用了JDK1.7的时候计算的规律，也就是扩容前的原始位置+扩容的大小值=JDK1.8的计算方式，而不再是JDK1.7的那种异或的方法。但是这种方式就相当于只需要判断Hash值的新增参与运算的位是0还是1就直接迅速计算出了扩容后的储存方式。

- HashMap vs Hashtable
跟HashMap相比Hashtable是线程安全的，适合在多线程的情况下使用，但是效率可不太乐观。是对所有的方法上加锁实现线程安全，所以效率比较低。

相同点:
hashmap和Hashtable都实现了map、Cloneable（可克隆）、Serializable（可序列化）这三个接口

不同点:
1. 底层数据结构不同:jdk1.7底层都是数组+链表,但jdk1.8 HashMap加入了红黑树
2. Hashtable 是不允许键或值为 null 的，HashMap 的键值则都可以为 null。
3. 添加key-value的hash值算法不同：HashMap添加元素时，是使用自定义的哈希算法,而HashTable是直接采用key的hashCode()
4. 实现方式不同：Hashtable 继承的是 Dictionary类，而 HashMap 继承的是 AbstractMap 类。
5. 初始化容量不同：HashMap 的初始容量为：16，Hashtable 初始容量为：11，两者的负载因子默认都是：0.75。
6. 扩容机制不同：当已用容量>总容量 * 负载因子时，HashMap 扩容规则为当前容量翻倍，Hashtable 扩容规则为当前容量翻倍 +1。
7. 支持的遍历种类不同：HashMap只支持Iterator遍历,而HashTable支持Iterator和Enumeration两种方式遍历
8. 迭代器不同：HashMap的迭代器(Iterator)是fail-fast迭代器，而Hashtable的enumerator迭代器不是fail-fast的。所以当有其它线程改变了HashMap的结构（增加或者移除元素），将会抛出ConcurrentModificationException，但迭代器本身的remove()方法移除元素则不会抛出ConcurrentModificationException异常。但这并不是一个一定发生的行为，要看JVM。而Hashtable 则不会。
9. 部分API不同：HashMap不支持contains(Object value)方法，没有重写toString()方法,而HashTable支持contains(Object value)方法，而且重写了toString()方法
10. 同步性不同: Hashtable是同步(synchronized)的，适用于多线程环境,而hashmap不是同步的，适用于单线程环境。多个线程可以共享一个Hashtable；而如果没有正确的同步的话，多个线程是不能共享HashMap的。
