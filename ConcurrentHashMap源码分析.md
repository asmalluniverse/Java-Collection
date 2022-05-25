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
- put方法流程总结
put方法实际上是调用的putVal方法，

该方法做了5件事：
1. 如果key或者value为空，抛出异常。
2. 计算key的hash值：spread(key.hashCode()) （和hashmap有略微的区别，多了一个于运算，保证最高位的 1 个 bit 位总是0
(h ^ (h >>> 16)) & HASH_BITS
(h = key.hashCode()) ^ (h >>> 16)
）
3. binCount用来记录所在table数组中的桶的中链表的个数，后面会用于判断是否链表过长需要转红黑树
4. for循环（cas操作），执行新增/修改逻辑。直到put成功插入数据才会跳出。
 - 4.1 如果table没有初始化，则初始化table(lazy的过程)。第一次调用put的时候就是没有初始化的。
 - 4.2 如果已经初始化了，根据数组长度减1再对hash值取余得到在node数组中位于哪个下标，用tabAt获取数组中该下标的元素，如果该元素为空，直接将put的值包装成Node用casTabAt方法放入数组内这 个下标的位置中，这个时候它是这个桶中链表的头节点。这里的tabAt获取元素和casTabAt写入元素都是使用的CAS保证原子性。
 - 4.3  如果头结点hash值为-1，则为ForwardingNode结点（ForwardingNode相当于一个占位符，状态值为-1，后面扩容会说到），说明正再扩容， 调用hlepTransfer帮助扩容。
 - 4.4 如果，当前位置已经有元素了，且没有进行扩容操作，那么就使用synchronized锁住该下标的元素，即是锁住该桶的头节点，这样其他写操作会等待该资源。双重锁检测，因为多线程操作，任何一个时刻可能都存在竞争问题，再次判断该桶的头节点是不是被改过了。
如果修改过了，则进入下一次for循环，否则，把该节点放入链表或者红黑树中
    - 4.4.1 如果桶表头的hash值>=0，遍历链表，每遍历一次binCount计数器加一，如果遇到节点hash值相同，key相同，看是否需要更新value。如果到链表尾部都没有遇到相同的，就生成Node挂在链表尾部，该Node成为一个新的链尾。
    - 4.4.2 如果桶的头节点是个TreeBin，调用putTreeVal方法用红黑树的形式添加节点或者更新相同hash、key的值。
- 4.5 如果链表长度>需要树化的阈值（默认是8），调用treeifyBin方法将链表转换为红黑树（而这个方法中会判断数组值是否大于64，如果没有大于64则只扩容）。
- 4.6 如果是修改，不是新增，则返回被修改的原值
- 5. addCount方法计数器加1，完成新增后，table扩容，就是这里面触发


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
```java 
// x为1,check代表链表上的元素个数
private final void addCount(long x, int check) {
 CounterCell[] as; long b, s;
 //此处要进入if有两种情况
 //1.数组不为空，说明数组已经被创建好了。
 //2.若数组为空，说明数组还未创建，很有可能竞争的线程非常少，因此就直接 CAS 操作 baseCount
 //若 CAS 成功，则方法跳转到 (2)处，若失败，则需要考虑给当前线程分配一个格子（指CounterCell对象）
 if ((as = counterCells) != null ||
  !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
  CounterCell a; long v; int m;
  //字面意思，是无竞争，这里先标记为 true，表示还没有产生线程竞争
  boolean uncontended = true;
  //这里有三种情况，会进入 fullAddCount 方法
  //1.若数组为空，进方法 (1)
  //2.ThreadLocalRandom.getProbe() 方法会给当前线程生成一个随机数（可以简单的认为也是一个hash值）
  //然后用随机数与数组长度取模，计算它所在的格子。若当前线程所分配到的格子为空，进方法 (1)。
  //3.若数组不为空，且线程所在格子不为空，则尝试 CAS 修改此格子对应的 value 值加1。
  //若修改成功，则跳转到 (3)，若失败，则把 uncontended 值设为 fasle，说明产生了竞争，然后进方法 (1)
  if (as == null || (m = as.length - 1) < 0 ||
   (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
   !(uncontended =
     U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
   //方法(1), 这个方法的目的是让当前线程一定把 1 加成功。情况更多，更复杂，稍后讲。
   fullAddCount(x, uncontended);
   return;
  }
  //(3)能走到这，说明数组不为空，且修改 baseCount失败，
  //且线程被分配到的格子不为空，且修改 value 成功。
  //但是这里没明白为什么小于等于1，就直接返回了，这里我怀疑之前的方法漏掉了binCount=0的情况。
  //而且此处若返回了，后边怎么判断扩容？（存疑）
  if (check <= 1)
   return;
  //计算总共的元素个数
  s = sumCount();
 }
 //(2)这里用于检查是否需要扩容（下边这部分很多逻辑不懂的话，等后边讲完扩容，再回来看就理解了）
 if (check >= 0) {
  Node<K,V>[] tab, nt; int n, sc;
  //若元素个数达到扩容阈值，且tab不为空，且tab数组长度小于最大容量
  while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
      (n = tab.length) < MAXIMUM_CAPACITY) {
   //这里假设数组长度n就为16，这个方法返回的是一个固定值，用于当做一个扩容的校验标识
   //可以跳转到最后，看详细计算过程，0000 0000 0000 0000 1000 0000 0001 1011
   int rs = resizeStamp(n);
   //若sc小于0，说明正在扩容
   if (sc < 0) {
       //sc的结构类似这样，1000 0000 0001 1011 0000 0000 0000 0001
    //sc的高16位是数据校验标识，低16位代表当前有几个线程正在帮助扩容,RESIZE_STAMP_SHIFT=16
    //因此判断校验标识是否相等，不相等则退出循环
    //sc == rs + 1,sc == rs + MAX_RESIZERS 这两个应该是用来判断扩容是否已经完成，但是计算方法存疑
    //感兴趣的可以看这个地址，应该是一个 bug ，
    // https://bugs.java.com/bugdatabase/view_bug.do?bug_id=JDK-8214427
    //nextTable=null 说明需要扩容的新数组还未创建完成
    //transferIndex这个参数小于等于0，说明已经不需要其它线程帮助扩容了，
    //但是并不说明已经扩容完成，因为有可能还有线程正在迁移元素。稍后扩容细讲就明白了。
    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
     sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
     transferIndex <= 0)
     break;
    //到这里说明当前线程可以帮助扩容，因此sc值加一，代表扩容的线程数加1
    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
     transfer(tab, nt);
   }
   //当sc大于0，说明sc代表扩容阈值，因此第一次扩容之前肯定走这个分支，用于初始化新表 nextTable
   //rs<<16
   //1000 0000 0001 1011 0000 0000 0000 0000
   //+2
   //1000 0000 0001 1011 0000 0000 0000 0010
   //这个值，转为十进制就是 -2145714174，用于标识，这是扩容时，初始化新表的状态，
   //扩容时，需要用到这个参数校验是否所有线程都全部帮助扩容完成。
   else if (U.compareAndSwapInt(this, SIZECTL, sc,
           (rs << RESIZE_STAMP_SHIFT) + 2))
    //扩容，第二个参数代表新表，传入null，则说明是第一次初始化新表(nextTable)
    transfer(tab, null);
   s = sumCount();
  }
 }
}

```

### FullAddCount()方法
```java
private final void fullAddCount(long x, boolean wasUncontended) {
 int h;
 //如果当前线程的随机数为0，则强制初始化一个值
 if ((h = ThreadLocalRandom.getProbe()) == 0) {
  ThreadLocalRandom.localInit();      // force initialization
  h = ThreadLocalRandom.getProbe();
  //此时把 wasUncontended 设为true，认为无竞争
  wasUncontended = true;
 }
 //用来表示比 contend （竞争）更严重的碰撞，若为true，表示可能需要扩容，以减少碰撞冲突
 boolean collide = false;                // True if last slot nonempty
 //循环内，外层if判断分三种情况，内层判断又分为六种情况
 for (;;) {
  CounterCell[] as; CounterCell a; int n; long v;
  //1. 若counterCells数组不为空。  建议先看下边的2和3两种情况，再回头看这个。 
  if ((as = counterCells) != null && (n = as.length) > 0) {
   // (1) 若当前线程所在的格子（CounterCell对象）为空
   if ((a = as[(n - 1) & h]) == null) {
    if (cellsBusy == 0) {    
     //若无锁，则乐观的创建一个 CounterCell 对象。
     CounterCell r = new CounterCell(x); 
     //尝试加锁
     if (cellsBusy == 0 &&
      U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
      boolean created = false;
      //加锁成功后，再 recheck 一下数组是否不为空，且当前格子为空
      try {               
       CounterCell[] rs; int m, j;
       if ((rs = counterCells) != null &&
        (m = rs.length) > 0 &&
        rs[j = (m - 1) & h] == null) {
        //把新创建的对象赋值给当前格子
        rs[j] = r;
        created = true;
       }
      } finally {
       //手动释放锁
       cellsBusy = 0;
      }
      //若当前格子创建成功，且上边的赋值成功，则说明加1成功，退出循环
      if (created)
       break;
      //否则，继续下次循环
      continue;           // Slot is now non-empty
     }
    }
    //若cellsBusy=1，说明有其它线程抢锁成功。或者若抢锁的 CAS 操作失败，都会走到这里，
    //则当前线程需跳转到(9)重新生成随机数，进行下次循环判断。
    collide = false;
   }
   /**
   *后边这几种情况，都是数组和当前随机到的格子都不为空的情况。
   *且注意每种情况，若执行成功，且不break，continue，则都会执行(9)，重新生成随机数，进入下次循环判断
   */
   // (2) 到这，说明当前方法在被调用之前已经 CAS 失败过一次，若不明白可回头看下 addCount 方法，
   //为了减少竞争，则跳转到⑨处重新生成随机数，并把 wasUncontended 设置为true ，认为下一次不会产生竞争
   else if (!wasUncontended)       // CAS already known to fail
    wasUncontended = true;      // Continue after rehash
   // (3) 若 wasUncontended 为 true 无竞争，则尝试一次 CAS。若成功，则结束循环，若失败则判断后边的 (4)(5)(6)。
   else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
    break;
   // (4) 结合 (6) 一起看，(4)(5)(6)都是 wasUncontended=true，且CAS修改value失败的情况。
   //若数组有变化，或者数组长度大于等于当前CPU的核心数，则把 collide 改为 false
   //因为数组若有变化，说明是由扩容引起的；长度超限，则说明已经无法扩容，只能认为无碰撞。
   //这里很有意思，认真思考一下，当扩容超限后，则会达到一个平衡，即 (4)(5) 反复执行，直到 (3) 中CAS成功，跳出循环。
   else if (counterCells != as || n >= NCPU)
    collide = false;            // At max size or stale
   // (5) 若数组无变化，且数组长度小于CPU核心数时，且 collide 为 false，就把它改为 true，说明下次循环可能需要扩容
   else if (!collide)
    collide = true;
   // (6) 若数组无变化，且数组长度小于CPU核心数时，且 collide 为 true，说明冲突比较严重，需要扩容了。
   else if (cellsBusy == 0 &&
      U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
    try {
     //recheck
     if (counterCells == as) {// Expand table unless stale
      //创建一个容量为原来两倍的数组
      CounterCell[] rs = new CounterCell[n << 1];
      //转移旧数组的值
      for (int i = 0; i < n; ++i)
       rs[i] = as[i];
      //更新数组
      counterCells = rs;
     }
    } finally {
     cellsBusy = 0;
    }
    //认为扩容后，下次不会产生冲突了，和(4)处逻辑照应
    collide = false;
    //当次扩容后，就不需要重新生成随机数了
    continue;                   // Retry with expanded table
   }
   // (9)，重新生成一个随机数，进行下一次循环判断
   h = ThreadLocalRandom.advanceProbe(h);
  }
  //2.这里的 cellsBusy 参数非常有意思，是一个volatile的 int值，用来表示自旋锁的标志，
  //可以类比 AQS 中的 state 参数，用来控制锁之间的竞争，并且是独占模式。简化版的AQS。
  //cellsBusy 若为0，说明无锁，线程都可以抢锁，若为1，表示已经有线程拿到了锁，则其它线程不能抢锁。
  else if (cellsBusy == 0 && counterCells == as &&
     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
   boolean init = false;
   try {    
    //这里再重新检测下 counterCells 数组引用是否有变化
    if (counterCells == as) {
     //初始化一个长度为 2 的数组
     CounterCell[] rs = new CounterCell[2];
     //根据当前线程的随机数值，计算下标，只有两个结果 0 或 1，并初始化对象
     rs[h & 1] = new CounterCell(x);
     //更新数组引用
     counterCells = rs;
     //初始化成功的标志
     init = true;
    }
   } finally {
    //别忘了，需要手动解锁。
    cellsBusy = 0;
   }
   //若初始化成功，则说明当前加1的操作也已经完成了，则退出整个循环。
   if (init)
    break;
  }
  //3.到这，说明数组为空，且 2 抢锁失败，则尝试直接去修改 baseCount 的值，
  //若成功，也说明加1操作成功，则退出循环。
  else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
   break;                          // Fall back on using base
 }
}

```
