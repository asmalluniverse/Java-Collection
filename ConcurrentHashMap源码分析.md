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
                                                              扩容完毕sizeCtl回复正常
```  
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
- 流程步骤总结
addCount方法有两个入参，分别代表添加元数的个数和链表上元素的个数，该方法的主要作用有两个作用：**统计元素个数**以及**扩容操作**。
1. 首先会判断counterCells的数组是否为空，如果不为空说明数组已经创建好，会给当前线程生成一个随机数，然后用随机数和数组长度取模计算他所在的格子，如果所在的格子为空调用fullAddCount方法，如果当前线程分配的格子不为空，则尝试cas修改该格子的value值+1,如果修改成功计算总共的元素个数，如果修改失败，说明竞争激烈需要调用fullAddCount方法进行元素个数的统计。
2. 如果数组为空，说明还未创建，很有可能此时的竞争不是很激烈，直接通过cas操作baseCount进行元素个数的统计，如果cas操作成功，就要进行扩容的逻辑判断
3. 如果数组为空，但是cas操作失败，则需要考虑给当前线程分配一个格子（指CounterCell对象），执行fullAddCount方法。
4. fullAdCount方法后面分析，先把该方法中的后续扩容操作进行解析，首先扩容的条件是元素个数达到扩容阈值，且tab不为空，且tab数组长度小于最大容量则进入循环进行扩容，并且会把类中的sezeCtl变量赋值给局部变量sc。生成一个扩容标识为rs，rs = resizeStamp(n)
```java
sizeCtl ：默认为0，用来控制table的初始化和扩容操作
-1 代表table正在初始化
-N N对应的二进制的低16位数值为M，此时有M-1个线程进行扩容。
其余情况：
1、如果table未初始化，表示table需要初始化的大小。
2、如果table初始化完成，表示table的容量，默认是table大小的0.75倍
```
5. 如果sc<0说明正在扩容，sc的高16位是数据校验标识，低16位代表当前有几个线程正在帮助扩容，如果扩容已经完成即，sc=rs+1 或者sc==rs+MAX_RESIZERS 表示帮助线程线程已经达到最大值了，或者需要扩容的新数组还未创建完成或者transferIndex这个参数小于等于0，表示所有的transfer任务都被领取完了，没有剩余的hash桶来给自己自己好这个线程来做transfer，说明已经不需要其它线程帮助扩容了，但是并不说明已经扩容完成，因为有可能还有线程正在迁移元素。满足以上任何一个条件直接跳出循环。如果以上条件都不满足会调用cas操作让当前线程进行协助扩容，sc值加一，代表扩容的线程数加1，如果成功调用transfer方法进行扩容。
6. 如果sc>=0，说明当前还没有在进行扩容操作，因此第一次扩容之前肯定走这个分支，用于初始化新表 nextTable，会对sc进行cas运算 sc = rs<<16 + 2 ,即第一次扩容时会有一个+2的操作，那么扩容是否结束也是使用-2进行判断，后续在进行协助扩容的线程都是+1
7. 最后计算元素总和。整个addCount方法结束。

以上在addCount方法中有几个核心的方法 fullAddCount， sumCount， transfer，接着对这几个方法进行分析。

### suncount方法
- 流程步骤总结
该方法比较简单，这里就叙述整体的流程。  

该方法的目的就是计算元素总数，以baseCount这个值作为累加基准，遍历 counterCells 数组，得到每个对象中的value值，进行累加操作得到总和。
可能得到的总数不是准确的，考虑性能的问题，只保证最终一致性，因为该方法没有加任何锁的操作。

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
- 流程步骤总结

1. 先介绍该方法中的核心参数
```java
 //用来表示比 contend（竞争）更严重的碰撞，若为true，表示可能需要扩容，以减少碰撞冲突
 boolean collide  

//标记着是否有线程竞争，在addcount方法中传过来的可能是true 也可能是false
 boolean wasUncontended

//cellsBusy 参数非常有意思，是一个volatile的 int值，用来表示自旋锁的标志，可以类比 AQS 中的 state 参数，用来控制锁之间的竞争，并且是独占模式。简化版的AQS。cellsBusy 若为0，说明无锁，线程都可以抢锁，若为1，表示已经有线程拿到了锁，则其它线程不能抢锁。
 cellsBusy
```

上边的 addCount 方法还没完，如果元素个数没有统计成功，就走到 fullAddCount 这个方法了。  
1.如果countcells数组不为空，分为多种情况  
	a.如果当前线程生成的随机数所在的格子等于空，判断锁的状态，如果cellsBusy是0代表无锁，创建一个counterCell对象，把cellbusy通过cas操作修改为1，如果修改成功，再次检测数组是否为空且当前所在的格子是否为空，如果不为空，把创建的countcelss放入到当前格子，释放锁，created状态置为true，跳出循环。  
	b.如果当前线程生成的随机数所在的格子不为空：  
		(1)在addcount方法中通过cas操作设置失败，并把 wasUncontended 设置为true ，认为下一次不会产生竞争，对当前线程重新生成一个随机数，进入下次循环  
		(2)如果 wasUncontended为true，直接尝试使用cas对改格子操作，如果cas成功则跳出循环  
		(3)如果（2）中的cas操作失败，先判断数组是否有变化，如果有变化，或者数组长度已经超过当前cpu核心数把collide 置为 false  
		(4)如果数组没有变化且数组长度小于cpu核心数且 collide 为 false，就把它改为 true，说明下次循环可能需要扩容  
		(5)若数组无变化，且数组长度小于CPU核心数时，且 collide 为 true，说明冲突比较严重，需要扩容了。通过cas操作吧cellBusy设置为1，如果设置成功，会创建一个原来两倍的数组		    并进行数据的转移，转以结束后cellBusy设置为0并且collide = false;跳出循环。  

2.如果countcells数据为空，这里会有一个cellsBusy 参数，是一个volatile的 int值，用来表示自旋锁的标志，可以类比 AQS 中的 state 参数，用来控制锁之间的竞争，并且是独占模式。简化版的AQS。cellsBusy 若为0，说明无锁，线程都可以抢锁，若为1，表示已经有线程拿到了锁，则其它线程不能抢锁。如果此时cellsBusy为零，那么通过cas操作修改为1，如果修改成功，再次检测counterCells数组是否有变化，如果没有变化就初始化一个长度为2的数组，根据当前线程的随机数值，计算下标，只有两个结果 0 或 1，并初始化对象，初始化成功后释放锁，并且把init设置为初始化成功的标志，用于跳出循环。  
3.如果cellsBusy为1或者cas操作修改cellBusy失败，说明此时的数组还是为空，那么就直接通过cas操作修改baseCount的值，如果修改成功跳出循环。否则继续按条件执行1 2 3 的步骤。  

### transfer()方法
需要说明的一点是，虽然我们一直在说帮助扩容，其实更准确的说应该是帮助迁移元素。因为扩容的第一次初始化新表（扩容后的新表）这个动作，只能由一个线程完成。其他线程都是在帮助迁移元素到新数组。  
```java
A   A    A    A    B    B     A    A
0   1    2    3    4    5     6    7
```
为了方便，上边以原数组长度 8 为例。在元素迁移的时候，所有线程都遵循从后向前推进的规则，即如图A线程是第一个进来的线程，会从下标为7的位置，开始迁移数据。
而且当前线程迁移时会确定一个范围，限定它此次迁移的数据范围，如图 A 线程只能迁移 bound=6到 i=7 这两个数据。
此时，其它线程就不能迁移这部分数据了，只能继续向前推进，寻找其它可以迁移的数据范围，且每次推进的步长为固定值 stride（此处假设为2）。如图中 B线程发现 A 线程正在迁移6,7的数据，因此只能向前寻找，然后迁移 bound=4 到 i=5 的这两个数据。  
当每个线程迁移完成它的范围内数据时，都会继续向前推进。那什么时候是个头呢？  
这就需要维护一个全局的变量 transferIndex，来表示所有线程总共推进到的元素下标位置。如图，线程 A 第一次迁移成功后又向前推进，然后迁移2,3 的数据。此时，若没有其他线程在帮助迁移，则 transferIndex 即为2。  
剩余部分等待下一个线程来迁移，或者有任何的 A 和B线程已经迁移完成，也可以推进到这里帮助迁移。直到 transferIndex=0 。（会做一些其他校验来判断是否迁移全部完成，看代码）。  

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
 int n = tab.length, stride;
 //根据当前CPU核心数，确定每次推进的步长，最小值为16.（为了方便我们以2为例）
 if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
  stride = MIN_TRANSFER_STRIDE; // subdivide range
 //从 addCount 方法，只会有一个线程跳转到这里，初始化新数组
 if (nextTab == null) {            // initiating
  try {
   @SuppressWarnings("unchecked")
   //新数组长度为原数组的两倍
   Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
   nextTab = nt;
  } catch (Throwable ex) {      // try to cope with OOME
   sizeCtl = Integer.MAX_VALUE;
   return;
  }
  //用 nextTable 指代新数组
  nextTable = nextTab;
  //这里就把推进的下标值初始化为原数组长度（以16为例）
  transferIndex = n;
 }
 //新数组长度
 int nextn = nextTab.length;
 //创建一个标志类
 ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
 //是否向前推进的标志
 boolean advance = true;
 //是否所有线程都全部迁移完成的标志
 boolean finishing = false; // to ensure sweep before committing nextTab
 //i 代表当前线程正在迁移的桶的下标，bound代表它本次可以迁移的范围下限
 for (int i = 0, bound = 0;;) {
  Node<K,V> f; int fh;
  //需要向前推进
  while (advance) {
   int nextIndex, nextBound;
   //(1) 先看 (3) 。i每次自减 1，直到 bound。若超过bound范围，或者finishing标志为true，则不用向前推进。
   //若未全部完成迁移，且 i 并未走到 bound，则跳转到 (7)，处理当前桶的元素迁移。
   if (--i >= bound || finishing)
    advance = false;
   //(2) 每次执行，都会把 transferIndex 最新的值同步给 nextIndex
   //若 transferIndex小于等于0，则说明原数组中的每个桶位置，都有线程在处理迁移了，
   //于是，需要跳出while循环，并把 i设为 -1，以跳转到④判断在处理的线程是否已经全部完成。
   else if ((nextIndex = transferIndex) <= 0) {
    i = -1;
    advance = false;
   }
   //(3) 第一个线程会先走到这里，确定它的数据迁移范围。(2)处会更新 nextIndex为 transferIndex 的最新值
   //因此第一次 nextIndex=n=16，nextBound代表当次迁移的数据范围下限，减去步长即可，
   //所以，第一次时，nextIndex=16，nextBound=16-2=14。后续，每次都会间隔一个步长。
   else if (U.compareAndSwapInt
      (this, TRANSFERINDEX, nextIndex,
       nextBound = (nextIndex > stride ?
           nextIndex - stride : 0))) {
    //bound代表当次数据迁移下限
    bound = nextBound;
    //第一次的i为15，因为长度16的数组，最后一个元素的下标为15
    i = nextIndex - 1;
    //表明不需要向前推进，只有当把当前范围内的数据全部迁移完成后，才可以向前推进
    advance = false;
   }
  }
  //(4)
  if (i < 0 || i >= n || i + n >= nextn) {
   int sc;
   //若全部线程迁移完成
   if (finishing) {
    nextTable = null;
    //更新table为新表
    table = nextTab;
    //扩容阈值改为原来数组长度的 3/2 ，即新长度的 3/4，也就是新数组长度的0.75倍
    sizeCtl = (n << 1) - (n >>> 1);
    return;
   }
   //到这，说明当前线程已经完成了自己的所有迁移（无论参与了几次迁移），
   //则把 sc 减1，表明参与扩容的线程数减少 1。
   if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
    //在 addCount 方法最后，我们强调，迁移开始时，会设置 sc=(rs << RESIZE_STAMP_SHIFT) + 2
    //每当有一个线程参与迁移，sc 就会加 1，每当有一个线程完成迁移，sc 就会减 1。
    //因此，这里就是去校验当前 sc 是否和初始值是否相等。相等，则说明全部线程迁移完成。
    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
     return;
    //只有此处，才会把finishing 设置为true。
    finishing = advance = true;
    //这里非常有意思，会把 i 从 -1 修改为16，
    //目的就是，让 i 再从后向前扫描一遍数组，检查是否所有的桶都已被迁移完成，参看 (6)
    i = n; // recheck before commit
   }
  }
  //(5) 若i的位置元素为空，则说明当前桶的元素已经被迁移完成，就把头结点设置为fwd标志。
  else if ((f = tabAt(tab, i)) == null)
   advance = casTabAt(tab, i, null, fwd);
  //(6) 若当前桶的头结点是 ForwardingNode ，说明迁移完成，则向前推进 
  else if ((fh = f.hash) == MOVED)
   advance = true; // already processed
  //(7) 处理当前桶的数据迁移。
  else {
   synchronized (f) {  //给头结点加锁
    if (tabAt(tab, i) == f) {
     Node<K,V> ln, hn;
     //若hash值大于等于0，则说明是普通链表节点
     if (fh >= 0) {
      int runBit = fh & n;
      //这里是 1.7 的 CHM 的 rehash 方法和 1.8 HashMap的 resize 方法的结合体。
      //会分成两条链表，一条链表和原来的下标相同，另一条链表是原来的下标加数组长度的位置
      //然后找到 lastRun 节点，从它到尾结点整体迁移。
      //lastRun前边的节点则单个迁移，但是需要注意的是，这里是头插法。
      //另外还有一点和1.7不同，1.7 lastRun前边的节点是复制过去的，而这里是直接迁移的，没有复制操作。
      //所以，最后会有两条链表，一条链表从 lastRun到尾结点是正序的，而lastRun之前的元素是倒序的，
      //另外一条链表，从头结点开始就是倒叙的。看下图。
      Node<K,V> lastRun = f;
      for (Node<K,V> p = f.next; p != null; p = p.next) {
       int b = p.hash & n;
       if (b != runBit) {
        runBit = b;
        lastRun = p;
       }
      }
      if (runBit == 0) {
       ln = lastRun;
       hn = null;
      }
      else {
       hn = lastRun;
       ln = null;
      }
      for (Node<K,V> p = f; p != lastRun; p = p.next) {
       int ph = p.hash; K pk = p.key; V pv = p.val;
       if ((ph & n) == 0)
        ln = new Node<K,V>(ph, pk, pv, ln);
       else
        hn = new Node<K,V>(ph, pk, pv, hn);
      }
      setTabAt(nextTab, i, ln);
      setTabAt(nextTab, i + n, hn);
      setTabAt(tab, i, fwd);
      advance = true;
     }
     //树节点
     else if (f instanceof TreeBin) {
      TreeBin<K,V> t = (TreeBin<K,V>)f;
      TreeNode<K,V> lo = null, loTail = null;
      TreeNode<K,V> hi = null, hiTail = null;
      int lc = 0, hc = 0;
      for (Node<K,V> e = t.first; e != null; e = e.next) {
       int h = e.hash;
       TreeNode<K,V> p = new TreeNode<K,V>
        (h, e.key, e.val, null, null);
       if ((h & n) == 0) {
        if ((p.prev = loTail) == null)
         lo = p;
        else
         loTail.next = p;
        loTail = p;
        ++lc;
       }
       else {
        if ((p.prev = hiTail) == null)
         hi = p;
        else
         hiTail.next = p;
        hiTail = p;
        ++hc;
       }
      }
      ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
       (hc != 0) ? new TreeBin<K,V>(lo) : t;
      hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
       (lc != 0) ? new TreeBin<K,V>(hi) : t;
      setTabAt(nextTab, i, ln);
      setTabAt(nextTab, i + n, hn);
      setTabAt(tab, i, fwd);
      advance = true;
     }
    }
   }
  }
 }
}
```
- 流程步骤总结
###### **重要的思想：**  
1. ForwardingNode节点，这个类是一个标志，用来代表当前桶（数组中的某个下标位置）的元素已经全部迁移完成
2. 高低位迁移
3. 协助扩容，每个线程分配一定的区域进行扩容，默认大小是16
4. boolean advance = true; 是否向前推进的标志，当前线程是否已经处理完自己的区域
5. boolean finishing = false;  是否所有线程都全部迁移完成的标志

###### 流程总结
1. 通过计算 CPU 核心数和 Map 数组的长度得到每个线程（CPU）要帮助处理多少个桶，并且这里每个线程处理都是平均的。默认每个线程处理 16 个桶。因此，如果长度是 16 的时候，扩容的时候只会有一个线程扩容。
2. 初始化临时变量 nextTable。将其在原有基础上扩容两倍。
3. 通过CAS操作开始转移。根据一个 finishing 变量来判断，该变量为 true 表示扩容结束，否则继续扩容。  
3.1 进入一个 while 循环，分配数组中一个桶的区间给线程，默认是 16. 从大到小进行分配。当拿到分配值后，进行 i-- 递减。这个 i 就是数组下标。（其中有一个 bound 参数，这个参数指的是该线程此次可以处理的区间的最小下标，超过这个下标，就需要重新领取区间或者结束扩容，还有一个 advance 参数，该参数指的是是否继续递减转移下一个桶，如果为 true，表示可以继续向后推进，反之，说明还没有处理好当前桶，不能推进)。总体来说while循环中做了三件事情：  
（1） 确认当前线程扩容的范围  
（2）是否所有的任务都已经分配出去，所有线程都有了自己的扩容区间  
（3）--i操作，处理当前线程范围内的元素  
3.2 出 while 循环，进 if 判断，判断扩容是否结束，如果扩容结束，清空临时变量，更新 table 变量，更新扩容阈值。如果没完成，但是当前线程已经扩容结束，将 sizeCtl 减一，表示扩容的线程少一个了。如果减完这个数以后，sizeCtl 回归了初始状态（-2），表示没有线程再扩容了，该方法所有的线程扩容结束了。（这里主要是判断扩容任务是否结束，如果结束了就让线程退出该方法，并更新相关变量）。然后检查所有的桶，防止遗漏。  
3.3 若i的位置元素为空，则说明当前桶的元素已经被迁移完成，就把头结点设置为fwd标志。  
3.4 如果 i 对应的槽位不是空，且有了占位符，那么该线程跳过这个槽位，处理下一个槽位。  
3.5 如果以上都是不是，说明这个槽位有一个实际的值。开始同步处理这个桶。  
3.6 到这里，都还没有对桶内数据进行转移，只是计算了下标和处理区间，然后一些完成状态判断。同时，如果对应下标内没有数据或已经被占位了，就跳过了。  
4. 处理每个桶的行为都是同步的。防止 putVal 的时候向链表插入数据。  
4.1 如果这个桶是链表，那么就将这个链表根据 length 取于拆成两份，取于结果是 0 的放在新表的低位，取于结果是 1 放在新表的高位。  
4.2 如果这个桶是红黑数，那么也拆成 2 份，方式和链表的方式一样，然后，判断拆分过的树的节点数量，如果数量小于等于 6，改造成链表。反之，继续使用红黑树结构。  
4.3 到这里，就完成了一个桶从旧表转移到新表的过程。  


