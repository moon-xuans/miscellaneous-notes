# 1.HashMap源码剖析

## 1.1.HashMap的底层实现

### 1.1.1.JDK1.8之前

JDK1.8之前`HashMap`底层是**数组和链表**结合在一起使用也就是**链表散列**。**HashMap通过key的hashCode经过扰动函数处理过后得到hash值，然后通过(n - 1) & hash判断当前元素存放的位置(这里的n指的是数组的长度)，如果当前位置存放元素的话，就判断该元素要与存入的元素的hash值以及key是否相同，如果相同的话，直接覆盖，不相同就通过拉链法解决冲突**。

> 所谓扰动函数指的就是HashMap的hash方法。使用hash方法也就是扰动函数是为了防止一些实现比较差的hashCode()方法，换句话说使用扰动函数之后可以减少碰撞。



> 所谓“拉链法”就是：将链表和数组相结合。也就是说创建一个链表数组，数组中每一格就是一个链表。若遇到哈希冲突，则将冲突的值加到链表中即可。

JDK1.7的HashMap的hash方法源码。

```java
static int hash(int h) {
    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).

    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```

JDK1.7的hash方法的性能会稍差一点点，因为毕竟扰动了4次。

### 1.1.2.JDK1.8之后

相比于之前的版本，JDK1.8之后在解决哈希冲突时有了较大的变化，当链表长度大于阈值(默认为8)(将链表转换为红黑树前会判断，如果当前数组的长度小于64，那么会选择先进行数组扩容，而不是转换为红黑树)时，将链表转化为红黑树，以减少搜索时间。

> TreeMap、TreeSet以及JDk1.8之后的HashMap底层都用到了红黑树。红黑树就是为了解决二叉查找树的缺陷，因为二叉查找树在某些情况下会退化成一个线性结构。

可以看看JDK1.8的hash方法，相比于JDK1.7方法更加简化，但是原理不变。

```java
 static final int hash(Object key) {
      int h;
      // key.hashCode()：返回散列值也就是hashcode
      // ^ ：按位异或
      // >>>:无符号右移，忽略符号位，空位都以0补齐
      return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
  }
```

### 1.1.3.HashMap的长度为什么是2的幂次方

为了能让HashMap存取高效，尽量减少碰撞，也就是要尽量把数据分配均匀。Hash值的范围值-2147483648到2147483647，前后加起来大概40亿的映射空间，只要哈希函数映射得比较均匀松散，一般应用是很难出现碰撞的。但问题是一个 40 亿长度的数组，内存是放不下的。所以这个散列值是不能直接拿来用的。用之前还要先做对数组的长度取模运算，得到的余数才能用来要存放的位置也就是对应的数组下标。这个数组下标的计算方法是“ `(n - 1) & hash`”。（n 代表数组长度）。

**这个算法为什么要这样设计呢？**

我们首先可能会想到采用%取余的操作来实现。但是，重点来了：**“取余(%)操作中如果除数是 2 的幂次则等价于与其除数减一的与(&)操作（也就是说 hash%length==hash&(length-1)的前提是 length 是 2 的 n 次方；）。”** 并且 **采用二进制位操作 &，相对于%能够提高运算效率，这就解释了 HashMap 的长度为什么是 2 的幂次方。**

### 1.1.4.为什么链表长度达到8之后才开始转换为红黑树？

源码上说，为了配合使用分布良好的hashCode，树节点很少使用。并且在理想状态下，受随机分布的hashCode影响，链表中的节点遵循泊松分布，而且根据统计，链表中节点数是8的概率已经接近千分之一，而且此时链表的性能已经很差了。所以在这种比较罕见和极端的情况下，才会把链表转变为红黑树。因为链表转换为红黑树也是需要消耗性能的，特殊情况特殊处理，为了挽回性能，权衡之下，才使用红黑树，提高性能。也就是大部分情况下，hashmap还是使用的链表，如果是理想的均匀分布，节点数不到8，hashmap就自动扩容了。为什么这么说呢，看图。

![image-20220112150424139](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220112150424139.png)

当数组长度小于MIN_TREEIFY_CAPACITY,就会扩容，而不是直接转变为红黑树。

为啥用8？因为通常情况下，链表长度很难达到8，但是特殊情况下链表长度为8，哈希表容量又很大，造成链表性能很差的时候，只能采用红黑树提高性能，这是一种应对策略。

## 1.2.分析源码

### 1.2.1.属性



```java
	//hash表初始化的大小为16
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    //hash表的最大容量
    static final int MAXIMUM_CAPACITY = 1 << 30;

    //默认的加载因子为0.75
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    //链表转红黑树的结点数为8
    static final int TREEIFY_THRESHOLD = 8;

    //红黑树转链表的结点数为6（剪枝）
    static final int UNTREEIFY_THRESHOLD = 6;

    //最小的树化hash表大小为64
    static final int MIN_TREEIFY_CAPACITY = 64;
```

### 1.2.2 构造器

```java
 
	public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }

    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

    public HashMap(Map<? extends K, ? extends V> m) {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        putMapEntries(m, false);
    }
```

### 1.2.3.方法

#### 1.2.3.1.put()

```java
public class HashMap_ {
    public static void main(String[] args) {
        HashMap hashMap = new HashMap();
        hashMap.put("java",10);
        hashMap.put("php", 10);
        hashMap.put("java", 20);
    }
}

public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }

public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }

 public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

 static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0) // 判断hash表是否为空，或者长度为0
            n = (tab = resize()).length;  // 进行扩容操作
        if ((p = tab[i = (n - 1) & hash]) == null) // 判断索引位置是否为null
            tab[i] = newNode(hash, key, value, null); // 为空的话，直接在上面创建一个结点
        else { // 否则不为空
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k)))) // 判断索引位置上的元素与put元素hash值是否相等，并且判断key是否相等，或者使用key的equals方法进行比较
                e = p; // 相等的话，记录索引位置元素
            else if (p instanceof TreeNode) // 判断此时的p结点是否是树形结点
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value); //是的话，以树节点的形式进行put操作
            else { // 还会从索引位置上，遍历后面的链表结点
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) { // 如果结点的下一个位置为空的话
                        p.next = newNode(hash, key, value, null); // 则将put元素放在结点的下一个位置
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st // 并判断是否需要树化，大于8-1，就进行树化
                            treeifyBin(tab, hash); 
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k)))) // 判断put元素与所比较元素是否相等
                        break;
                    p = e;
                }
            }
            if (e != null) { //当链表中存在这个相同的结点时
                V oldValue = e.value;//记录这个结点的值
                if (!onlyIfAbsent || oldValue == null)//如果旧value为null的话
                    e.value = value;//将这个即将加入的结点的value赋给此节点的value
                afterNodeAccess(e);
                return oldValue;//返回老结点的value，即当put进去同一个key的键值对的话，做替换工作，返回旧的value
            }
        }
        ++modCount;
        if (++size > threshold) // 判断此时达到阈值
            resize(); // 进行扩容
        afterNodeInsertion(evict);
        return null;
    }
```

(注：size指的是元素的个数，而不是数组中已有元素的个数)

#### 1.2.3.2.resize()

##### 1.2.3.2.1.JDK1.7的扩容中的操作

可以看看jdk1.7的转移数组的方法。

```java
    void transfer(Entry[] newTable) {
        Entry[] src = table;                   //src引用了旧的Entry数组
        int newCapacity = newTable.length;
        for (int j = 0; j < src.length; j++) { //遍历旧的Entry数组
            Entry<K, V> e = src[j];             //取得旧Entry数组的每个元素
            if (e != null) {
                src[j] = null;//释放旧Entry数组的对象引用（for循环后，旧的Entry数组不再引用任何对象）
                do {
                    Entry<K, V> next = e.next;
                    int i = indexFor(e.hash, newCapacity); //！！重新计算每个元素在数组中的位置
                    e.next = newTable[i]; //标记[1]
                    newTable[i] = e;      //将元素放在数组上
                    e = next;             //访问下一个Entry链上的元素
                } while (e != null);
            }
        }
    }
```

```java
    static int indexFor(int h, int length) {
        return h & (length - 1);
    }
```



newTable[i]的引用赋给了e.next，也就是使用了单链表的头插入方式，同一位置上新元素总会被放在链表的头部位置；这样先放在一个索引上的元素终会被放到Entry链的尾部（如果发生了hash冲突的话）。

**头插法是如何实现的，这块比较复杂**

要是在插入新数组的时候，也出现了一个数组下标的位置处，出现了多个节点的话，那又是怎么插入的呢？
1，假设现在刚刚插入到新数组上，因为是对象数组，数组都是要默认有初始值的，那么这个数组的初始值都是null。
那么e.next = newTable[i],也就是e.next = null啦。然后再newTable[i] = e;也就是 说这个时候，这个数组的这个下标位置的值设置成这个e啦。
2，假设这个时候，继续上面的循环，又取第二个数据e2的时候，恰好他的下标和刚刚上面的那个下标相同啦，那么这个时候，是又要有链表产生啦、
e.next = newTable[i];，假设上面第一次存的叫e1吧，那么现在e.next = newTable[i];也就是e.next = e1；
然后再，newTable[i] = e;，把这个后来的赋值在数组下标为i的位置，当然他们两个的位置是相同的啦。然后注意现在的e，我们叫e2吧。e2.next指向的是刚刚的e1，e1的next是null。
这就解释啦：先放在一个索引上的元素终会被放到Entry链的尾部。这句话。

##### 1.2.3.2.2.JDK1.8中的优化点

```java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table; // 保存原有的数组
        int oldCap = (oldTab == null) ? 0 : oldTab.length; // 得到原有数组的容量
        int oldThr = threshold; // 得到原有的阈值
        int newCap, newThr = 0; 
        if (oldCap > 0) { // 判断原有的容量是否大于0
            if (oldCap >= MAXIMUM_CAPACITY) { // 判断是否大于最大容量
                threshold = Integer.MAX_VALUE; // 阈值赋为integer最大值
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY) // 新容量为原容量的2倍
                newThr = oldThr << 1; // double threshold // 新阈值为原阈值的2倍
        }
        else if (oldThr > 0) // initial capacity was placed in threshold // 原阈值是否大于0
            newCap = oldThr; // 新容量为原阈值
        else {               // zero initial threshold signifies using defaults // 使用默认值
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap]; // 创建一个新数组
        table = newTab; // 直接将新数组赋给原数组
        if (oldTab != null) { 
            for (int j = 0; j < oldCap; ++j) { // 遍历原数组
                Node<K,V> e;
                if ((e = oldTab[j]) != null) { 
                    oldTab[j] = null; // 将原数组的元素指向null，便于gc回收
                    if (e.next == null) // 如果下个元素为null
                        newTab[e.hash & (newCap - 1)] = e; // 重新计算索引位置
                    else if (e instanceof TreeNode) // 如果这个元素是树节点
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap); // 就要进行修剪
                    else { // preserve order // 遍历每个bucket里的链表，每个链表可能会被拆分成两个链表，不需要移动的元素置入loHead为首的链表，需要移动的元素置入hiHead为首的链表，然后分别分配给老的bucket和新的bucket。
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
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

需要注意的是:在jdk1.7的时候，使用头插法进行移动元素，如果在新表的数组索引位置相同，则链表元素会倒置，但是jdk1.8不会倒置。它是使用尾插法，避免了在1.7时扩容出现死循环的情况。



由于我们使用的是2次幂的扩展(指长度扩为原来2倍)，所以，

**经过rehash之后，元素的位置要么是在原位置，要么是在原位置再移动2次幂的位置**。

看下图可以明白这句话的意思，n为table的长度，图（a）表示扩容前的key1和key2两种key确定索引位置的示例，图（b）表示扩容后key1和key2两种key确定索引位置的示例，其中hash1是key1对应的哈希值(也就是根据key1算出来的hashcode值)与高位与运算的结果。

![image-20220112145550518](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220112145550518.png)

元素在重新计算hash之后，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，因此新的index就会发生这样的变化：

![image-20220112145606668](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220112145606668.png)

因此，我们在扩充HashMap的时候，不需要像JDK1.7的实现那样重新计算hash，只需要看看原来的hash值新增的那个bit是1还是0就好了，是0的话索引没变，是1的话索引变成“原索引+oldCap”，可以看看下图为16扩充为32的resize示意图：

![image-20220112145631399](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220112145631399.png)

这个设计确实非常的巧妙，既省去了重新计算hash值的时间，而且同时，由于新增的1bit是0还是1可以认为是随机的，因此resize的过程，均匀的把之前的冲突的节点分散到新的bucket了。这一块就是JDK1.8新增的优化点。