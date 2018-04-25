#### HashMap
1. 初始容量16、默认加载因子是 0.75
```
/**
 * The default initial capacity - MUST be a power of two.
 */
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

/**
 * The maximum capacity, used if a higher value is implicitly specified
 * by either of the constructors with arguments.
 * MUST be a power of two <= 1<<30.
 */
static final int MAXIMUM_CAPACITY = 1 << 30;

/**
 * The load factor used when none specified in constructor.
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;
```
2. **数据结构**  
HashMap是一个“链表散列”,底层实现还是数组，只是数组的每一项都是一条链【单向链表】。其中table[0]存“key为null”的元素，其他通过hash()计算hash值  
![数据结构](https://images0.cnblogs.com/blog/381060/201401/152128347367.png)
3. 解决冲突【拉链法、哈希桶】  
hashCode是使用Key通过hash()计算出来的，不同的Key可能会得到同样的hashCode。通过拉链法解决冲突，把hashCode相同的拉个链表。  
![拉链法](https://images2017.cnblogs.com/blog/1165325/201708/1165325-20170821134711230-1857962628.png)
4. 计算hash以及table索引
```
static int hash(int h) {
    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}

/**
 * Returns index for hash code h.
 * & 取模。除了取模运算外还有一个责任：均匀分布table数据和充分利用空间
 */
static int indexFor(int h, int length) {
    return h & (length-1);
}
```
5. 方法举例
- get
```
public V get(Object key) {
    if (key == null)
        return getForNullKey();
    int hash = hash(key.hashCode());
    for (Entry<K,V> e = table[indexFor(hash, table.length)];
         e != null;
         e = e.next) {
        Object k;
        // hash值相同并且key相等
        if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
            return e.value;
    }
    return null;
}

// key为null的 在table[0]里面取,Entry是个单向链表
private V getForNullKey() {
    for (Entry<K,V> e = table[0]; e != null; e = e.next) {
        if (e.key == null)
            return e.value;
    }
    return null;
}
```
- put
```
public V put(K key, V value) {
    if (key == null)
        return putForNullKey(value);
    int hash = hash(key.hashCode());
    int i = indexFor(hash, table.length);
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        // 如果桶中已经存在该key的元素 那么对其值进行替换。
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }

    modCount++;
    addEntry(hash, key, value, i);
    return null;
}

// key为null的 在放到table[0]里面
private V putForNullKey(V value) {
    for (Entry<K,V> e = table[0]; e != null; e = e.next) {
        if (e.key == null) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }
    modCount++;
    addEntry(0, null, value, 0);
    return null;
}
```
- remove
```
public V remove(Object key) {
    Entry<K,V> e = removeEntryForKey(key);
    return (e == null ? null : e.value);
}

/**
 * Removes and returns the entry associated with the specified key
 * in the HashMap.  Returns null if the HashMap contains no mapping
 * for this key.
 */
final Entry<K,V> removeEntryForKey(Object key) {
    // table[0] or [i]
    int hash = (key == null) ? 0 : hash(key.hashCode());
    int i = indexFor(hash, table.length);
    Entry<K,V> prev = table[i];
    Entry<K,V> e = prev;

    // 删除链表中的节点
    while (e != null) {
        Entry<K,V> next = e.next;
        Object k;
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k)))) {
            modCount++;
            size--;
            if (prev == e)
                table[i] = next;
            else
                prev.next = next;
            e.recordRemoval(this);
            return e;
        }
        prev = e;
        e = next;
    }

    return e;
}
```