# ConcurrentHashMap源码(jdk1.8)
## 基本介绍
      
     数据结构 : 同HashMap一样 , 也是采用的数组 + 链表 + 红黑树的结构
     并发控制 : 通过CAS + synchronized + Unsafe类直接操作地址的方式
     并发粒度 : 定位到了桶上而不是节点上 . 即一个链表或者红黑树结构同时只支持一个线程操作 .
     对比jdk1.7 : 采用ReentrantLock(Segment)即分段锁来实现并发控制 , 但是最多支持16个并发线程
## 数据结构
1.8中放弃了一个HashMap被一个Segment封装加上锁的复杂设计，  
取而代之的是在HashMap的每个Node上增加CAS + Synchronized来保证并发安全进行实现，  
结构如下：   

   ![](images/17842ac4.png)
## UML类图
![](images/871b5ea7.png)
## 主要参数
>重要成员变量

    table：默认为null，初始化发生在第一次插入操作，默认大小为16的数组，用来存储Node节点数据，扩容时大小总是2的幂次方。  
    nextTable：默认为null，扩容时新生成的数组，其大小为原数组的两倍。  
    sizeCtl ：默认为0，用来控制table的初始化和扩容操作，具体应用在后续会体现出来。    
              -1 :代表table正在初始化,其他线程应该交出CPU时间片;   
              -N: 表示正有N-1个线程执行扩容操作（高 16 位是 length 生成的标识符，低 16 位是扩容的线程数）  
              >0: 如果table已经初始化,代表table容量,默认为table大小的0.75,如果还未初始化,代表需要初始化的大小  
    其余情况：  
    1、如果table未初始化，表示table需要初始化的大小。  
    2、如果table初始化完成，表示table的容量，默认是table大小的0.75倍，居然用这个公式算0.75（n - (n >>> 2)）。  
    Node：保存key，value及key的hash值的数据结构。  
        其中value和next都用volatile修饰，保证并发的可见性。  

>代码段

        /**
         * 默认最大容量
         */
        private static final int MAXIMUM_CAPACITY = 1 << 30;
    
        /**
         * 默认初始化数组容量=16 . 即默认有16个bucket
         */
        private static final int DEFAULT_CAPACITY = 16;
    
        /**
         * 最大数组长度 8 = '_length='的长度
         */
        static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
    
        /**
         * 默认的并发级别:16 . 即最多允许16个线程并发 ..
         * jdk1.7中采用的是分段锁并发访问处理 . 此方式有很大的局限性
         * jdk1.8中放弃了锁分段的做法 , 采用了CAS+synchronized方式处理并发 .
         */
        private static final int DEFAULT_CONCURRENCY_LEVEL = 16;
    
        /**
         * 扩容时的触发条件 . 即负载因子. 当达到当前容量 * 0.75时 , 进行扩容
         */
        private static final float LOAD_FACTOR = 0.75f;
    
        /**
         * 树形化阈值.当链表长度达到8时 , 转换为树形结构
         */
        static final int TREEIFY_THRESHOLD = 8;
    
        /**
         * 当树结构中的节点数量达到6时 , 红黑树结构转换为链表结构
         */
        static final int UNTREEIFY_THRESHOLD = 6;
    
        /**
         * 最小树形化需要的数组长度 . 达不到该值时 , 只进行扩容操作
         */
        static final int MIN_TREEIFY_CAPACITY = 64;
    
        /**m
         * table进行扩容时 , 每个线程最小迁移的table的槽位数 .
         * 扩容时 , 在最大容量范围内 , 容量都是以2倍的方式进行 . 初始时容量16 , 则最小迁移槽位是16
         */
        private static final int MIN_TRANSFER_STRIDE = 16;
    
        /**
         * 作为偏移量参与和线程数量的标记数
         */
        private static int RESIZE_STAMP_BITS = 16;
    
        /**
         * 允许执行扩容的最大线程数
         */
        private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
    
        /**
         * 扩容阈值计算时的偏移量-default = 16
         */
        private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;
    
        /*
         * Encodings for Node hash fields. See above for explanation.
         */
        static final int MOVED = -1; // hash for forwarding nodes
        static final int TREEBIN = -2; // hash for roots of trees
        static final int RESERVED = -3; // hash for transient reservations
        static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash
    
        /**
         * 当前CPU的数量
         */
        static final int NCPU = Runtime.getRuntime().availableProcessors();
    
        /**
         * 执行序列化时要包含的对象域
         */
        private static final ObjectStreamField[] serialPersistentFields = {
                //分段锁对象初始化 . 兼容JDK1.7中的JDK实现
                new ObjectStreamField("segments", Segment[].class),
                new ObjectStreamField("segmentMask", Integer.TYPE),
                new ObjectStreamField("segmentShift", Integer.TYPE)
        };
    
        /* ---------------- Nodes -------------- */
    
        /**
         * 节点链表对象封装 ,特殊处理 :key-value均不能为null ,且不提供手动set节点value的方法[大概是为了防止并发问题]
         *
         * @param <K>
         * @param <V>
         */
        static class Node<K, V> implements Entry<K, V> {
            final int hash;
            final K key;
            volatile V val;
            volatile Node<K, V> next;
    
            Node(int hash, K key, V val, Node<K, V> next) {
                this.hash = hash;
                this.key = key;
                this.val = val;
                this.next = next;
            }
    
            @Override
            public final K getKey() {
                return key;
            }
    
            @Override
            public final V getValue() {
                return val;
            }
    
            @Override
            public final int hashCode() {
                return key.hashCode() ^ val.hashCode();
            }
    
            @Override
            public final String toString() {
                return key + "=" + val;
            }
    
            //不支持直接手动操作节点内容
            @Override
            public final V setValue(V value) {
                throw new UnsupportedOperationException();
            }
    
            /**
             * 对象地址相同或者对象内容相同
             * @param o
             * @return
             */
            public final boolean equals(Object o) {
                Object k, v, u;
                Entry<?, ?> e;
                return ((o instanceof Map.Entry) &&
                        (k = (e = (Entry<?, ?>) o).getKey()) != null &&
                        (v = e.getValue()) != null &&
                        (k == key || k.equals(key)) &&
                        (v == (u = val) || v.equals(u)));
            }
    
            /**
             * Map中的get(key)方法,获取key=k && key.hashCode = h 的节点
             */
            Node<K, V> find(int h, Object k) {
                Node<K, V> e = this;
                if (k != null) {
                    do {
                        K ek;
                        if (e.hash == h &&
                                ((ek = e.key) == k || (ek != null && k.equals(ek))))
                            return e;
                    } while ((e = e.next) != null);
                }
                return null;
            }
        }
    
        /* ---------------- Static utilities -------------- */
    
    
        /**
         * 位运算 . 通过移位降低Hash碰撞的风险
         *
         * @param h
         * @return
         */
        private static final int spread(int h) {
            return (h ^ (h >>> 16)) & HASH_BITS;
        }
    
        /**
         * 通过位运算将传入的数据换算为2的N次方的数值
         */
        private static final int tableSizeFor(int c) {
            int n = c - 1;
            n |= n >>> 1;
            n |= n >>> 2;
            n |= n >>> 4;
            n |= n >>> 8;
            n |= n >>> 16;
            return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
        }
    
        /**
         * 判断对象是否实现compare接口 ,
         * 若实现 : 返回对象类型 .
         * 否则 : 返回null
         * @param x
         * @return
         */
        static Class<?> comparableClassFor(Object x) {
            if (x instanceof Comparable) {
                Class<?> c;
                Type[] ts, as;
                Type t;
                ParameterizedType p;
                if ((c = x.getClass()) == String.class)
                    //若为字符串 . 返回string
                    return c;
                if ((ts = c.getGenericInterfaces()) != null) {
                    //循环类x实现的所有接口
                    for (int i = 0; i < ts.length; ++i) {
                        if (((t = ts[i]) instanceof ParameterizedType) &&
                                ((p = (ParameterizedType) t).getRawType() ==
                                        Comparable.class) &&
                                (as = p.getActualTypeArguments()) != null &&
                                as.length == 1 && as[0] == c) // type arg is c
                            return c;
                    }
                }
            }
            return null;
        }
    
        /**
         * 如果x与kc（k）类型匹配 ，则返回 k.compareTo（x）对比值，若x为空或者类型不匹配 . 直接返回0
         */
        @SuppressWarnings({"rawtypes", "unchecked"}) // for cast to Comparable
        static int compareComparables(Class<?> kc, Object k, Object x) {
            return (x == null || x.getClass() != kc ? 0 :
                    ((Comparable) k).compareTo(x));
        }
    
        /* ---------------- Table element access -------------- */
    
        /**
         * 寻找指定数组在内存中i位置的内容
         * getObjectVolatile(): 获取obj对象中offset偏移地址对应的tab的field值,支持volatile load语义
         * ABASE : 起始位置
         *
         * @param tab 指定的数组
         * @param i   位置
         * @param <K>
         * @param <V>
         * @return
         */
        @SuppressWarnings("unchecked")
        static final <K, V> Node<K, V> tabAt(Node<K, V>[] tab, int i) {
            return (Node<K, V>) U.getObjectVolatile(tab, ((long) i << ASHIFT) + ABASE);
        }
    
        /**
         * 比较并替换
         *
         * @param tab 包含要替换对象的数组
         * @param i   字段在对象内的下标,用于计算对象地址偏移量
         * @param c   期望的值
         * @param v   若当前值等于期望的值,进行替换 .用于进行更新替换的值
         * @param <K>
         * @param <V>
         * @return
         */
        static final <K, V> boolean casTabAt(Node<K, V>[] tab, int i,
                                             Node<K, V> c, Node<K, V> v) {
            //native 方法 . 采用C++实现
            // 通过我们传入的字段在对象中的偏移量,来获取到字段的地址(包括首地址+偏移量)
            // 通过对比两个字段的地址是否相同.若相同,则使用v的地址进行替换
            return U.compareAndSwapObject(tab, ((long) i << ASHIFT) + ABASE, c, v);
        }
    
        /**
         * 设置数组中某个下标处的值
         *
         * @param tab 数组对象
         * @param i   字段在对象内的下标,用于计算对象地址偏移量
         * @param v   要设置的值
         * @param <K>
         * @param <V>
         */
        static final <K, V> void setTabAt(Node<K, V>[] tab, int i, Node<K, V> v) {
            U.putObjectVolatile(tab, ((long) i << ASHIFT) + ABASE, v);
        }
    
        /* ---------------- Fields -------------- */
    
        /**
         * 在第一次插入数据时才会延迟初始化
         * 大小总是二的幂次方。由迭代器直接访问。
         * 为线程共享变量
         */
        transient volatile Node<K, V>[] table;
    
        /**
         * 下一个要使用的数组,只有在扩容时才不为空
         * 线程共享变量
         */
        private transient volatile Node<K, V>[] nextTable;
    
        /**
         * 基本计数器,主要在没有争用时使用,但在数组初始化竞争期间用作比对回退参考值.更新通过CAS-共享变量
         */
        private transient volatile long baseCount;
    
        /**
         * 表初始化和调整控件大小。如果为负值，则表正在初始化或调整大小：-1用于初始化，若为>= 1 : 活动调整大小线程的数量
         * 否则，当table为null时，将保留创建时使用的初始表大小，默认值为0。初始化后，保存下一个要调整表大小的元素计数值。-共享变量
         */
        private transient volatile int sizeCtl;
    
        /**
         * 扩容时要分割的下一个表的索引-共享变量
         */
        private transient volatile int transferIndex;
    
        /**
         * Spinlock (locked via CAS) used when resizing and/or creating CounterCells.
         * 扩容或者创建计数器单元格时使用的自旋锁(通过CAS锁定)-共享变量,用来标记当前数组是否在初始化或者扩容
         */
        private transient volatile int cellsBusy;
    
        /**
         * 计数器数组。
         * 非空时，大小为2的幂次方。
         * 并发情况下,采用分段并发,每个值记录每个段的数据的长度.
         * 计算总量时,将数组的值累加求和即为当前集合的总节点数
         */
        private transient volatile CounterCell[] counterCells;
    
        // views
        private transient KeySetView<K, V> keySet;
        private transient ValuesView<K, V> values;
        private transient EntrySetView<K, V> entrySet;
## 构造函数
>   说明：构造函数复制创建，初始化还是put()方法里面，和hashMap一样

        /**
        * Creates a new, empty map with the default initial table size (16).
        */
        //构造器,创建一个空的map对象 , 默认带长度[16]的数组
       public ConcurrentHashMap() {
       }
   
       //构造器, 创建一个新的空map,可指定其初始化的数组的大小 . 而不需动态调增
       public ConcurrentHashMap(int initialCapacity) {
           if (initialCapacity < 0)
               throw new IllegalArgumentException();
           int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
                      MAXIMUM_CAPACITY :
                      tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
           this.sizeCtl = cap;
       }
   
      //根据入参的map集合类型,创建一个相同的map集合 , 并初始化数据
       public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
           this.sizeCtl = DEFAULT_CAPACITY;
           putAll(m);
       }
   
        /**
           * 构造器
           * @param initialCapacity 初始化容量
           * @param loadFactor      加载因子
           */
       public ConcurrentHashMap(int initialCapacity, float loadFactor) {
           this(initialCapacity, loadFactor, 1);
       }
   
       
    /**
     * 构造器
     * @param initialCapacity  初始化容量
     * @param loadFactor       加载因子
     * @param concurrencyLevel 并发级别--default = 16
     */
       public ConcurrentHashMap(int initialCapacity,
                                float loadFactor, int concurrencyLevel) {
           if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
               throw new IllegalArgumentException();
           if (initialCapacity < concurrencyLevel)   // Use at least as many bins
               initialCapacity = concurrencyLevel;   // as estimated threads
           long size = (long)(1.0 + (long)initialCapacity / loadFactor);
           int cap = (size >= (long)MAXIMUM_CAPACITY) ?
               MAXIMUM_CAPACITY : tableSizeFor((int)size);
           this.sizeCtl = cap;
       }

## get方法
      /**
         * Returns the value to which the specified key is mapped,
         * or {@code null} if this map contains no mapping for the key.
         *
         * <p>More formally, if this map contains a mapping from a key
         * {@code k} to a value {@code v} such that {@code key.equals(k)},
         * then this method returns {@code v}; otherwise it returns
         * {@code null}.  (There can be at most one such mapping.)
         *
         * @throws NullPointerException if the specified key is null
         */
         //get方法 . 若key为空 . 抛空指针异常
        public V get(Object key) {
            Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
            //位运算，得到key的hash值
            int h = spread(key.hashCode());
            //若当前table节点数组不为空,且节点值不为null
            if ((tab = table) != null && (n = tab.length) > 0 &&
                (e = tabAt(tab, (n - 1) & h)) != null) {
                if ((eh = e.hash) == h) {
                //两种条件判断 : 1.入参key与数组当前偏移量处的对象地址相同 . 2.对象内容相同,则认为当前节点即为目标节点
                    if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                        return e.val;
                }
                //若目标节点的hash值<0,则从当前节点处,向下寻找,直到满足key的地址相同或者对象内容相同
                else if (eh < 0)
                    return (p = e.find(h, key)) != null ? p.val : null;
                //hash值相同 , 但不满足对象地址相同或者内容相同的情况下 , 沿着数组当前节点向下[链表或者红黑树]寻找 . 直到满足条件
                while ((e = e.next) != null) {
                    if (e.hash == h &&
                        ((ek = e.key) == key || (ek != null && key.equals(ek))))
                        return e.val;
                }
            }
            return null;
        }
    
## put方法
    /**
         * Maps the specified key to the specified value in this table.
         * Neither the key nor the value can be null.
         *
         * <p>The value can be retrieved by calling the {@code get} method
         * with a key that is equal to the original key.
         *
         * @param key key with which the specified value is to be associated
         * @param value value to be associated with the specified key
         * @return the previous value associated with {@code key}, or
         *         {@code null} if there was no mapping for {@code key}
         * @throws NullPointerException if the specified key or value is null
         */
        public V put(K key, V value) {
        
            //调用putVal
            return putVal(key, value, false);
        }
    
        /** Implementation for put and putIfAbsent */
        final V putVal(K key, V value, boolean onlyIfAbsent) {
            //如果key和value为null，报空指针
            if (key == null || value == null) throw new NullPointerException();
            //计算下表、偏移量-类似于计算hashcode.[将结果值控制在Integer.max范围内]
            int hash = spread(key.hashCode());
            int binCount = 0;
             //死循环处理,直到插入成功
            for (Node<K,V>[] tab = table;;) {
                Node<K,V> f; int n, i, fh;
                //若当前数组table为空 . 则进行初始化
                if (tab == null || (n = tab.length) == 0)
                 //初始化node数组
                    tab = initTable();
                else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
                    //i = (n - 1) & hash 等同于 i=hash%n(前提是n等于2的幂次方)
                    //如果table[i]==null(没有发生碰撞 即该位置的节点为空),
                    //直接利用CAS操作将该value存储在该位置,若CAS成功,直接退出死循环
                    if (casTabAt(tab, i, null,
                                 new Node<K,V>(hash, key, value, null)))
                        break;                   // no lock when adding to empty bin 当添加到空的bin时不上锁 
                }
                //若节点位置不为空 , 说明map中该位置已存在节点
                //若table[i]节点的hash值为-1 , 说明当前map正在扩容 , 这时当前线程会帮助执行扩容以加快速度并返回扩容后的新的数组
                else if ((fh = f.hash) == MOVED)
                    tab = helpTransfer(tab, f);
                else {
                   //table[i]的节点的hash值不等于MOVED[-1] ,说明当前不是处于扩容状态
                    V oldVal = null;
                     //锁住节点f[在上面的处理中说明逻辑执行到这里时节点f不为null],在加锁期间不允许其他线程操作
                     //synchronized (f)也实现了线程安全,确保该节点不会被其他线程操作处理 .
                      // 但也只是锁住该节点.该数组下其他的数据不受影响,也支持并发处理
                    synchronized (f) {
                        if (tabAt(tab, i) == f) {
                            //循环插入到链表中
                            if (fh >= 0) {
                                binCount = 1;
                                for (Node<K,V> e = f;; ++binCount) {
                                    K ek;
                                    //查询链表中是否出现了同样的key,若出现则直接更新value,并跳出循环
                                    if (e.hash == hash &&
                                        ((ek = e.key) == key ||
                                         (ek != null && key.equals(ek)))) {
                                        oldVal = e.val;
                                        if (!onlyIfAbsent)
                                            e.val = value;
                                        break;
                                    }
                                     //若查询链表没有同样的key , 则直接将数据插入到链表尾部 , 并跳出循环
                                    Node<K,V> pred = e;
                                    if ((e = e.next) == null) {
                                        pred.next = new Node<K,V>(hash, key,
                                                                  value, null);
                                        break;
                                    }
                                }
                            }
                            else if (f instanceof TreeBin) {
                            //如果table[i]为树节点，将节点node转成TreeBin类型 , 则将此节点插入树中即可
                                Node<K,V> p;
                                binCount = 2;
                                if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                               value)) != null) {
                                    oldVal = p.val;
                                    if (!onlyIfAbsent)
                                        p.val = value;
                                }
                            }
                        }
                    }
                    if (binCount != 0) {
                    //链表结构中新增节点时,变量binCount才会依次新增,红黑树结构中添加节点时,binCount = 2;
                   //binCount >= TREEIFY_THRESHOLD[8]: 判断是否达到树形化阈值条件 . 将链表结构转变为红黑树
                   //binCount = 2;表示当前已经是树形化结构了
                        if (binCount >= TREEIFY_THRESHOLD)
                            treeifyBin(tab, i);
                        if (oldVal != null)
                            return oldVal;
                        break;
                    }
                }
            }
             //节点计数器+1
            addCount(1L, binCount);
            return null;
        }
## initTable
      /**
           * 使用sizeCtl中记录的值初始化数组
           */
          private final Node<K, V>[] initTable() {
              Node<K, V>[] tab;
              int sc;
              // 执行条件 , 当前数组table为空或者长度为0
              while ((tab = table) == null || tab.length == 0) {
                  //若sizeCtl[偏移量-用于计算字段在数组中的地址]小于0.执行Thread.yield():使当前线程由执行状态，变成为就绪状态，让出cpu时间，
                  // 在下一个线程执行时候，此线程有可能被执行，也有可能没有被执行。
                  if ((sc = sizeCtl) < 0)
                      Thread.yield();
                      //将偏移量设置为-1,表明当前正在执行初始化 . 后续线程将被挂起进入就绪状态.
                  else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                      //初始化数组长度,并替换当前sizeCtl的值用于后续扩容
                      try {
                          if ((tab = table) == null || tab.length == 0) {
                              int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                              @SuppressWarnings("unchecked")
                              Node<K, V>[] nt = (Node<K, V>[]) new Node<?, ?>[n];
                              table = tab = nt;
                              sc = n - (n >>> 2);
                          }
                      } finally {
                          sizeCtl = sc;
                      }
                      break;
                  }
              }
              return tab;
          }
## addCount 方法
    /**
         * 从 putVal 传入的参数是 1， binCount，binCount 默认是0，
         *只有 hash 冲突了才会大于1.且他的大小是链表的长度（如果不是红黑数结构的话）12
         * @param x     the count to add
         * @param check if <0, don't check resize, if <= 1 only check if uncontended
         */
        private final void addCount(long x, int check) {
            CounterCell[] as;
            long b, s;
            //若当前计数器数组不为空
            if ((as = counterCells) != null ||
                    !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
                CounterCell a;
                long v;
                int m;
                boolean uncontended = true;
                if (as == null || (m = as.length - 1) < 0 ||
                        (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                        !(uncontended =
                                U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                    fullAddCount(x, uncontended);
                    return;
                }
                if (check <= 1)
                    return;
                s = sumCount();
            }
            if (check >= 0) {
                Node<K, V>[] tab, nt;
                int n, sc;
                while (s >= (long) (sc = sizeCtl) && (tab = table) != null &&
                        (n = tab.length) < MAXIMUM_CAPACITY) {
                    int rs = resizeStamp(n);
                    if (sc < 0) {
                        if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                                sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                                transferIndex <= 0)
                            break;
                        if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                            transfer(tab, nt);
                    } else if (U.compareAndSwapInt(this, SIZECTL, sc,
                            (rs << RESIZE_STAMP_SHIFT) + 2))
                        transfer(tab, null);
                    s = sumCount();
                }
            }
        }
## tryPresize方法
    /**
         * 数组扩容
         */
        private final void tryPresize(int size) {
            //size + (size >>> 1) + 1 : 8 + (8/2) + 1
            //tableSizeFor : 15->16 . 7->8 . 8->8
            int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
                    tableSizeFor(size + (size >>> 1) + 1);
            int sc;
            while ((sc = sizeCtl) >= 0) {
                Node<K, V>[] tab = table;
                int n;
                //若当前table为空,则初始化,采用CAS算法自旋确保并发
                if (tab == null || (n = tab.length) == 0) {
                    n = (sc > c) ? sc : c;
                    if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                        try {
                            if (table == tab) {
                                @SuppressWarnings("unchecked")
                                Node<K, V>[] nt = (Node<K, V>[]) new Node<?, ?>[n];
                                table = nt;
                                sc = n - (n >>> 2);
                            }
                        } finally {
                            sizeCtl = sc;
                        }
                    }
                } else if (c <= sc || n >= MAXIMUM_CAPACITY)
                    //若已超出最大值.直接退出
                    break;
                else if (tab == table) {
                    int rs = resizeStamp(n);
                    if (sc < 0) {
                        Node<K, V>[] nt;
                        if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                                sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                                transferIndex <= 0)
                            break;
                        if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                            //数据迁移
                            transfer(tab, nt);
                    } else if (U.compareAndSwapInt(this, SIZECTL, sc,
                            (rs << RESIZE_STAMP_SHIFT) + 2))
                        transfer(tab, null);
                }
            }
        }
## transfer方法
         /** 数据迁移方法
          * Moves and/or copies the nodes in each bin to new table. See
          * above for explanation.
          * 
          * transferIndex 表示转移时的下标，初始为扩容前的 length。
          * 
          * 我们假设长度是 32
          */
         private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
             int n = tab.length, stride;
             // 将 length / 8 然后除以 CPU核心数。如果得到的结果小于 16，那么就使用 16。
             // 这里的目的是让每个 CPU 处理的桶一样多，避免出现转移任务不均匀的现象，如果桶较少的话，默认一个 CPU（一个线程）处理 16 个桶
             if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
                 stride = MIN_TRANSFER_STRIDE; // subdivide range 细分范围 stridea：TODO
             // 新的 table 尚未初始化
             if (nextTab == null) {            // initiating
                 try {
                     // 扩容  2 倍
                     Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                     // 更新
                     nextTab = nt;
                 } catch (Throwable ex) {      // try to cope with OOME
                     // 扩容失败， sizeCtl 使用 int 最大值。
                     sizeCtl = Integer.MAX_VALUE;
                     return;// 结束
                 }
                 // 更新成员变量
                 nextTable = nextTab;
                 // 更新转移下标，就是 老的 tab 的 length
                 transferIndex = n;
             }
             // 新 tab 的 length
             int nextn = nextTab.length;
             // 创建一个 fwd 节点，用于占位。当别的线程发现这个槽位中是 fwd 类型的节点，则跳过这个节点。
             ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
             // 首次推进为 true，如果等于 true，说明需要再次推进一个下标（i--），反之，如果是 false，那么就不能推进下标，需要将当前的下标处理完毕才能继续推进
             boolean advance = true;
             // 完成状态，如果是 true，就结束此方法。
             boolean finishing = false; // to ensure sweep before committing nextTab
             // 死循环,i 表示下标，bound 表示当前线程可以处理的当前桶区间最小下标
             for (int i = 0, bound = 0;;) {
                 Node<K,V> f; int fh;
                 // 如果当前线程可以向后推进；这个循环就是控制 i 递减。同时，每个线程都会进入这里取得自己需要转移的桶的区间
                 while (advance) {
                     int nextIndex, nextBound;
                     // 对 i 减一，判断是否大于等于 bound （正常情况下，如果大于 bound 不成立，说明该线程上次领取的任务已经完成了。那么，需要在下面继续领取任务）
                     // 如果对 i 减一大于等于 bound（还需要继续做任务），或者完成了，修改推进状态为 false，不能推进了。任务成功后修改推进状态为 true。
                     // 通常，第一次进入循环，i-- 这个判断会无法通过，从而走下面的 nextIndex 赋值操作（获取最新的转移下标）。其余情况都是：如果可以推进，将 i 减一，然后修改成不可推进。如果 i 对应的桶处理成功了，改成可以推进。
                     if (--i >= bound || finishing)
                         advance = false;// 这里设置 false，是为了防止在没有成功处理一个桶的情况下却进行了推进
                     // 这里的目的是：1. 当一个线程进入时，会选取最新的转移下标。2. 当一个线程处理完自己的区间时，如果还有剩余区间的没有别的线程处理。再次获取区间。
                     else if ((nextIndex = transferIndex) <= 0) {
                         // 如果小于等于0，说明没有区间了 ，i 改成 -1，推进状态变成 false，不再推进，表示，扩容结束了，当前线程可以退出了
                         // 这个 -1 会在下面的 if 块里判断，从而进入完成状态判断
                         i = -1;
                         advance = false;// 这里设置 false，是为了防止在没有成功处理一个桶的情况下却进行了推进
                     }// CAS 修改 transferIndex，即 length - 区间值，留下剩余的区间值供后面的线程使用
                     else if (U.compareAndSwapInt
                              (this, TRANSFERINDEX, nextIndex,
                               nextBound = (nextIndex > stride ?
                                            nextIndex - stride : 0))) {
                         bound = nextBound;// 这个值就是当前线程可以处理的最小当前区间最小下标
                         i = nextIndex - 1; // 初次对i 赋值，这个就是当前线程可以处理的当前区间的最大下标
                         advance = false; // 这里设置 false，是为了防止在没有成功处理一个桶的情况下却进行了推进，这样对导致漏掉某个桶。下面的 if (tabAt(tab, i) == f) 判断会出现这样的情况。
                     }
                 }// 如果 i 小于0 （不在 tab 下标内，按照上面的判断，领取最后一段区间的线程扩容结束）
                 //  如果 i >= tab.length(不知道为什么这么判断)
                 //  如果 i + tab.length >= nextTable.length  （不知道为什么这么判断）
                 if (i < 0 || i >= n || i + n >= nextn) {
                     int sc;
                     if (finishing) { // 如果完成了扩容
                         nextTable = null;// 删除成员变量
                         table = nextTab;// 更新 table
                         sizeCtl = (n << 1) - (n >>> 1); // 更新阈值
                         return;// 结束方法。
                     }// 如果没完成
                     if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {// 尝试将 sc -1. 表示这个线程结束帮助扩容了，将 sc 的低 16 位减一。
                         if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)// 如果 sc - 2 不等于标识符左移 16 位。如果他们相等了，说明没有线程在帮助他们扩容了。也就是说，扩容结束了。
                             return;// 不相等，说明没结束，当前线程结束方法。
                         finishing = advance = true;// 如果相等，扩容结束了，更新 finising 变量
                         i = n; // 再次循环检查一下整张表
                     }
                 }
                 else if ((f = tabAt(tab, i)) == null) // 获取老 tab i 下标位置的变量，如果是 null，就使用 fwd 占位。
                     advance = casTabAt(tab, i, null, fwd);// 如果成功写入 fwd 占位，再次推进一个下标
                 else if ((fh = f.hash) == MOVED)// 如果不是 null 且 hash 值是 MOVED。
                     advance = true; // already processed // 说明别的线程已经处理过了，再次推进一个下标
                 else {// 到这里，说明这个位置有实际值了，且不是占位符。对这个节点上锁。为什么上锁，防止 putVal 的时候向链表插入数据
                     synchronized (f) {
                         // 判断 i 下标处的桶节点是否和 f 相同
                         if (tabAt(tab, i) == f) {
                             Node<K,V> ln, hn;// low, height 高位桶，低位桶
                             // 如果 f 的 hash 值大于 0 。TreeBin 的 hash 是 -2
                             if (fh >= 0) {
                                 // 对老长度进行与运算（第一个操作数的的第n位于第二个操作数的第n位如果都是1，那么结果的第n为也为1，否则为0）
                                 // 由于 Map 的长度都是 2 的次方（000001000 这类的数字），那么取于 length 只有 2 种结果，一种是 0，一种是1
                                 //  如果是结果是0 ，Doug Lea 将其放在低位，反之放在高位，目的是将链表重新 hash，放到对应的位置上，让新的取于算法能够击中他。
                                 int runBit = fh & n;
                                 Node<K,V> lastRun = f; // 尾节点，且和头节点的 hash 值取于不相等
                                 // 遍历这个桶
                                 for (Node<K,V> p = f.next; p != null; p = p.next) {
                                     // 取于桶中每个节点的 hash 值
                                     int b = p.hash & n;
                                     // 如果节点的 hash 值和首节点的 hash 值取于结果不同
                                     if (b != runBit) {
                                         runBit = b; // 更新 runBit，用于下面判断 lastRun 该赋值给 ln 还是 hn。
                                         lastRun = p; // 这个 lastRun 保证后面的节点与自己的取于值相同，避免后面没有必要的循环
                                     }
                                 }
                                 if (runBit == 0) {// 如果最后更新的 runBit 是 0 ，设置低位节点
                                     ln = lastRun;
                                     hn = null;
                                 }
                                 else {
                                     hn = lastRun; // 如果最后更新的 runBit 是 1， 设置高位节点
                                     ln = null;
                                 }// 再次循环，生成两个链表，lastRun 作为停止条件，这样就是避免无谓的循环（lastRun 后面都是相同的取于结果）
                                 for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                     int ph = p.hash; K pk = p.key; V pv = p.val;
                                     // 如果与运算结果是 0，那么就还在低位
                                     if ((ph & n) == 0) // 如果是0 ，那么创建低位节点
                                         ln = new Node<K,V>(ph, pk, pv, ln);
                                     else // 1 则创建高位
                                         hn = new Node<K,V>(ph, pk, pv, hn);
                                 }
                                 // 其实这里类似 hashMap 
                                 // 设置低位链表放在新链表的 i
                                 setTabAt(nextTab, i, ln);
                                 // 设置高位链表，在原有长度上加 n
                                 setTabAt(nextTab, i + n, hn);
                                 // 将旧的链表设置成占位符
                                 setTabAt(tab, i, fwd);
                                 // 继续向后推进
                                 advance = true;
                             }// 如果是红黑树
                             else if (f instanceof TreeBin) {
                                 TreeBin<K,V> t = (TreeBin<K,V>)f;
                                 TreeNode<K,V> lo = null, loTail = null;
                                 TreeNode<K,V> hi = null, hiTail = null;
                                 int lc = 0, hc = 0;
                                 // 遍历
                                 for (Node<K,V> e = t.first; e != null; e = e.next) {
                                     int h = e.hash;
                                     TreeNode<K,V> p = new TreeNode<K,V>
                                         (h, e.key, e.val, null, null);
                                     // 和链表相同的判断，与运算 == 0 的放在低位
                                     if ((h & n) == 0) {
                                         if ((p.prev = loTail) == null)
                                             lo = p;
                                         else
                                             loTail.next = p;
                                         loTail = p;
                                         ++lc;
                                     } // 不是 0 的放在高位
                                     else {
                                         if ((p.prev = hiTail) == null)
                                             hi = p;
                                         else
                                             hiTail.next = p;
                                         hiTail = p;
                                         ++hc;
                                     }
                                 }
                                 // 如果树的节点数小于等于 6，那么转成链表，反之，创建一个新的树
                                 ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                     (hc != 0) ? new TreeBin<K,V>(lo) : t;
                                 hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                     (lc != 0) ? new TreeBin<K,V>(hi) : t;
                                 // 低位树
                                 setTabAt(nextTab, i, ln);
                                 // 高位数
                                 setTabAt(nextTab, i + n, hn);
                                 // 旧的设置成占位符
                                 setTabAt(tab, i, fwd);
                                 // 继续向后推进
                                 advance = true;
                             }
                         }
                     }
                 }
             }
         }

## helpTransfer方法
        /** 
         * 帮助执行扩容（复制扩容）
         * 关于 sizeCtl 变量：
         * -1 :代表table正在初始化,其他线程应该交出CPU时间片; 
         * -N: 表示正有N-1个线程执行扩容操作（高 16 位是 length 生成的标识符，低 16 位是扩容的线程数）
         * 大于 0: 如果table已经初始化,代表table容量,默认为table大小的0.75,如果还未初始化,代表需要初始化的大小
         *
         * @param tab 当前table数组
         * @param f   当前槽位的变量
         * @return
         */
        final Node<K, V>[] helpTransfer(Node<K, V>[] tab, Node<K, V> f) {
            Node<K, V>[] nextTab;
            int sc;
            //数据校验 : table数组不为空 , 且node节点为转移类型 , 且node节点的nextTable不为空. 尝试帮助扩容
            if (tab != null && (f instanceof MyConcurrentHashMap.ForwardingNode) &&
                    (nextTab = ((ForwardingNode<K, V>) f).nextTable) != null) {
                //根据数组长度计算得到一个标识符号
                int rs = resizeStamp(tab.length);
                //如果nextTab和tab没有被并发修改,且sizeCtl < 0[表示正在扩容],
                while (nextTab == nextTable && table == tab &&
                        (sc = sizeCtl) < 0) {
                    //如果sizeCtl无符号右移16位不等于rs[即若sc的前16位不等于标识符,说明标识符发生了变化.这时将结束执行]
                    //或者若sc = rs + 1,说明sc此时为非负数,表示此时扩容已结束
                    //或者sc = rs + 65535[允许的最大帮助线程数量],不在执行
                    //或者transferIndex <= 0[转移下标发生了调整,扩容结束]
                    //满足上述一点,结束循环,返回table
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                            sc == rs + MAX_RESIZERS || transferIndex <= 0)
                        break;
                    //将SIZECTL + 1,表示此时辅助扩容的线程又多了一个 . SIZECTL为共享变量,可以在其他线程执行时实时读取到.起到线程间通知的效果
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                        //转移节点
                        transfer(tab, nextTab);
                        break;
                    }
                }
                return nextTab;
            }
            return table;
        }
## balanceInsertion方法

            /** 平衡红黑树的结构
             * 插入后用于维持红黑树性质的修复操作，
             * @param root
             * @param x
             * @param <K>
             * @param <V>
             * @return
             */
            static <K,V> TreeNode<K,V> balanceInsertion(TreeNode<K,V> root,
                                                        TreeNode<K,V> x) {
                x.red = true;//插入的结点设为红色
                for (TreeNode<K,V> xp, xpp, xppl, xppr;;) {
                    if ((xp = x.parent) == null) {
                        x.red = false;//x的父亲为null代表x是根结点，x改黑色直接结束
                        return x;
                    }
                    else if (!xp.red || (xpp = xp.parent) == null)
                        return root;//若x的父结点为黑色或者x的父亲为根结点(实际上根应该是黑色)插入红色结点不影响红黑树性质
                    if (xp == (xppl = xpp.left)) {
                        if ((xppr = xpp.right) != null && xppr.red) {
                            //xppr为x的叔叔，且叔叔为红色，x的叔叔和父亲改为红色，x的爷爷改为黑色，x指针上移到爷爷的位置
                            xppr.red = false;
                            xp.red = false;
                            xpp.red = true;
                            x = xpp;
                        }
                        else {
                            if (x == xp.right) {
                                //情况2，x的叔叔是黑色且x是右儿子。对x上升至父亲后执行一次左旋
                                root = rotateLeft(root, x = xp);
                                xpp = (xp = x.parent) == null ? null : xp.parent;
                            }
                            if (xp != null) {
                                //情况3，x的叔叔是黑色且x是左儿子。x的父亲改黑色，x的爷爷改红色后对x的爷爷进行右旋
                                xp.red = false;
                                if (xpp != null) {
                                    xpp.red = true;
                                    root = rotateRight(root, xpp);
                                }
                            }
                        }
                    }
                    else {//以下为对称的操作.平衡红黑树的节点数
                        if (xppl != null && xppl.red) {
                            xppl.red = false;
                            xp.red = false;
                            xpp.red = true;
                            x = xpp;
                        }
                        else {
                            if (x == xp.left) {
                                //右旋
                                root = rotateRight(root, x = xp);
                                xpp = (xp = x.parent) == null ? null : xp.parent;
                            }
                            if (xp != null) {
                                xp.red = false;
                                if (xpp != null) {
                                    xpp.red = true;
                                    //左旋
                                    root = rotateLeft(root, xpp);
                                }
                            }
                        }
                    }
                }
            }
## treeifyBin方法
        /** 链表转红黑树
         * 数组中index下的链表数据进行树形化转换
         */
        private final void treeifyBin(Node<K, V>[] tab, int index) {
            Node<K, V> b;
            int n, sc;
            if (tab != null) {
                //若当前数组长度<64.
                if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
                    //n <<1 = n*2 对table进行扩容
                    tryPresize(n << 1);
                else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
                    synchronized (b) {
                        if (tabAt(tab, index) == b) {
                            TreeNode<K, V> hd = null, tl = null;
                            //1.取出下标index的数据节点
                            //2.依次循环将Node节点对象转为TreeNode
                            for (Node<K, V> e = b; e != null; e = e.next) {
                                TreeNode<K, V> p =
                                        new TreeNode<K, V>(e.hash, e.key, e.val,
                                                null, null);
                                if ((p.prev = tl) == null)
                                    //若节点P的父节点数据为空,p为该树结构的头节点
                                    hd = p;
                                else
                                    //不然,将tl的下节点设置为P
                                    tl.next = p;
                                //重置tl临时变量,便于下次循环处理prev/next[上下节点指针]
                                tl = p;
                            }
                            //重设数组数据
                            setTabAt(tab, index, new TreeBin<K, V>(hd));
                        }
                    }
                }
            }
        }
## 总结
ConcurrentHashMap 在 JDK 1.7 和 1.8 变化很大，在 JDK 1.7 中，采用 Segment 分段存储数据，也通过 Segment 分段加锁。    

而在 JDK 1.8 中，使用 synchronized 锁定 hash 桶的链表的首节点/红黑树的根节点，只要 hash(key) 不冲突，就不会影响其他线程。  
