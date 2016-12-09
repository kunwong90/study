1.默认初始大小为16，并且容量只能为2的指数次方

    /**
     * The default initial capacity - MUST be a power of two.
     */
    static final int DEFAULT_INITIAL_CAPACITY = 16;
   

2.默认负载因子为0.75f

    /**
     * The load factor used when none specified in constructor.
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;


3.当元素个数大于多少时，准备进行扩容。一般为容量*负载因子，当超过这个数值时，就认为要扩容了

    /**
     * The next size value at which to resize (capacity * load factor).
     * @serial
     */
    int threshold;

4.当我们指定了初始容量，怎么能保证容量始终是2的指数次方？

    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);

        // Find a power of 2 >= initialCapacity
        int capacity = 1;
        while (capacity < initialCapacity)
            capacity <<= 1;

        this.loadFactor = loadFactor;
        threshold = (int)(capacity * loadFactor);
        table = new Entry[capacity];
        init();
    }

可以看出，通过左移运算来保证始终是2的指数次方。

5.如何根据key获取value

    public V get(Object key) {
        if (key == null)
            return getForNullKey();
        int hash = hash(key.hashCode());
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k)))
                return e.value;
        }
        return null;
    }
    
首先判断key是否为null，如果为null，则执行下面一段代码
`
   private V getForNullKey() {
   
        for (Entry<K,V> e = table[0]; e != null; e = e.next) {
            if (e.key == null)
                return e.value;
        }
        return null;
    }
`
从table[0]上依次判断key是否==null，如果找到，则返回对于的value，否则返回null。

如果key不为null，则计算出key的hashCode的hash值，然后再和数组的长度-1做一个与运算，得出key在数组中的下标值，再依次和table[i]链上的对象进行比较，找到就返回对应的值，否则返回null。

Q：我们可以通过map.get(key)==null来判断key不存在吗？

A：不可以，因为map.get(key)==null有可能key对应的value值就是null。

6.如何判断key是否存在？

    public boolean containsKey(Object key) {
        return getEntry(key) != null;
    }

    final Entry<K,V> getEntry(Object key) {
        int hash = (key == null) ? 0 : hash(key.hashCode());
        for (Entry<K,V> e = table[indexFor(hash, table.length)];
             e != null;
             e = e.next) {
            Object k;
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))
                return e;
        }
        return null;
    }

根据key找到对应的数组下标，然后依次判断链上的key是否存在。

7.为什么map是无须的，且每次输出的顺序有可能也是会变的？
因为map元素在添加的时候，是根据hash值来确定下标的。

会变是因为不能保证下标的位置是始终不变的，比如数组在扩容的时候，旧的元素迁移到新的元素后，计算出的下标位置可能会发生变化。

8.map是如何扩容的？

	void addEntry(int hash, K key, V value, int bucketIndex) {
		Entry<K,V> e = table[bucketIndex];
        	table[bucketIndex] = new Entry<K,V>(hash, key, value, e);
        	if (size++ >= threshold)
            		resize(2 * table.length);
    	}
	
如果元素个数大于等于threshold，就会进行扩容，扩容后的大小是原来的两倍。

	void resize(int newCapacity) {
        	Entry[] oldTable = table;
        	int oldCapacity = oldTable.length;
        	if (oldCapacity == MAXIMUM_CAPACITY) {
            		threshold = Integer.MAX_VALUE;
            		return;
        	}

        	Entry[] newTable = new Entry[newCapacity];
        	transfer(newTable);
        	table = newTable;
        	threshold = (int)(newCapacity * loadFactor);
    	}
	void transfer(Entry[] newTable) {
        	Entry[] src = table;
        	int newCapacity = newTable.length;
        	for (int j = 0; j < src.length; j++) {
            	Entry<K,V> e = src[j];
            	if (e != null) {
                	src[j] = null;
                	do {
                    		Entry<K,V> next = e.next;
                    		int i = indexFor(e.hash, newCapacity);
                    		e.next = newTable[i];
                    		newTable[i] = e;
                    		e = next;
                	} while (e != null);
            	}
        	}
    	}

9.HashMap允许key和value为null。

为什么key为null？

在底层的实现中，单独对key为null做了处理

    public V put(K key, V value) {
        if (key == null)
            return putForNullKey(value);
        int hash = hash(key.hashCode());
        int i = indexFor(hash, table.length);
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
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
    
当key为null时，使用putForNullKey方法添加，我们看下这个方法

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
    
可以看到，对于key为null的时候，是直接放到数组的第0个位置，然后依次判断链上的key是否==null，如果等于，则添加新的值，并返回旧的值。如果不等于，则直接添加新值，并返回null。
