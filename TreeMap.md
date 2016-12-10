我们知道，HashMap和HashTable都是无序的，那有没有可以实现有序的Map呢？答案是肯定的。

TreeMap和LinkedHashMap都是可以实现有序的。今天来看一下TreeMap是怎么实现有序的。

我们看下TreeMap是怎么说的：

	/**
	 * A Red-Black tree based {@link NavigableMap} implementation.
	 * The map is sorted according to the {@linkplain Comparable natural
	 * ordering} of its keys, or by a {@link Comparator} provided at map
	 * creation time, depending on which constructor is used.
	 */
 
 TreeMap是基于红黑树实现的，可以根据key的自然排序进行排序，或者在创建的时候通过构造器传入一个Comparator比较器，实现自定义排序。
 
	  public V put(K key, V value) {

		Entry<K,V> t = root;
		if (t == null) {
		    // TBD:
		    // 5045147: (coll) Adding null to an empty TreeSet should
		    // throw NullPointerException
		    //
		    // compare(key, key); // type check
		    root = new Entry<K,V>(key, value, null);
		    size = 1;
		    modCount++;
		    return null;
		}
		int cmp;
		Entry<K,V> parent;
		// split comparator and comparable paths
		Comparator<? super K> cpr = comparator;
		if (cpr != null) {
		    do {
			parent = t;
			cmp = cpr.compare(key, t.key);
			if (cmp < 0)
			    t = t.left;
			else if (cmp > 0)
			    t = t.right;
			else
			    return t.setValue(value);
		    } while (t != null);
		}
		else {
		    if (key == null)
			throw new NullPointerException();
		    Comparable<? super K> k = (Comparable<? super K>) key;
		    do {
			parent = t;
			cmp = k.compareTo(t.key);
			if (cmp < 0)
			    t = t.left;
			else if (cmp > 0)
			    t = t.right;
			else
			    return t.setValue(value);
		    } while (t != null);
		}
		Entry<K,V> e = new Entry<K,V>(key, value, parent);
		if (cmp < 0)
		    parent.left = e;
		else
		    parent.right = e;
		fixAfterInsertion(e);
		size++;
		modCount++;
		return null;
	    }
    
    从上面代码中可以看出，TreeMap也可以允许添加key为null的元素，但是就不允许添加其它元素了，如果添加了会发生空指针异常。但是空指针异常报的行数
    会不一样，如果TreeMap中第一个元素添加的key为null，第二个元素的key不为null，空指针发生在cmp = k.compareTo(t.key);这一行；如果TreeMap中第
    一个元素key不为null，第二个元素的key为null，空指针发生在
    
    	if (key == null)
		throw new NullPointerException();
		
这一行。
 
 
 
 
 
 
 
 
 
