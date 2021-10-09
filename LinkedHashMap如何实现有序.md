#### LinkedHashMap如何实现有序

HashMap有一个问题，就是迭代HashMap的顺序并不是HashMap放置的顺序，也就是无序。HashMap的这一缺点往往会带来困扰，因为有些场景，我们期待一个有序的Map。

这个时候，LinkedHashMap就闪亮登场了，它虽然增加了时间和空间上的开销，但是通过维护一个运行于所有条目的双向链表，LinkedHashMap保证了元素迭代的顺序。


#### LinkedHashMap基本数据结构
LinkedHashMap继承了HashMap，可以认为是HashMap+LinkedList，即它既使用HashMap操作数据结构，又使用LinkedList维护插入元素的先后顺序。

```java
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V>
{
```
其实我们通过看源码发现LinkedHashMap并没有操作数据结构的方法，所以操作数据结构的动作还是在hashMap中。但是细心的同学会发现
LinkedHashMap.Entry继承了HashMap.Node，并多了两个属性，这两个属性组成了一个双向链表保证了插入元素的先后顺序

```java
static class Entry<K,V> extends HashMap.Node<K,V> {
        // before:前置Entry
        // after:后置Entry
        Entry<K,V> before, after;
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
```

#### 实现
注意LinkedHashMap重写了hashMap创建newNode的方法，也就是在put操作数据结构时实际使用的是LinkedHashMap中的newNode方法，这就是多态。

```java
Map<String,String> map = new LinkedHashMap();
map.put("test","JayChen");
```

```java
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
        LinkedHashMap.Entry<K,V> p =
            new LinkedHashMap.Entry<K,V>(hash, key, value, e);
        // 将Entry放进链表末尾    
        linkNodeLast(p);
        return p;
    }



// link at the end of list
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
        LinkedHashMap.Entry<K,V> last = tail;
        tail = p;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
    }    
```

下面这三个方法分别对应不同情况下LinkedHashMap双向链表顺序的调整实现

```java
// Callbacks to allow LinkedHashMap post-actions
// 所有的Entry按照访问的顺序排列
void afterNodeAccess(Node<K,V> p) { }
// 插入新的节点，是否需要删除旧的节点
void afterNodeInsertion(boolean evict) { }
// 删除节点时，将节点从双向链表中移除掉
void afterNodeRemoval(Node<K,V> p) { }
```

#### LinkedHashMap实现lru算法
利用LinkedHashMap实现URL算法主要是通过afterNodeAccess方法实现。
accessOrder为true时才会执行。

（1）false，所有的Entry按照插入的顺序排列

（2）true，所有的Entry按照访问的顺序排列


可以通过构造方法传入

```java
public LinkedHashMap(int initialCapacity,
                         float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);
        this.accessOrder = accessOrder;
    }
```

```java
 void afterNodeAccess(Node<K,V> e) { // move node to last
        LinkedHashMap.Entry<K,V> last;
        if (accessOrder && (last = tail) != e) {
            LinkedHashMap.Entry<K,V> p =
                (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
            p.after = null;
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
            ++modCount;
        }
    }
```

