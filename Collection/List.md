参考资料：http://www.cnblogs.com/skywang12345/p/3323085.html
#### ArrayList
1. 数组实现
```
 public ArrayList(int initialCapacity) {
	super();
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
	this.elementData = new Object[initialCapacity];
    }
```

2. 默认10，扩容增加原先一半
```
public void ensureCapacity(int minCapacity) {
	modCount++;
	int oldCapacity = elementData.length;
	if (minCapacity > oldCapacity) {
	    Object oldData[] = elementData;
	    int newCapacity = (oldCapacity * 3)/2 + 1;
    	    if (newCapacity < minCapacity)
		newCapacity = minCapacity;
            // minCapacity is usually close to size, so this is a win:
            elementData = Arrays.copyOf(elementData, newCapacity);
	}
    }
```

3. modCount  
    **fail-fast**机制。该字段表示list结构上被修改的次数。结构上的修改指的是那些改变了list的长度大小或者使得遍历过程中产生不正确的结果的其它方式。
    该字段被Iterator以及ListIterator的实现类所使用，如果该值被意外更改，Iterator或者ListIterator将抛出ConcurrentModificationException异常，这是jdk在面对迭代遍历的时候为了避免不确定性而采取的快速失败原则。
    子类对此字段的使用是可选的，如果子类希望支持快速失败，只需要覆盖该字段相关的所有方法即可。单线程调用不能添加删除terator正在遍历的对象，否则将可能抛出ConcurrentModificationException异常，如果子类不希望支持快速失败，该字段可以直接忽略。【ConcurrentModificationException是在操作Iterator时抛出的异常】

```
// class AbstractList
protected transient int modCount = 0;
```
```
// class Itr implements Iterator<E>
final void checkForComodification() {
　　if(modCount != expectedModCount)//表示如果期待修改次数与实际修改次数不等 则抛ConcurrentModificationException异常
        throw new ConcurrentModificationException();
    }
```
解决办法：java.util.concurrent包的CopyOnWriteArrayList， 他的iterator方法不是AbstractList实现的， 没有校验modCount。

#### LinkedList
1. 双向链表 继承AbstractSequentialList 【环】
```
public LinkedList() {
    header.next = header.previous = header;
}
```
2. addBefore、remove举例 实际就是普通的链表操作
```
private Entry<E> addBefore(E e, Entry<E> entry) {
	Entry<E> newEntry = new Entry<E>(e, entry, entry.previous);
	newEntry.previous.next = newEntry;
	newEntry.next.previous = newEntry;
	size++;
	modCount++;
	return newEntry;
}

private E remove(Entry<E> e) {
	if (e == header)
	    throw new NoSuchElementException();

        E result = e.element;
	e.previous.next = e.next;
	e.next.previous = e.previous;
        e.next = e.previous = null;
        e.element = null;
	size--;
	modCount++;
        return result;
}
```
3. 查找,它就是通过一个计数索引值来实现的。例如，当我们调用get(int location)时，首先会比较“location”和“双向链表长度的1/2”；若前者大，则从链表头开始往后查找，直到location位置；否则，从链表末尾开始先前查找，直到location位置
```
public E get(int index) {
    return entry(index).element;
}
/**
 * Returns the indexed entry.
 */
private Entry<E> entry(int index) {
    if (index < 0 || index >= size)
        throw new IndexOutOfBoundsException("Index: "+index+
                                            ", Size: "+size);
    Entry<E> e = header;
    if (index < (size >> 1)) {
        for (int i = 0; i <= index; i++)
            e = e.next;
    } else {
        for (int i = size; i > index; i--)
            e = e.previous;
    }
    return e;
}
```
4. LinkedList可以作为FIFO(先进先出)的队列，作为FIFO的队列时，下表的方法等价：
```
队列方法       等效方法
add(e)        addLast(e)
offer(e)      offerLast(e)
remove()      removeFirst()
poll()        pollFirst()
element()     getFirst()
peek()        peekFirst()
```

5. LinkedList可以作为LIFO(后进先出)的栈，作为LIFO的栈时，下表的方法等价：
```
栈方法        等效方法
push(e)      addFirst(e)
pop()        removeFirst()
peek()       peekFirst()
```
#### Vector
线程安全，
1. 数组实现 矢量队列
```
public Vector(int initialCapacity, int capacityIncrement) {
	super();
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
	this.elementData = new Object[initialCapacity];
	this.capacityIncrement = capacityIncrement;
}
```
2. 默认10  当Vector容量不足以容纳全部元素时，Vector的容量会增加。若容量增加系数 >0，则将容量的值增加“容量增加系数”；否则，将容量大小增加一倍。
3. 线程安全 synchronized
```
public synchronized void addElement(E obj) {
	modCount++;
	ensureCapacityHelper(elementCount + 1);
	elementData[elementCount++] = obj;
}

```
4. Let gc do its work
刪除操作都存在这类操作
```
protected synchronized void removeRange(int fromIndex, int toIndex) {
	modCount++;
	int numMoved = elementCount - toIndex;
        System.arraycopy(elementData, toIndex, elementData, fromIndex,
                         numMoved);

	// Let gc do its work
	int newElementCount = elementCount - (toIndex-fromIndex);
	while (elementCount != newElementCount)
	    elementData[--elementCount] = null;
    }
```
#### Stact
1. Stack是栈。它的特性是：先进后出(FILO, First In Last Out)。 继承自Vector 同样是数组实现，线程安全
2. API
```
             boolean       empty()
synchronized E             peek()
synchronized E             pop()
             E             push(E object)
synchronized int           search(Object o)
```