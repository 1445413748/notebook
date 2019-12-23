## 概述

`LinkedHashMap` 继承自 `HashMap` ，在 `HashMap` 中访问元素是无序的 ，而 `LinkedHashMap` 可以控制访问元素的访问顺序。

`LinkedHashMap` 内部维护有一个双向链表，将节点连接在一起，用于支持顺序访问，大概结构如下：

![](img/linkedHashmap.png)



## 初步使用

```java
public class LinkedHashMapTest {

    public static void main(String[] args) {
        LinkedHashMap<Integer, String> linkedHashMap = new LinkedHashMap<>();
        linkedHashMap.put(1, "val1");
        linkedHashMap.put(2, "val2");
        linkedHashMap.put(3, "val3");
        linkedHashMap.put(4, "val4");
        Iterator<Map.Entry<Integer, String>> iterator = linkedHashMap.entrySet().iterator();
        while (iterator.hasNext())
            System.out.println(iterator.next());
        
        
    }
}

// 结果
1=val1
2=val2
3=val3
4=val4
```

可以看到输出是有序的。



## 源码实现



### 相关属性

```java
// 双向链表节点，继承自HashMap.Node，主要添加了头节点和尾节点
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}

// 双向链表的头节点
transient LinkedHashMap.Entry<K,V> head;
// 双向链表的尾节点
transient LinkedHashMap.Entry<K,V> tail;

// 控制遍历的模式
// true：按照访问顺序
// false：按照插入顺序。（默认）
final boolean accessOrder;
```



### 主要构造函数

```java
public LinkedHashMap(int initialCapacity,
                     float loadFactor,
                     boolean accessOrder) {
    // 调用了 HashMap 构造函数
    super(initialCapacity, loadFactor);
    // 设置遍历模式
    this.accessOrder = accessOrder;
}
```



### 增删改查

#### 增删改

查看 `LinkedHashMap`，发现其并没有重写增删改的函数，而是直接使用 `HashMap` 的方法，其维护二维链表，主要是重写了 `HashMap`  提供的几个钩子函数

```java
// Callbacks to allow LinkedHashMap post-actions
// 访问节点后的操作
void afterNodeAccess(Node<K,V> p) { }
// 插入节点后的操作
void afterNodeInsertion(boolean evict) { }
// 删除节点后的操作
void afterNodeRemoval(Node<K,V> p) { }
```

三个函数的主要实现

```java
// 使用 remove 后应在双向链表中删除
void afterNodeRemoval(Node<K,V> e) { // unlink
    // 得到e的前驱节点和后继节点
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    // 断开连接
    p.before = p.after = null;
    // 前驱节点为null，说明删除的是头节点
    if (b == null)accessOrder
        head = a;
    else
        b.after = a;
    // 后继节点为null，说明删除的是后继节点
    if (a == null)
        tail = b;
    else
        a.before = b;
}

// 插入节点后
// evict 为false表示处于创建模式，即初始化
// 我们初始化后调用put时 evict总为true
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    // 根据removeEldestEntry判断是否删除最老的节点，也就是链表的头节点
    // removeEldestEntry为protecta方法，默认返回false
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}

// 访问节点后
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    // 如果开启了按照访问顺序，即accessOrder=true且当前节点不是最后的节点，则将当前节点移动到最后
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;s
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        // 修改 modCount
        ++modCount;
    }
}
```



**注意到 `afterNodeAccess` 会修改 `modCount`，所以如果我们在 `accessOrder = true` 模式下进行迭代的同时查询数据，会导致 `fail-fast`。**



#### 查

`LinkedHashMap` 重写了 `get` 和 `getOrDefault`

```java
public V get(Object key) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return null;
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}

/**
     * {@inheritDoc}
     */
public V getOrDefault(Object key, V defaultValue) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return defaultValue;
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}
```

实质上还是调用了 `HashMap` 的查询函数，查询后根据 `accessOrder` 判断是否需要更新链表，之所以 `LinkedHashMap` 不直接使用 `HashMap` 的 `get` 和 `getOrDefault` 主要是因为 `accessOrder` 是 `LinkedHashMap` 的属性， `HashMap` 的无法访问。



#### 利用 LinkedHashMap实现LRU

```java
class LRU<K,V> extends LinkedHashMap<K,V>{
    private final int size;

    public LRU(int size){
        super((int)Math.ceil(size/0.75f), 0.75f, true);
        this.size = size;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > size;
    }
}
```

