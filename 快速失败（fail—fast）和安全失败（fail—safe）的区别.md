### 1.概述
Fail-safe和Fail-fast，是多线程并发操作集合时的一种失败处理机制。

### 2.快速失败（fail—fast）
在用 **迭代器遍历一个集合对象时** ，如果遍历过程中对集合对象的内容进行了修改（增加、删除、修改），则会抛出Concurrent Modification Exception。  

```java
Map<String,String> map = new HashMap<>();
        map.put("1","a");
        map.put("2","b");
        map.put("3","c");

Iterator<String> iterator = map.keySet().iterator();

while (iterator.hasNext()) {
    System.out.println(map.get(iterator.next()));
    map.put("4","d");
}

结果：
a
Exception in thread "main" java.util.ConcurrentModificationException
	at java.util.HashMap$HashIterator.nextNode(HashMap.java:1442)
	at java.util.HashMap$KeyIterator.next(HashMap.java:1466)
	at com.example.demo.config.NacosMain.main(NacosMain.java:32)
```
- 原理
迭代器在遍历时直接访问集合中的内容，并且在遍历过程中使用一个 modCount 变量。集合在被遍历期间如果内容发生变化，就会改变modCount的值。每当迭代器使用hashNext()/next()遍历下一个元素之前，都会检测modCount变量是否为expectedmodCount值，是的话就返回遍历；否则抛出异常，终止遍历。  
这里异常的抛出条件是检测到 modCount！=expectedmodCount 这个条件。如果集合发生变化时修改modCount值刚好又设置为了expectedmodCount值，则异常不会抛出。  

- 场景
java.util包下的集合类都是快速失败的，不能在多线程下发生并发修改（迭代过程中被修改）算是一种安全机制吧。

### 3.安全失败（fail—safe）
表示失败安全，也就是在这种机制下，出现集合元素的修改，不会抛出ConcurrentModificationException。

- 原理
原因是采用安全失败机制的集合容器，在遍历时不是直接在集合内容上访问的，而是先复制原有集合内容，在拷贝的集合上进行遍历。由于迭代时是对原集合的拷贝进行遍历，所以在遍历过程中对原集合所作的修改并不能被迭代器检测到。

```java
CopyOnWriteArrayList<String> copyOnWriteArrayList = new CopyOnWriteArrayList<>();
copyOnWriteArrayList.add("a");
copyOnWriteArrayList.add("b");
copyOnWriteArrayList.add("c");

Iterator<String> iterator1 = copyOnWriteArrayList.iterator();
while (iterator1.hasNext()){
    System.out.println(iterator1.next());
    copyOnWriteArrayList.add("d");
}

- 结果
a
b
c
```
- 场景
java.util.concurrent包下的容器都是安全失败的,可以在多线程下并发使用,并发修改。常见的的使用Fail-safe方式遍历的容器有ConcerrentHashMap和CopyOnWriteArrayList等。
