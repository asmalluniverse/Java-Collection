### 1.概述
ConcurrentHashMap是JUC包下提供的工具，相对于HashMap来说是线程安全的，以jdk1.8为例，内部使用（数组 + 链表 + 红黑树）的结构来存储元素。相比于同样线程安全的HashTable来说，效率有了很大的提升。

### 2.为什么要使用ConcurrentHashMap
可以从两个方面考虑该问题，**性能**和**安全性**之间的平衡

### 3.1.7 1.8之间的区别
1. 新增了方法：
(1)compute
(2)meger
(3)computeIfAbsent
(4)computeIfPresent 

2. 存储结构和实现
(1)1.7 使用segment实现分段锁，锁的粒度较大1.8进行了优化
(2)1.8引入了红黑树，解决链表长度过长导致查询效率低 时间复杂度由O(n) -> O(lngn)

### 3.put()源码分析
```java
public V put(K key, V value) {
	return putVal(key, value, false);
}


/** Implementation for put and putIfAbsent */
final V putVal(K key, V value, boolean onlyIfAbsent) {
	//chm中 key value都不能为Null(否则会违反二义性)
	if (key == null || value == null) throw new NullPointerException();
	//计算key的Hash值，作为保存的下标位置
	int hash = spread(key.hashCode());
  //定义变量用于判断数组是否需要转化为红黑树
	int binCount = 0;
	//这里是一个自旋 + cas 操作，保证多线程的情况下是线程安全的
	for (Node<K,V>[] tab = table;;) {
		Node<K,V> f; int n, i, fh;
		//先判断是否初始化，如果没有初始化先进行初始化的操作
		if (tab == null || (n = tab.length) == 0)
			tab = initTable();
			//tabAt也是一个cas操作，用于计算f的值， 如果当前位置为空，说明还没有元素则直接通过casTabAt方法cas 操作保存，保证原子性
		else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
			if (casTabAt(tab, i, null,
						 new Node<K,V>(hash, key, value, null)))
				break;                   // no lock when adding to empty bin
		}
		//正在扩容需要去协助扩容
		else if ((fh = f.hash) == MOVED)
			tab = helpTransfer(tab, f);
			//如果当前位置已经存在值，又会判断 是放入链表中还是红黑树中 ，因为可能存在hash 冲突
		else {
			V oldVal = null;
			//f 代表当前下标，也就是对一个节点进行加锁，粒度更细，和1.7 有区别
			synchronized (f) {
			    //这里又进行了重新判断，因为时多线程的情况下，f的值可能被其他线程所修改，这里是一个双重检锁判断
				if (tabAt(tab, i) == f) {
					//针对链表进行处理
					if (fh >= 0) {
						binCount = 1;
						//这个for循环的意思就是找到链表中是否存在相同的key，如果存在则覆盖，否则找到链表的尾部(尾部肯定为空)，把元数插入到链表的尾部
						//binCount 则用来记录链表的长度，用于判断是否需要扩容的操作
						for (Node<K,V> e = f;; ++binCount) {
							K ek;
							//如果当前的key相同，直接覆盖
							if (e.hash == hash &&
								((ek = e.key) == key ||
								 (ek != null && key.equals(ek)))) {
								oldVal = e.val;
								if (!onlyIfAbsent)
									e.val = value;
								break;
							}
							//如果链表中的Key不同，直接插入链表尾部
							Node<K,V> pred = e;
							if ((e = e.next) == null) {
								pred.next = new Node<K,V>(hash, key,
														  value, null);
								break;
							}
						}
					}
					//针对红黑树进行处理
					else if (f instanceof TreeBin) {
						Node<K,V> p;
						binCount = 2;
						if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
													   value)) != null) {
							oldVal = p.val;
							if (!onlyIfAbsent)
								p.val = value;
						}
					}
				}
			}
			//如果bigCount>=8 则调用treeifyBin(tab, i);方法， 该方法并不一定会把链表转化为红黑树，
			//还有一个条件就是需要map 中的元数个数大于64 才会把链表转化位红黑树，否则进行扩容操作
			if (binCount != 0) {
				if (binCount >= TREEIFY_THRESHOLD)
					treeifyBin(tab, i);
				if (oldVal != null)
					return oldVal;
				break;
			}
		}
	}
  //该方法也是一个核心方法，主要有两个作用，扩容和统计元素个数
	addCount(1L, binCount);
	return null;
}
```

### 4.initTable()
初始化table,因为可能存在多线程竞争的情况，但是这里并不是通过简单的加锁进行实现的
```java
private final Node<K,V>[] initTable() {
	Node<K,V>[] tab; int sc;
	table 为空或者长度为零，进行初始化
	while ((tab = table) == null || tab.length == 0) {
		//如果sizeCtl的值小于零代表有线程已经再初始化了，其他线程只能等待
		if ((sc = sizeCtl) < 0)
			Thread.yield(); // lost initialization race; just spin
		//因为可能存在多线程同时初始化的问题，所以这里使用的是cas 操作，把sizeCtl设置为-1,如果成功了，该线程可以进行初始化
		else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
			try {
				if ((tab = table) == null || tab.length == 0) {
					//判断是否给了map的容量，没有使用默认值
					int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
					@SuppressWarnings("unchecked")
					Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
					table = tab = nt;
					//n = 16  sc =12  这里是位运算 计算扩容的阈值 (16用2进制表示是： 0001 0000 ，右移两位 0000 0100 为4)
					sc = n - (n >>> 2);
				}
			} finally {
				sizeCtl = sc;
			}
			break;
		}
	}
	return tab;
}	
```
1. sizeCtl参数的作用
sizeCtl 默认为0，用来控制table的初始化和扩容操作,如果sizeCtl 为-1 则说明正在初始化

sizeCtl的几种状态：
```shell
sizeCtl >= 0              sizeCtl =-1          sizeCtl = 0.75* n
表示初始化容量          				               存储扩容阈值             sizeCtl < -1 
未初始化  -------------->  初始化中  --------------->  正常  ------到达扩容阈值需要进行扩容操作--------->  扩容中
```                                                                扩容完毕sizeCtl回复正常

SIZECTL中获取的 sizeCtl的地址偏移值，是在static中初始化的

第一步：
是判断SizeCtl是不是<0? 判断是否正在初始化。如果是那就Thread.yield() 实则就只允许一个线程操作，是个自选的操作

第二步：
U.compareAndSwapInt(this, SIZECTL, sc, -1)? 这个cas的判断地址并操作为-1
unsafe方法中的Cas 判断了地址偏移，（SIZECTL早在static就以初始化好了）
如果比较为True 那就更新为-1。原子操作保证了安全。（不明白CAS的移步百度查询Unsafe的Cas）
同时volatile保证了顺序与内存可见性。

总结：在第一步进行判断，是不能保证并发安全的，如果两个线程同时进入，就需要Cas去保证安全，并且原子变更数值
当然sizeCtl 不仅仅在init中使用，还在扩容中使用。纵观整个类会发现大量的Unsafe的方法。虽然官方并不推荐使用.

### 5.addCount()
