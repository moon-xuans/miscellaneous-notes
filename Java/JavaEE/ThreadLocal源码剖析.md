# ThreadLocal源码剖析

## 1.举个例子

我们先看下`ThreadLocal`的使用示例:

```java
 public class ThreadLocalDemo {
    private List<String> messages = new ArrayList<>();

    public static final ThreadLocal<ThreadLocalDemo> holder = ThreadLocal.withInitial(ThreadLocalDemo::new);

    public static void add(String message) {
        holder.get().messages.add(message);
    }

    public static List<String> clear() {
        List<String> messages = holder.get().messages;
        holder.remove();

        System.out.println("size:" + holder.get().messages.size());
        return messages;
    }

    public static void main(String[] args) {
        ThreadLocalDemo.add("一枝花算不算浪漫");
        System.out.println(holder.get().messages);
        ThreadLocalDemo.clear();
        ThreadLocalDemo.clear();
    }
}
```

打印结果:

```
[一枝花算不算浪漫]
size:0
```



`ThreadLocal`对象可以提供线程局部变量，每个线程`Thread`拥有一份自己的**副本变量**，多个线程互不干扰。

## 2.ThreadLocal的数据结构

![下载](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/%E4%B8%8B%E8%BD%BD.png)

`Thread`类有一个类型为`ThreadLocal.ThreadLocalMap`的实例变量`threadLocals`，也就是说每个线程有一个自己的`ThreadLocalMap`。

`Thread `有自己的独立实现，可以简单地将它的`key`视作``ThreadLocal`，`value`为代码中放入的值(实际上`key`并不是`ThreadLocal`本身，而是它的一个**弱引用**)。

每个线程在往`ThreadLocal`里放值的时候，都会往自己的`ThreadLocalMap`里存，读也是以`ThreadLocal`作为引用，在自己的`map`里找对应的`key`,从而实现了**线程隔离**。

`ThreadLocalMap`有点类似`HashMap`的结构，只是`HashMap`是由**数组+链表**实现的，而`ThreadLocal`并没有**链表**结构。

我们还要注意`Entry`,它的`key`是`ThreadLocal<?> k`，继承自`WeakReference`，也就是我们常说的弱类型。

## 3.会出现的问题

### 3.1.弱引用和内存泄露

此时ThreadLocal的内存图(实线表示强引用，虚线表示弱引用)，如下:

![20210124144945153](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/20210124144945153.png)

假设在业务代码中使用完ThreadLocal，threadLocal Ref被回收了。

由于ThreadLocalMap只持有ThreadLocal的弱引用，没有任何强引用指向threadLocal实例，所以threadLocal就可以顺利被gc回收，此时**Entry中的key=null**。

但是在没有手动删除这个Entry以及CurrentThread依然运行的前提下，也存在强引用链threadRef -> currentThread -> threadLocalMap -> Entry -> value,value不会被回收，**而这块value永远不会被访问到了，导致value内存泄露**。

内存泄露的真正原因是什么呢？

有两个前提:

- 没有手动删除这个Entry
- CurrentThread依然运行

第一点很好理解，只要在使用完ThreadLocal，调用其remove方法删除对应的Entry，就能避免内存泄露。

第二点稍微复杂一点。由于ThreadLocalMap是Thread的一个属性，被当前线程所引用，所以它的生命周期跟Thread一样长，那么在使用完ThreadLocal之后，如果当前Thread也随之执行结束，ThreadLocalMap自然也会被gc回收，从根源上避免了内存泄露。

**综上，ThreadLocal内存泄露的根源是：由于ThreadLocalMap的生命周期跟Thread一样长，如果没有手动删除对应的key就会导致内存泄露**。

## 4.源码分析

### 4.1.ThreadLocal.set()方法

![6.ca0fd1f6](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/6.ca0fd1f6.png)

`ThreadLocal`中的`set`方法原理如上图所示，很简单，主要是判断`ThreadLocalMap`是否存在，然后使用`ThreadLocal`中的`set`方法进行数据处理。

代码如下：

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

主要的核心逻辑还是在`ThreadLocalMap`中的，一步步往下看，后面还有更详细的剖析。

### 4.2.`ThreadLocalMap`Hash算法

既然是`Map`结构，那么`ThreadLocalMap`当然也要实现自己的`hash`算法来解决散列表数组冲突问题。

```java
int i = key.threadLocalHashCode & (len-1);
```

`ThreadLocalMap`中`hash`算法很简单，这里`i`就是当前key在散列表中对应的数组下标位置。

这里最关键的就是`threadLocalHashCode`值的计算，`ThreadLocal`中有一个属性为`HASH_INCREMENT = 0x61c88647`

```java
public class ThreadLocal<T> {
    private final int threadLocalHashCode = nextHashCode();

    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    private static final int HASH_INCREMENT = 0x61c88647;

    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
    
      static class ThreadLocalMap {
		 ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
      }
}
```

每当创建一个`ThreadLocal`对象，这个`ThreadLocal.nextHashCode`这个值就会增长`0x61c88647`.

这个值很特殊，它是**斐波那契数**也叫**黄金分割数**。`hash`增量为这个数字，带来的好处就是`hash`分布非常均匀。

### 4.3..`ThreadLocalMap`Hash冲突

虽然`ThreadLocalMap`中使用了**黄金分割数**来作为`hash`计算因子，大大减少了`hash`冲突的概率，但是仍然会存在冲突。

ThreadLocalMap这里采用开放地址法进行解决哈希冲突。

![7.1f9d4116](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/7.1f9d4116.png)

如上图所示，我们插入一个`value = 27`的数据，通过`hash`计算后应该落入槽位4中，而槽位4已经有了`Entry`数据。

此时就会线性向后查找，一直找到`Entry`为`null`的槽位才会停止查找，将当前元素放入此槽位中。遇到其他情况，有相应处理，后面再说。

### 4.4.ThreadLocalMap.set()

往`ThreadLocalMap`中`set`数据(新增或更新数据)分为好几种情况。

**第一种情况**:通过`hash`计算后的槽位对应的`Entry`数据为空:

![9.c6310b8b](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/9.c6310b8b.png)

这里直接将数据放到该槽位即可。

**第二种情况**:槽位数据不为空，`key`值与当前`ThreadLocal`通过`hash`计算获取的`key`值一致。

![test](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/test.png)

这里直接更新该槽位的数据。

**第三种情况**：槽位数据不为空，往后遍历中，在找到`Entry`为`null`的槽位之前，没有遇到`key`过期的`Entry`:

![11.bc629184](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/11.bc629184.png)

遍历散列数组，线性往后查找，如果找到`Entry`为`null`的槽位，则将数据放入该槽位中，或者往后遍历过程中，遇到了**key值相等**的数据，直接更新即可。

**第四种情况**：槽位数据不为空，往后遍历过程中，在找到`Entry`为`null`的槽位之前，遇到`key`过期的`Entry`,如下图，往后遍历过程中，遇到了`index = 7`的槽位数据`Entry`的`key = null`;

![12.f2bf5c84](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/12.f2bf5c84.png)

散列数组下标为7位置对应的`Entry`数据`key`为`null`,表明此数据`key`值已经被垃圾回收掉了，此时就会执行`replaceStaleEntry()`方法，该方法的含义是**替换过期数据的逻辑**，以**index=7**位起点开始遍历，进行探测式清理工作。

初始化探测式清理过期数据扫描的开始位置:`slotToExpunge = staleSlot = 7`

以当前`staleSlot`开始向前迭代查找，找其他过期的数据，然后更新过期数据起始扫描下标`slotToExpunge`。`for`循环，直接碰到`Entry`为`null`结束。

如果找到了过期的数据，继续向前迭代，直到遇到`Entry=null`的槽位才停止迭代，如下图所示，**slotToExpunge被更新为0**：

![13.0a26fdb7](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/13.0a26fdb7.png)

上面向前迭代的操作是为了更新探测清理过期数据的起始下标`slotToExpunge`的值，这个值后面会讲，它是用来判断当前过期槽位`staleSlot`之前是否还有过期元素。

接着开始以`staleSlot`位置`(index=7)`向后迭代，**如果找到了相同key值的Entry数据**：

![14.86c91024](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/14.86c91024.png)

从当前节点`staleSlot`向后查找`key`相等的`Entry`元素，找到后更新`Entry`的值并交换`staleSlot`元素的位置(`staleSlot`位置为过期元素),更新`Entry`数据，然后开始进行过期`Entry`的清理工作，如下图所示：

![view](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/view.png)

向后遍历过程中，如果没有找到相同key值的Entry数据：

![15.ebdeb9f2](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/15.ebdeb9f2.png)

从当前节点`staleSlot`向后查找`key`相等的`Entry`元素，直到`Entry`为`null`则停止寻找。通过上图可知，此时`table`中没有`key`值相同的`Entry`.

创建新的`Entry`，替换`table[staleSlot]`位置:

![16.9d3d6eda](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/16.9d3d6eda.png)

替换完成后也是进行过期元素清理工作，清理工作只要是有两个方法:`expungeStaleEntry()`和`cleanSomeSlots`，具体细节后面讲。

上面已经用图解析了set实现的原理，下来看下源码。

```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);

    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;
            return;
        }

        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

这里会通过`key`来计算在散列表中的对应位置，然后以当前`key`对应的桶的位置向后查找，找到可以使用的桶。

```java
Entry[] tab = table;
int len = tab.length;
int i = key.threadLocalHashCode & (len-1);
```

什么情况下桶才是可以使用的呢？

1. `k = key`说明是替换操作，可以使用
2. 碰到一个过期的桶，执行替换逻辑，占用过期桶
3. 查找过程中，碰到桶中`Entry=null`的情况，直接使用。

接着就是执行`for`循环遍历，向后查找，我们看下`nextIndex()`、`prevIndex()`方法实现。

![test02](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/test02.png)

```java
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}

private static int prevIndex(int i, int len) {
    return ((i - 1 >= 0) ? i - 1 : len - 1);
}
```

接着看剩下`for`循环中的逻辑:

1. 遍历当前`key`值对应的桶中`Entry`数据为空，这说明散列数组这里没有数据冲突，跳出`for`循环，直接`set`数据到对应的桶中

2. 如果`key`值对应的桶中`Entry`数据不为空

   2. 1 如果`k = key`，说明当前`set`操作是一个替换操作，做替换逻辑，直接返回

   2. 2 如果`k = null`，说明当前桶位置的`Entry`是过期数据，执行`replaceStaleEntry`方法(核心方法)，然后返回

3. `for`循环执行完毕，继续往下执行说明向后迭代的过程中遇到了`entry`为`null`的情况

   3. 1 在`Entry`为`null`的桶中创建一个新的`Entry`对象

   3. 2 执行`++size`操作

4.  调用`cleanSomeSlots`做一次启发式清理工作，清理散列数组中`Entry`的`key`过期的数据

   4. 1 如果清理工作完成后，未清理到任何数据，且`size`超过了阈值(数组长度的2/3)，进行`rehash()`操作

   4. 2 `rehash()`中会先进行一轮探测式清理，清理过期`key`，清理完成后如果**size >=threshold - threshold/4**,就会执行真正的扩容逻辑。

接着重点看下`replaceStaleEntry()`方法，`replaceStaleEntry()`方法提供替换过期数据的功能，具体代码如下:

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                       int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))

        if (e.get() == null)
            slotToExpunge = i;

    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {

        ThreadLocal<?> k = e.get();

        if (k == key) {
            e.value = value;

            tab[i] = tab[staleSlot];
            tab[staleSlot] = e;

            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

`slotToExpunge`表示开始探测式清理过期数据的开始下标，默认从当前的`staleSlot`开始。以当前的`staleSlot`开始，向前迭代查找，找到没有过期的数据，`for`循环一直碰到`Entry `为`null`才会结束。如果向前找到了过期数据，更新探测清理过期数据的开始下标i，即`slotToExpunge = i`

```java
 for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))

        if (e.get() == null)
            slotToExpunge = i;
```

接着开始从`staleSlot`向后查找，也是碰到`Entry`为`null`的桶结束。如果迭代过程中，**碰到k == key**，这说明这里是替换逻辑，替换新数据并且交换当前`staleSlot`位置。如果`slotToExpunge == staleSlot`，这说明`replaceStaleEntry()`一开始向前查找过期数据时并未找到过期的`Entry`数据，接着向后查找过程中也未发现过期数据，修改开始探测式清理过期数据的下标为当前循环的index，即`slotToExpunge = i`。最后调用`cleanSomeSlots(expungeStaleEntry(slotToExpunge),len);`进行启发式过期数据清理。

```java
if (k == key) {
    e.value = value;

    tab[i] = tab[staleSlot];
    tab[staleSlot] = e;

    if (slotToExpunge == staleSlot)
        slotToExpunge = i;

    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
    return;
}
```

`cleanSomeSlots`和`expungeStaleEntry()`后面会细讲，一个是过期`key`相关的`Entry`的启发式清理`(Heuristically scan)`,另一个是过期`key`相关`Entry`的探测式清理。

**如果k != key**则会接着往下走，`k == null`说明当前遍历的`entry`是一个过期数据，`slotToExpunge == staleSlot`说明，一开始的向前查找数据并未找到过期的`Entry`。如果条件成立，则更新`slotToExpunge`为当前位置，这个前提是前驱节点扫描时未发现过期数据。

```java
if (k == null && slotToExpunge == staleSlot)
    slotToExpunge = i;
```

往后迭代的过程中如果没有找到`k == key`，且碰到`Entry`为`null`的数据，则结束当前的迭代操作。此时说明这里是一个添加的逻辑，将新的数据添加到`table[staleSlot]`对应的`slot`中。

```java
tab[staleSlot].value = null;
tab[staleSlot] = new Entry(key, value);
```

最后判断除了`staleSlot`以外，还发现其他过期的`slot`数据，就要开启清理数据的逻辑 ：

```java
if (slotToExpunge != staleSlot)
    cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
```

### 4.5.`ThreadLocalMap`过期key的探测式清理

我们先讲下探测式清理，也即是`expungeStaleEntry`方法，遍历散列数组，从开始位置向后探测清理过期数据，将过期数据的`Entry`设置为`null`，沿途中碰到未过期的数据将此数据`rehash`后重新在`table`数组中定位，如果定位的位置已经有了数组，则会将未过期的数据放到最靠近此位置的`Entry = null`的桶中，使`rehash`后的`Entry`数据距离正确的桶的位置更近一些。操作逻辑如下:

![18.67655263](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/18.67655263.png)

`set(27)`经过hash计算后应该落到`index=4`的桶中，由于`index=4`桶中已经有了数据，所以往后迭代最终数据放入到`index=7`的桶中，放入后一段时间后`index=5`中的`Entry`数据`key`变为了`null`。

![19.d97719c4](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/19.d97719c4.png)

如果再有其他数据`set`到`map`中，就会触发**探测式清理**操作。

如上图，执行**探测式清理**后，`index=5`的数据被清理后，继续往后迭代，到`index=7`的元素时，经过`rehash`后发现该元素正确的`index=4`,而此位置已经有了数据，往后查找离`index=4`最近的`Entry = null`的节点(刚被探测式清理掉的数据：`index=5`),找到后移动`index=7`的数据到`index=5`中，此时桶的位置离正确的位置`index=4`更近了。

经过一轮探测式清理后，`key`过期的数据会被清理掉，没过期的数据经过`rehash`重定位后所处的桶位置理论上更近`i = key.hashCode & (tab.len - 1)`的位置。这种优化会提高这个散列表的查询性能。

接着看下`expungeStaleEntry()`具体流程。

![20.2571d0fa](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/20.2571d0fa.png)

我们假设`expungeStaleEntry(3)`来调用此方法，如上图说是，我们可以看到`ThreadLocalMap`中`stale`的数据情况，接着执行清理操作：

![test6](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/test6.png)

第一步是清空当前`staleSlot`位置的数据，`index=3`位置的`Entry`变成了`null`。然后接着往后探测:

![22.04c31f1b](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/22.04c31f1b.png)

执行娃你第二步后，index=4的元素挪到了index=3的槽位中。

继续后迭代检查，碰到正常数据，计算该数据位置是否偏移，如果被偏移，则重新计算`slot`位置，目的是让正常数据尽可能存放在正确位置或离正确位置更近的位置

![test7](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/test7.png)

在往后迭代过程中碰到空的槽位，终止探测，这样一轮探测式清理工作就完成了，接着我们看下源码。

```java
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;

                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
```

这里我们还是以`staleSlot = 3`来做示例说明，首先是将`tab[staleSlot]`槽位的数组清空，然后设置`size--`接着以`slaleSlot`位置往后迭代，如果遇到`k == null`的过期数据，也是清空该槽位数据，然后`size--`

```java
ThreadLocal<?> k = e.get();

if (k == null) {
    e.value = null;
    tab[i] = null;
    size--;
}
```

如果`key`没有过期，重新计算当前`key`的下标位置是不是当前槽位下标位置，如果不是，那么说明产生了`hash`冲突，此时以新计算出来正确的槽位位置往后迭代，找到最近一个可以存放`entry`的位置。

```java
int h = k.threadLocalHashCode & (len - 1);
if (h != i) {
    tab[i] = null;

    while (tab[h] != null)
        h = nextIndex(h, len);

    tab[h] = e;
}
```

这里是处理正常的产生`hash`冲突的数据，经过迭代后，有过`hash`冲突数据的`Entry`位置会更靠近正确位置，这样的话，查询的时候效率才会更高。

### 4.6.`ThreadLocalMap`扩容机制

在`ThreadLocalMap.set()`方法的最后，如果执行完启发式清理工作后，未清理到任何数据，且当前散列数组中`Entry`的数量已经达到了列表的扩容阈值`(len * 2 / 3)`，就开始执行`rehash()`逻辑:

```java
if (!cleanSomeSlots(i, sz) && sz >= threshold)
    rehash();
```

接着看下`rehash()`具体实现：

```java
private void rehash() {
    expungeStaleEntries();

    if (size >= threshold - threshold / 4)
        resize();
}

private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null)
            expungeStaleEntry(j);
    }
}
```

这里首先是会进行探测式清理工作，从`table`的起始位置往后清理，上面有分析清理的详细流程。清理完成之后，`table`中可能有一些`key`为`null`的`Entry`数据被清理掉，所以此时通过判断`size >= thread - thread / 4`也就是`size >= threshold * 3 / 4`来决定是否扩容。

记得上面进行`rehash()`的阈值是`size >= threshold`,所以当面试官套路我们`ThreadLocalMap`扩容机制的时候，我们一定要说清楚这两个步骤:

![24.786f016e](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/24.786f016e.png)

接着看看具体的`resize()`方法，为了方便演示，我们以`oldTab.len  = 8`

![25.ea13b459](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/25.ea13b459.png)

扩容后的`tab`的大小为`oldLen * 2`,然后遍历老的散列表，重新计算`hash`位置，然后放到新的`tab`数组中，如果出现`hash`冲突则往后寻找最近的`entry`为`null`的槽位，遍历完成之后，`oldTab`中所有的`entry`数据都已经放入到新的`tab`中了。重新计算`tab`下次扩容的阈值，具体代码如下：

```java
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2;
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null;
            } else {
                int h = k.threadLocalHashCode & (newLen - 1);
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e;
                count++;
            }
        }
    }

    setThreshold(newLen);
    size = count;
    table = newTab;
}
```

### 4.7.ThreadLocalMap.get()

**第一种情况**:通过查找`key`值计算出散列表中`slot`位置，然后该`slot`位置中的`Entry.key`和查找的`key`一致，则会直接返回:

![26.c616aa9a](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/26.c616aa9a.png)

**第二种情况**:`slot`位置中的`Entry.key`和要查找的`key`不一致:

![27.d4002a85](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/27.d4002a85.png)

我们以`get(ThreadLocal)`为例，通过`hash`计算后，正确的`slot`位置应该是4，而`index=4`的槽位已经有了数据，且`key`值不等于`ThreadLocal1`，然后需要继续往后迭代查找。

迭代到`index = 5`的数据时，此时`Entry.key = null`，触发一次探测式数据回收操作，执行`expungeStaleEntry()`方法，执行完后，`index 5, 8`的数据都会被回收，而`index 6,7`的数据都会前移，此时继续往后迭代，到`index = 6`的时候即找到了`key`值相等的`Entry`数据，如下图所示:

![28.d0c48c3f](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/28.d0c48c3f.png)

下来，我们来看看源码。

```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

### 4.8.`ThreadLocalMap`过期key的启发式清理流程

探测式清理是以当前`Entry`往后清理，遇到值为`null`则结束清理，属于`线性探测清理`。

而启发式清理被作者定义为:**Heuristically scan some cells looking for table entrires**.(启发式地扫描，寻找过期的key)

![29.e327fe32](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/29.e327fe32.png)

具体代码如下：

```java
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);
    return removed;
}
```

