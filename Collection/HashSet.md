#### HashSet
1. HashSet是一个没有重复元素的集合,内部有个HashMap变量, 基本操作都是由map进行实现
```
private transient HashMap<E,Object> map;

// Dummy value to associate with an Object in the backing Map
// HashSet中只需要用到key，PRESENT是向map中插入key-value对应的value
private static final Object PRESENT = new Object();
```
  
![image](https://images0.cnblogs.com/blog/497634/201401/280038034067137.jpg)

2. 如何保证元素不重复
```
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```
此处直接调用了hashMap的put方法，put方法中有对 **e.hash == hash && ((k = e.key) == key || key.equals(k))** 进行判断，如果相同则替换Value值，不同则新增键值对。
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
```

