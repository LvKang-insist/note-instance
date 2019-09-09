java 的并发集合有哪些，和同步集合有哪些区别：
- ConcurrentHashMap
- CopyOnWriteArrayList
- CopyOnWriteArraySet
- 

1. ConcurrentHashMap和HashTable的区别

他们都可以在多线程下执行，但是当HashTable的内容变多时他的性能就会降低，因为迭代它必须锁定更多的时间。而ConcurrentHashMap是专用于高并发的Map实现，内部使用了锁分离，即分段锁。他会把整个map划分为几个片段，然后对每个片段进行上锁。同时允许多线性访问其他未上锁的片段。
2. ConcurrentHashMap和Collections.synchronizedMap之间的区别

Collections.synchronizedMap和HashTable 一样，现上在调用map所有方法时，都对整个map进行同步，而ConcurrentHashMap的实现却更加精细，它对map中的所有桶加了锁。所以，只要要有一个线程访问map，其他线程就无法进入map，而如果一个线程在访问ConcurrentHashMap某个桶时，其他线程，仍然可以对map执行某些操作。


---

JAVA 并发集合

- List：

CopyOnWriteArrayList：

add：
```
    private transient volatile Object[] array;
    
    public boolean add(E e) {
    //拿到锁对象
        final ReentrantLock lock = this.lock;
        //获得锁
        lock.lock();
        try {
            Object[] elements = getArray();
            int len = elements.length;
            Object[] newElements = Arrays.copyOf(elements, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            lock.unlock();
        }
    }
    final Object[] getArray() {
        return array;
    }
     final void setArray(Object[] a) {
        array = a;
    }
```

首先是 拿到锁对象，然后获得锁，通过getArray方法拿到的数组，在拿到数组的长度，然后复制一个新的数组，长度为原来数组+1，然后将要保存的数据存入到数组，最后调用setArray方法使用复制的数组将原来的数组覆盖。

get：
```
    public E get(int index) {
        return get(getArray(), index);
    }
    
     @SuppressWarnings("unchecked")
    private E get(Object[] a, int index) {
        return (E) a[index];
    }
```
首先使用getArray拿到数组，在调用get方法返回数组中指定的下标元素。

Vector和CopyOnWriteArrayList是两个线程安全的List，Vector读写操作都用了同步，相对来说更适用于写多读少的场合，CopyOnWriteArrayList在写的时候会复制一个副本，对副本写，写完用副本替换原值，读的时候不需要同步，适用于写少读多的场合。

- Set：

CopyOnWriteArraySet：

add：
```
    private final CopyOnWriteArrayList<E> al;
    //构造器
    public CopyOnWriteArraySet() {
        al = new CopyOnWriteArrayList<E>();
    }
    
    public boolean add(E e) {
        return al.addIfAbsent(e);
    }
    
    //CopyOnWriteArrayList类中的方法
    
    public boolean addIfAbsent(E e) {
        //拿到数组
        Object[] snapshot = getArray();
        //indexOf:判读有没有重复元素，大于0则说明有重复元素，返回false
        //addIfAbsent:
        return indexOf(e, snapshot, 0, snapshot.length) >= 0 ? false :
            addIfAbsent(e, snapshot);
    }
    //判断有没有票重复元素
    private static int indexOf(Object o, Object[] elements,
                               int index, int fence) {
        //如果传入值为null                       
        if (o == null) {
        //遍历数组
            for (int i = index; i < fence; i++)
            //有等于null的返回下标
                if (elements[i] == null)
                    return i;
        } else {
        //遍历数组
            for (int i = index; i < fence; i++)
            //有没有和添加的元素相同的
            //相同返回下标
                if (o.equals(elements[i]))
                    return i;
        }
        //返回-1
        return -1;
    }
    private boolean addIfAbsent(E e, Object[] snapshot) {
        final ReentrantLock lock = this.lock;
        //获取锁
        lock.lock();
        try {
            //拿到数组
            Object[] current = getArray();
            int len = current.length;
            //判断 addIfAbsent方法中获取的数组和这里获取的数组是否一样，如果不一样则说明在遍历的过程中有别的线程将新的元素添加进来了，则需要进一步的处理
            if (snapshot != current) {
                // Optimize for lost race to another addXXX operation
                //拿到最小数
                int common = Math.min(snapshot.length, len);
                //根据最小数遍历数组
                for (int i = 0; i < common; i++)
                    //如果两个元素不相同，且要添加的元素和数组中的这个元素相同 则返回false，添加失败（因为已经重复了）
                    if (current[i] != snapshot[i] && eq(e, current[i]))
                        return false;
                //调用indexOf遍历数组，判读是否由相同的元素
                if (indexOf(e, current, common, len) >= 0)
                        return false;
            }
            //复制一个新数组，长度为原来的+1，然后添加新元素，最后覆盖原来的数组
            Object[] newElements = Arrays.copyOf(current, len + 1);
            newElements[len] = e;
            setArray(newElements);
            return true;
        } finally {
            //释放锁
            lock.unlock();
        }
    }
```

其实底层还是用的CopyOnWriteArrayList类中的方法。首先获取array，然后遍历array，如果没有和要添加元素重复的则进行添加，否则有重复，添加失败，返回false。
在添加新元素的时候 又获取了一次array，然后和刚开始获取的array进行比较（因为可能在刚开始遍历的时候 别的线程已经将新的元素添加进数组了,注意添加的时候是copy了一个新数组进行添加，然后传给array，所以在这里可以进行比较，如果别的线程没有添加的话 两次获取的array引用都相同，比较的时候不成立，直接添加即可）。如果相同，则复制新数组,长度+1，添加新元素，覆盖原来的数组。如果不相同，说明在开始遍历的时候 有新元素添加到数组里面了。则进行处理：拿到两个数组的最小长度，for循环遍历，如果两个数组的同一下标下的元素不相同，则说明有新元素了，然后和要添加的进行判断，相同则 元素重复，添加失败，返回false。等for循环完成后，则遍历有新元素的那个数组，没有重复则进行添加，有重复则添加失败。返回false。

get：
CopyOnWriteArraySet没有get方法，只能通过迭代器来实现遍历。

```
 ·public Iterator<E> iterator() {
        return al.iterator();
    }
    
    CopyOnWriteArrayList类中的方法
    public Iterator<E> iterator() {
        return new COWIterator<E>(getArray(), 0);
    }

    //CopyOnWriteArrayList中的内部类
    static final class COWIterator<E> implements ListIterator<E> {
        //列出比较重要的......
 
        /** Snapshot of the array */
        private final Object[] snapshot;
        /** Index of element to be returned by subsequent call to next.  */
        private int cursor;

        private COWIterator(Object[] elements, int initialCursor) {
            cursor = initialCursor;
            snapshot = elements;
        }

        //判断数组是否遍历完成
        public boolean hasNext() {
            return cursor < snapshot.length;
        }
       
        //返回 数组中的元素
        @SuppressWarnings("unchecked")
        public E next() {
            if (! hasNext())
                throw new NoSuchElementException();
            return (E) snapshot[cursor++];
        }

        public int nextIndex() {
            return cursor;
        }

        public int previousIndex() {
            return cursor-1;
        }
    }
```

> 如有错误 ，还请指出，谢谢
 
