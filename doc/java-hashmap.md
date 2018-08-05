# HashMap原理

### 前言：

HashMap在Java开发中非常常见，作为Java程序员，了解这个类可以说是一种修养了（毕竟每天见到，不知道内部原理说不过去呀～）

> 这里的HashMap的分析是基于Java 1.8的



### 简介：

在我们计算机学科，要学习一个东西，最好，最有效的方法就是看他的官方文档。在这里就是Javadoc，所以我们第一步，来看一下官方是怎么描述这个类的：('-'之后为本人翻译的中文总结)

```Java
/**
 * Hash table based implementation of the <tt>Map</tt> interface.  This
 * implementation provides all of the optional map operations, and permits
 * <tt>null</tt> values and the <tt>null</tt> key.  (The <tt>HashMap</tt>
 * class is roughly equivalent to <tt>Hashtable</tt>, except that it is
 * unsynchronized and permits nulls.)  This class makes no guarantees as to
 * the order of the map; in particular, it does not guarantee that the order
 * will remain constant over time.
 - 实现了Map接口，提供所有map操作
 - 允许null key和null value
 - 不同步（和Hashtable基本相同，除了【不同步】和【允许null值】）
 - 不保证顺序（可能一段时间之后，读取的顺序就发生了变化）（rehashing带来的问题）
 
 *
 * <p>This implementation provides constant-time performance for the basic
 * operations (<tt>get</tt> and <tt>put</tt>), assuming the hash function
 * disperses the elements properly among the buckets.  Iteration over
 * collection views requires time proportional to the "capacity" of the
 * <tt>HashMap</tt> instance (the number of buckets) plus its size (the number
 * of key-value mappings).  Thus, it's very important not to set the initial
 * capacity too high (or the load factor too low) if iteration performance is
 * important.
 - 提供常数时间的基本操作（get/put）
 - 集合迭代的时间和 capacity+size 成正相关（因此，需要好的迭代表现，需要capacity不要太高/load factor太低）(这几个参数，上面的英文和后文里有解释，这里就不写出了)
 
 *
 * <p>An instance of <tt>HashMap</tt> has two parameters that affect its
 * performance: <i>initial capacity</i> and <i>load factor</i>.  The
 * <i>capacity</i> is the number of buckets in the hash table, and the initial
 * capacity is simply the capacity at the time the hash table is created.  The
 * <i>load factor</i> is a measure of how full the hash table is allowed to
 * get before its capacity is automatically increased.  When the number of
 * entries in the hash table exceeds the product of the load factor and the
 * current capacity, the hash table is <i>rehashed</i> (that is, internal data
 * structures are rebuilt) so that the hash table has approximately twice the
 * number of buckets.
 - 两个影响HashMap表现的因素：initial capacity 和 load factor
 - 简单的说，Capacity就是buckets的数目，Load factor就是buckets填满程度的最大比例。如果对迭代性能要求很高的话不要把capacity设置过大，也不要把load factor设置过小。当bucket填充的数目（即hashmap中元素的个数）大于capacity*load factor时就调整buckets的数目为当前的2倍。
 
 *
 * <p>As a general rule, the default load factor (.75) offers a good
 * tradeoff between time and space costs.  Higher values decrease the
 * space overhead but increase the lookup cost (reflected in most of
 * the operations of the <tt>HashMap</tt> class, including
 * <tt>get</tt> and <tt>put</tt>).  The expected number of entries in
 * the map and its load factor should be taken into account when
 * setting its initial capacity, so as to minimize the number of
 * rehash operations.  If the initial capacity is greater than the
 * maximum number of entries divided by the load factor, no rehash
 * operations will ever occur.
 - entries数目的预估来设置initial capacity可以有效的减少rehashing的次数
 
 *
 * <p>If many mappings are to be stored in a <tt>HashMap</tt>
 * instance, creating it with a sufficiently large capacity will allow
 * the mappings to be stored more efficiently than letting it perform
 * automatic rehashing as needed to grow the table.  Note that using
 * many keys with the same {@code hashCode()} is a sure way to slow
 * down performance of any hash table. To ameliorate impact, when keys
 * are {@link Comparable}, this class may use comparison order among
 * keys to help break ties.
 - 当有很多key有相同的hashcode，肯定会对效率产生影响
 - 为了改善影响，这个类may(这里咋翻译，是可能么，还没有验证)用比较顺序来帮助打破这里的问题
 
 *
 * <p><strong>Note that this implementation is not synchronized.</strong>
 * If multiple threads access a hash map concurrently, and at least one of
 * the threads modifies the map structurally, it <i>must</i> be
 * synchronized externally.  (A structural modification is any operation
 * that adds or deletes one or more mappings; merely changing the value
 * associated with a key that an instance already contains is not a
 * structural modification.)  This is typically accomplished by
 * synchronizing on some object that naturally encapsulates the map.
 - structural modification: 添加/删除一个或多个键值对（影响内部的结构了）
 - 当发生多个线程使用hashmap，且至少有一个线程结构性的改变了这个map，则需要通过封装这个map（装饰器模式）来实现同步
 
 *
 * If no such object exists, the map should be "wrapped" using the
 * {@link Collections#synchronizedMap Collections.synchronizedMap}
 * method.  This is best done at creation time, to prevent accidental
 * unsynchronized access to the map:<pre>
 *   Map m = Collections.synchronizedMap(new HashMap(...));</pre>
 - 一般通过 Map m = Collections.synchronizedMap(new HashMap(...))在初始化的时候封装这个类
 
 *
 * <p>The iterators returned by all of this class's "collection view methods"
 * are <i>fail-fast</i>: if the map is structurally modified at any time after
 * the iterator is created, in any way except through the iterator's own
 * <tt>remove</tt> method, the iterator will throw a
 * {@link ConcurrentModificationException}.  Thus, in the face of concurrent
 * modification, the iterator fails quickly and cleanly, rather than risking
 * arbitrary, non-deterministic behavior at an undetermined time in the
 * future.
 - 有一个fail-fast机制，当在迭代时，map被结构性的改变了，则会【调用迭代器的remove方法】以及【抛出ConcurrentModificationException异常】
 
 *
 * <p>Note that the fail-fast behavior of an iterator cannot be guaranteed
 * as it is, generally speaking, impossible to make any hard guarantees in the
 * presence of unsynchronized concurrent modification.  Fail-fast iterators
 * throw <tt>ConcurrentModificationException</tt> on a best-effort basis.
 * Therefore, it would be wrong to write a program that depended on this
 * exception for its correctness: <i>the fail-fast behavior of iterators
 * should be used only to detect bugs.</i>
 - fail-fast机制是用于检测错误的，并不能保证正确性。因为毕竟非同步的并发操作是不可能作出任何硬性保证的
 
 *
 * <p>This class is a member of the
 * <a href="{@docRoot}/../technotes/guides/collections/index.html">
 * Java Collections Framework</a>.
 *
 * @param <K> the type of keys maintained by this map
 * @param <V> the type of mapped values
 *
 * @author  Doug Lea
 * @author  Josh Bloch
 * @author  Arthur van Hoff
 * @author  Neal Gafter
 * @see     Object#hashCode()
 * @see     Collection
 * @see     Map
 * @see     TreeMap
 * @see     Hashtable
 * @since   1.2
 */
```

总结一下：

首先介绍了特性：

 - 实现了Map接口，提供所有map操作
 - 允许null key和null value
 - 不同步（和Hashtable基本相同，除了【不同步】和【允许null值】）
 - 不保证顺序（可能一段时间之后，读取的顺序就发生了变化）（rehashing带来的问题）

接着介绍了影响HashMap表现的因素：

- Capacity：buckets的数目
- Load factor：buckets填满程度的最大比例。
- 如果对**迭代性能**要求很高的话不要把capacity设置过大，也不要把load factor设置过小。当bucket填充的数目（即hashmap中元素的个数）大于capacity*load factor时就调整buckets的数目为当前的2倍。
- entries数目的预估来设置initial capacity可以有效的**减少rehashing的次数**
- 当有很多key有相同的hashcode，肯定会对效率产生影响。为了**改善key hashcode重复的影响**，这个类may(这里咋翻译，是可能么，还没有验证有没有实现)用比较顺序来帮助打破这里的问题

最后介绍了并发的同步处理和非同步处理：

 - structural modification: 添加/删除一个或多个键值对（影响内部的结构了）

 - 同步处理：当发生多个线程使用hashmap，且至少有一个线程结构性的改变了这个map，则需要通过封装这个map（装饰器模式）来实现同步。一般通过 

   ```Java
   Map m = Collections.synchronizedMap(new HashMap(…))
   ```

   在初始化的时候封装这个类。

- 非同步处理（不进行处理）：

  - 有一个fail-fast机制，当在迭代时，map被结构性的改变了，则会【调用迭代器的remove方法】以及【抛出ConcurrentModificationException异常】
  - fail-fast机制是用于检测错误的，并不能保证正确性。因为毕竟非同步的并发操作是不可能作出任何硬性保证的




### Put函数实现：



### Get函数实现：



### Reference:

[Java HashMap工作原理及实现](https://yikun.github.io/2015/04/01/Java-HashMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)

[HashMap源码1.7分析](https://www.jianshu.com/p/4ee6cbb83cc8)