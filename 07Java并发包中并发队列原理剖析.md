# 第7章 Java并发包中并发队列原理剖析

LinkedBlockingQueue和ArrayBlockingQueue比较简单，不进行讲解了。下面只介绍PriorityBlockingQueue和DelayQueue。

## PriorityBlockingQueue

PriorityBlockingQueue是带优先级的无界阻塞队列，每次出队都返回优先级最高或最低的元素。内部使用二叉堆实现。

### 类图结构

![](images/10.png)

PriorityBlockingQueue内部有一个数组queue，用来存放队列元素。allocationSpinLock是个自旋锁，通过CAS操作来保证同时只有一个线程可以扩容队列，状态为0或1。

由于这是一个优先队列，所以有一个comparator用来比较元素大小。

下面为构造函数：

```java
private static final int DEFAULT_INITIAL_CAPACITY = 11;

public PriorityBlockingQueue() {
    this(DEFAULT_INITIAL_CAPACITY, null);
}

public PriorityBlockingQueue(int initialCapacity) {
    this(initialCapacity, null);
}
```

可知默认队列容量为11，默认比较器为null，也就是使用元素的compareTo方法进行比较来确定元素的优先级，这意味着队列元素必须实现Comparable接口。

###　原理介绍

#### boolean offer()

```java
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    // 获取独占锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    int n, cap;
    Object[] array;
    // 扩容
    while ((n = size) >= (cap = (array = queue).length))
        tryGrow(array, cap);
    try {
        Comparator<? super E> cmp = comparator;
        if (cmp == null)
            // 通过对二叉堆的上浮操作保证最大或最小的元素总在根节点
            siftUpComparable(n, e, array);
        else
            // 使用了自定义比较器
            siftUpUsingComparator(n, e, array, cmp);
        size = n + 1;
        // 激活因调用take()方法被阻塞的线程
        notEmpty.signal();
    } finally {
        // 释放锁
        lock.unlock();
    }
    return true;
}
```

流程比较简单，下面主要看扩容和建堆操作。

先看扩容。

```java
private void tryGrow(Object[] array, int oldCap) {
    // 由前面的代码可知，调用tryGrow函数前先获取了独占锁，
    // 由于扩容比较费时，此处先释放锁，
    // 让其他线程可以继续操作（如果满足可操作的条件的话），
    // 以并发性能
    lock.unlock();
    Object[] newArray = null;
    // 通过allocationSpinLock保证同时最多只有一个线程进行扩容操作。
    if (allocationSpinLock == 0 &&
        UNSAFE.compareAndSwapInt(this, allocationSpinLockOffset,0, 1)) {
        try {
            // 当容量比较小时，一次只增加2容量
            // 比较大时增加一倍
            int newCap = oldCap + ((oldCap < 64) ?(oldCap + 2) : (oldCap >> 1));
            // 溢出检测
            if (newCap - MAX_ARRAY_SIZE > 0) {
                int minCap = oldCap + 1;
                if (minCap < 0 || minCap > MAX_ARRAY_SIZE)
                    throw new OutOfMemoryError();
                newCap = MAX_ARRAY_SIZE;
            }
            if (newCap > oldCap && queue == array)
                newArray = new Object[newCap];
        } finally {
            // 释放锁，没用CAS是因为同时最多有一个线程操作allocationSpinLock
            allocationSpinLock = 0;
        }
    }
    // 如果当前线程发现有其他线程正在对队列进行扩容，
    // 则调用yield方法尝试让出CPU资源促使扩容操作尽快完成
    if (newArray == null)
        Thread.yield();
    lock.lock();
    if (newArray != null && queue == array) {
        queue = newArray;
        System.arraycopy(array, 0, newArray, 0, oldCap);
    }
}
```

下面来看建堆算法

```java
private static <T> void siftUpComparable(int k, T x, Object[] array) {
    Comparable<? super T> key = (Comparable<? super T>) x;
    while (k > 0) {
        // 获取父节点，设子节点索引为k，
        // 则由二叉堆的性质可知，父节点的索引总为(k - 1) >>> 1
        int parent = (k - 1) >>> 1;
        // 获取父节点对应的值
        Object e = array[parent];
        // 只有子节点的值小于父节点的值时才上浮
        if (key.compareTo((T) e) >= 0)
            break;
        array[k] = e;
        k = parent;
    }
    array[k] = key;
}
```

如果了解二叉堆的话，此处代码是十分容易理解的。关于二叉堆，可参看[《数据结构之二叉堆》](https://blog.csdn.net/u010960184/article/details/82717074)。

#### E poll()

```java
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        // 出队
        return dequeue();
    } finally {
        lock.unlock();
    }
}

private E dequeue() {
    int n = size - 1;
    if (n < 0)
        return null;
    else {
        Object[] array = queue;
        E result = (E) array[0];
        // 获取尾节点，在实现对二叉堆的下沉操作时要用到
        E x = (E) array[n];
        array[n] = null;
        Comparator<? super E> cmp = comparator;
        if (cmp == null)
            // 下沉操作，保证取走最小的节点（根节点）后，新的根节点仍时最小的，二叉堆的性质依然满足
            siftDownComparable(0, x, array, n);
        else
            // 使用自定义比较器
            siftDownUsingComparator(0, x, array, n, cmp);
        size = n;
        return result;
    }
}
```

poll方法通过调用dequeue方法使最大或最小的节点出队并将其返回。

下面来看二叉堆的下沉操作。

```java
private static <T> void siftDownComparable(int k, T x, Object[] array, int n) {
    if (n > 0) {
        Comparable<? super T> key = (Comparable<? super T>)x;
        int half = n >>> 1;
        while (k < half) {
            // child为两个子节点（如果有的话）中较小的那个对应的索引
            int child = (k << 1) + 1;
            Object c = array[child];
            int right = child + 1;
            // 通过比较保证child对应的为较小值的索引
            if (right < n &&
                ((Comparable<? super T>) c).compareTo((T) array[right]) > 0)
                c = array[child = right];
            if (key.compareTo((T) c) <= 0)
                break;
            // 下沉，将较小的子节点换到父节点位置
            array[k] = c;
            k = child;
        }
        array[k] = key;
    }
}
```

同上，对下沉操作有疑问的话可参考上述文章。

#### void put(E e)

调用了offer

```java
public void put(E e){
    offer(e);
}
```

#### E take()

take操作的作用是获取二叉堆的根节点元素，如果队列为空则阻塞。

```java
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    // 阻塞可被中断
    lock.lockInterruptibly();
    E result;
    try {
        // 队列为空就将当前线程放入notEmpty条件队列
        // 使用while循环判断是为了避免虚假唤醒
        while ( (result = dequeue()) == null)
            notEmpty.await();
    } finally {
        lock.unlock();
    }
    return result;
}
```




