---
title: Spark Shuffle 阶段用到的重要数据结构
tags: Spark,Shuffle,AppendOnlyMap
grammar_cjkRuby: true
grammar_mindmap: true
renderNumberedHeading: true
---
<span style="font-size: 12px;float:right;color: black;margin-right: 100px">作者:大数据spark源码研究小组</span>
#### SizeTracker
`重要程度***`
- SizeTracker  定义了对集合进行采样和集合所占内存字节大小的估算.
- SizeTracker重要的属性:
    - SAMPLE_GROWTH_RATE:Double 采样增长的速率,固定值1.1
    - samples:mutable.Queue 用于存储采集样本的队列,在该队列中,最多只能保存两个样本,且是最近采集的样本.
    - bytesPerUpdate: Double 平均每次更新集合的字节数,这里的更新主要是指插入新元素或者修改元素.
    - numUpdates: Long 从初始化样本或者重置样本开始,更新集合的总次数
    - nextSampleNum: Long 达到下次采集样本时对集合的操作总次数
 - SizeTracker重要的方法:
	- resetSamples() 初始化或者重置样本采集评判标准,将numUpdates nextSampleNum 的值都初始化为1,同时对集合当前状态进行一次采样.
	- afterUpdate() 对集合进行更新操作以后,会调用该方法,判断是不是需要进行下一次采样.
	- takeSample() 样本采集,估算当前集合的大小,并将集合的大小和当前更新集合操作的总次数封装到Sample样本类中,放入到样本队列,放入后需要判断当前样本队列中样本数量,如果>2,需要将最先加入到队列中的样本移除队列,以保证样本队列中最多只保存最近采集的两个样本.代码:
		```scala
			private def takeSample(): Unit = {
			// 对集合当前的状态进行大小估算,并将估算的大小以及当前对集合
			//操作的总次数封装到样本对象Sample中并将样本添加到样本队列
			samples.enqueue(Sample(SizeEstimator.estimate(this), numUpdates))
			// 保证样本队列存的是最近采集的两个样本
			if (samples.size > 2) {
			  samples.dequeue()
			}
			// 截止到本次样本采集,计算平均每次更新样本时的字节大小
			val bytesDelta = samples.toList.reverse match {
			  case latest :: previous :: tail =>
				//                      (本次采样大小 - 上次采样大小)
				//    bytesPerUpdate = ---------------------------
				//                      (本次采样编号 - 上次采样编号)
				(latest.size - previous.size).toDouble / (latest.numUpdates - previous.numUpdates)
			  // If fewer than 2 samples, assume no change
			  case _ => 0
			}
			// 将计算出来的最新的平均每次更新集合的字节大小更新到bytesPerUpdate
			bytesPerUpdate = math.max(0, bytesDelta)
			// 计算到下一次采集样本时对集合更新的总次数
			nextSampleNum = math.ceil(numUpdates * SAMPLE_GROWTH_RATE).toLong
		  }
		```
		
	 
	- estimateSize() 估算当前状态下集合的大小:估算的公式: 集合大小 = 上一次采集样本时集合的大小 + (最近采集样本时的大小 - 上一次采集样本的大小),代码如下:
		```scala
		  def estimateSize(): Long = {
			assert(samples.nonEmpty)
			// 计算本次采样增加的大小
			val extrapolatedDelta = bytesPerUpdate * (numUpdates - samples.last.numUpdates)
			// 相加
			(samples.last.size + extrapolatedDelta).toLong
		  }
		```
#### WritablePartitionedPairCollection
`重要程度***`
   - WritablePartitionedPairCollection 接口定义了将带有分区信息的key-value键值对插入到集合,并提供了获取可以将集合的内容按照字节写入磁盘的WritablePartitionedIterator迭代器.在其伴生类中,还提供了根据指定的key比较器获取默认的根据分区ID进行比较,或者既根据分区ID进行比较,也根据key-value键值对的key进行比较的比较器.
   - WritablePartitionedPairCollection的方法:
     - insert(partition: Int, key: K, value: V) 将键值对以及key对应的分区ID插入到集合中,抽象方法,需要子类去实现.
     - partitionedDestructiveSortedIterator(keyComparator: Option[Comparator[K]]): Iterator[((Int, K), V)]  根据指定的Key的比较器,返回分区有序,分区内元素也有序的迭代器,如果没有指定key的比较器,也即比较器参数传None,那么只返回分区有序的迭代器,抽象方法,需要子类去实现.
     - destructiveSortedWritablePartitionedIterator(keyComparator: Option[Comparator[K]]): WritablePartitionedIterator 创建可以将集合的内容按照字节写入磁盘的WritablePartitionedIterator迭代器,如果key的比较器没有传,也即是None时,这个迭代器分区有序的,如果传了key比较器,那么返回的这个迭代器分区有序,分区内元素也有序,源码如下:
		``` scala
			def destructiveSortedWritablePartitionedIterator(keyComparator: Option[Comparator[K]])
			  : WritablePartitionedIterator = {
				// 调用当前类的partitionedDestructiveSortedIterator方法生成迭代器
				val it = partitionedDestructiveSortedIterator(keyComparator)
				// 创建WritablePartitionedIterator匿名实现类的对象
				new WritablePartitionedIterator {
				  private[this] var cur = if (it.hasNext) it.next() else null
				 // 实现了将元素按照字节写入到磁盘的方法,这里写只是写元素的key和value,
				 // 并没有写分区信息
				  override def writeNext(writer: DiskBlockObjectWriter): Unit = {
					writer.write(cur._1._2, cur._2)
					cur = if (it.hasNext) it.next() else null
				  }

				  override def hasNext(): Boolean = cur != null

				  override def nextPartition(): Int = cur._1._1
				}
			  }
			  
			 private[spark] trait WritablePartitionedIterator {
				 // 可以将集合元素按照字节写入到磁盘中
				def writeNext(writer: DiskBlockObjectWriter): Unit
				def hasNext(): Boolean
				def nextPartition(): Int
			 }
		```
- 在伴生类中提供的两个获取迭代器的方法:
	 ```scala
	   /**
		* 此方法用于获取只比较分区ID的比较器
		* 这里的key其实是(partitionID,key)的形式
		*/
	  def partitionComparator[K]: Comparator[(Int, K)] = new Comparator[(Int, K)] {
		override def compare(a: (Int, K), b: (Int, K)): Int = {
		  a._1 - b._1
		}
	  }

	   /**
		* 次方法用于先按照分区id比较,如果分区id相同,再按照传入的key的比较器进行比较的比较器
		* 这里的key其实是(partitionID,key)的形式
		*/
	  def partitionKeyComparator[K](keyComparator: Comparator[K]): Comparator[(Int, K)] = {
		new Comparator[(Int, K)] {
		  override def compare(a: (Int, K), b: (Int, K)): Int = {
			val partitionDiff = a._1 - b._1
			if (partitionDiff != 0) {
			  partitionDiff
			} else {
			  keyComparator.compare(a._2, b._2)
			}
		  }
		}
	  }
	 ```
#### AppendOnlyMap
`重要程度*****`
 - AppendOnlyMap 单从命名上来看,是一个只能追加元素的Map结构.其实AppendOnlyMap确实只能追加或者修改元素,不能删除元素,而且它底层是使用数组结构实现的,这个数组结构具有有效性,在调用`destructiveSortedIterator`方法时会破坏这个数组的有效性.AppendOnlyMap是内存中在Shuffle阶段存储数据的一个重要数据结构,它提供了修改或插入元素,聚合元素的方法.但是在插入或者修改元素的时候,会有判断扩容数组容量的过程,如果达到扩容标准,将会对数组2倍容量进行扩容,扩容过程中原有元素并不是直接拷贝,而是进行原有元素的重新定位存储,如果集合中存在的数据量大,那么这里的操作将会耗时好资源的.AppenOnlyMap中最多能存储375809638 (0.7 * 2 ^ 29)个元素
 - AppendOnlyMap的属性:
   - LOAD_FACTOR 用于计算data数组容量增长的阈值的负载因子,固定值0.7
   - capacity data数组中能存储的元素容量,会根据元素的不断追加,这个容量也会不断的扩容,每次扩容都是2倍增加,capacity要求是2的指数次幂
   - mask 用于计算元素存放位置的掩码,值为capacity-1 保证元素的位置必须在数组的范围之内
   - curSize 当前集合中元素的个数
   - growThreshold data数组扩容的阈值,也就是当数组中存放元素的容量达到growThreshold,素组将会进行扩容.
   - data 数组,用于存储key-value组数的元素,这样的元素在数组中存放的格式为k1 v1 k2 v2 k3 v3 ...
   - nullValue 空值. 在AppendOnlyMap中,可以存放key为空的元素.但是这个key为空的元素并不存放到data数组中,而是将key为null的元素的值单独用一个变量nullValue来存储,并用属性haveNullValue来标识当前AppendOnlyMap中是否已经存储了空值.
   - haveNullValue 是否已经存储了空值
   - destroyed 数组有效性标识
 - AppendOnlyMap的重要的方法:
   - apply(key: K) 根据所给的key进行取值
   - update(key: K, value: V) 插入或者更新元素,代码如下:
		```scala
			def update(key: K, value: V): Unit = {
				assert(!destroyed, destructionMessage)
				val k = key.asInstanceOf[AnyRef]
				//判断key是不是为null
				if (k.eq(null)) { //如果key为null
					// 判断是不是已经存在null值
				  if (!haveNullValue) { //如果还没有存在空值
					incrementSize() //进行扩容判断
				  }
				  nullValue = value //将当前null对应的value赋值给nullValue属性
				  haveNullValue = true //并将是否存在空值的属性置为true
				  return
				}
				var pos = rehash(key.hashCode) & mask //计算key所在的位置
				var i = 1
				while (true) {
				  val curKey = data(2 * pos)  //获取数组中当前位置上的key
				  //如果当前位置还没有存过值,那么就要存的元素存到该位置,并进行扩容判断
				  if (curKey.eq(null)) { 
					data(2 * pos) = k 
					data(2 * pos + 1) = value.asInstanceOf[AnyRef]
					incrementSize() // Since we added a new key
					return
					//         ==
				  } 
				  //如果得到的当前位置的key不为null且与当前要存的元素的key相等,
				  //这将该位置上的value更新为要存元素的value
				  else if (k.eq(curKey) || k.equals(curKey)) { 
					data(2 * pos + 1) = value.asInstanceOf[AnyRef]
					return
				  } 
				  // 如果得到的当前位置的key不为null但是和当前要存的元素的key不相等,
				  // 也即是发生了hash冲突
				  else { 
					val delta = i
					// 当前位置+1 与mask计算后重复判断,直到满足上面两个条件为止
					pos = (pos + delta) & mask
					// eg: mask = 15
					// 如果计算出来的pos位置为14,但是在14的位置发生了hash碰撞,那么就
					//14+1继续与15相与运算,得到第0个位置,如果第0个位置还是发生了hash
					// 冲突,那么就用0 +(1+1) = 2 与15相与,
					// 再继续上面两个条件的判断,直到存储进去为止.
					i += 1
				  }
				}
			  }
		```
    - changeValue(key: K, updateFunc: (Boolean, V) => V): V 根据指定的key以及聚合函数,进行key对应的value的聚合,代码如下:
		```scala
			   /**
				*
				* 聚合算法
				* @param key        要聚合的key
				* @param updateFunc 聚合函数,由使用者定义:该函数接收两个参数:
				*                   param1:Boolean  key是否添加到AppendOnlyMap的data中进行过聚合
				*                   param2:表示key曾经添加到AppendOnlyMap的data数组中进行聚合时
				*                   的聚合值,新一轮的聚合值将在之前的聚合值上进行聚合
				*
				* @return
				*/
			  def changeValue(key: K, updateFunc: (Boolean, V) => V): V = {
				assert(!destroyed, destructionMessage) //校验数组的有效性
				val k = key.asInstanceOf[AnyRef]
				if (k.eq(null)) { //判断需要聚合的key是否为null
				//判断是否有空值,如果没有空值,需要将当前的value值通过聚合函数集合后
				//存储当nullValue上,所以这里也需要扩容判断
				  if (!haveNullValue) { 
					incrementSize()
				  }
				  //聚合空值,并将聚合后的值存储到nullValue上
				  nullValue = updateFunc(haveNullValue, nullValue)
				  haveNullValue = true //将是否有空值的标识置位true
				  return nullValue //返回集合后的值
				}
				//如果key不为null.计算当前key对应的位置
				var pos = rehash(k.hashCode) & mask
				var i = 1
				while (true) {
				  val curKey = data(2 * pos) //获取根据key计算出来的位置上的key
				  if (curKey.eq(null)) { //如果当前位置上还没有存储新的值
				  //将null当做该位置上的值使用聚合函数进行聚合后的新值存储到该key
				  // 对应的value的位置上,
				  // 聚合函数的第一个参数传false
					val newValue = updateFunc(false, null.asInstanceOf[V])
					data(2 * pos) = k
					data(2 * pos + 1) = newValue.asInstanceOf[AnyRef]
					incrementSize()//进行扩容判断
					return newValue
				  } 
				  //如果计算出来的位置上已经有存储过值了,并且与当前传入的key相等,
				  //这使用当前key对应的值根据聚合函数进行聚合的到新的值,并使用新值替换原来
				  //位置上的值,这里聚合函数的第一个参数应该传true了
					val newValue = updateFunc(true, data(2 * pos + 1).asInstanceOf[V])
				  else if (k.eq(curKey) || k.equals(curKey)) { 
					data(2 * pos + 1) = newValue.asInstanceOf[AnyRef]
					return newValue
				  }
				  //计算出来的位置上已经存储了值,并且与传进来的key不相等,即发生了hash冲突,
				  // 需要继续往下寻找,直到满足上面两个条件为止.
				  else {
					val delta = i
					pos = (pos + delta) & mask
					i += 1
				  }
				}
				// 在调用这个函数的过程中,并不会走到这,这里只是为了编译通过.
				null.asInstanceOf[V] 
			  }
		```
	- incrementSize() 扩容判断
	- growTable() 进行data数组的扩容,代码如下:
		```scala
		protected def growTable() {
			// capacity < MAXIMUM_CAPACITY (2 ^ 29) so capacity * 2 won't overflow
			val newCapacity = capacity * 2 // 先将capacity*2后负值给要扩容的大小
			// 检查扩容是否小于等于当前集合要求的最大容量
			require(newCapacity <= MAXIMUM_CAPACITY, s"Can't contain more than ${growThreshold} elements")
			// 创建新的数组
			val newData = new Array[AnyRef](2 * newCapacity) // create new array
			// 得到新的掩码
			val newMask = newCapacity - 1
			// Insert all our old values into the new array. Note that because our old keys are
			// unique, there's no need to check for equality here when we insert.
			//遍历元素组数中的元素
			var oldPos = 0
			// 因为掩码变了,所以需要重新将原来数组中的元素根据新的掩码计算新的位置,
			// 重新存储到新的数组中
			while (oldPos < capacity) { 
			  // 计算逻辑与上面update方法一样
			  if (!data(2 * oldPos).eq(null)) { // key not null
				val key = data(2 * oldPos) // get oldPos key value
			  val value = data(2 * oldPos + 1)
			  // 根据新的源码计算新的位置
				var newPos = rehash(key.hashCode) & newMask // [0 - (newCapacity - 1)]
				var i = 1
				var keepGoing = true
				while (keepGoing) {
				//获取新位置上的key,判断是不是等于null,如果等于null,说明还没有被使用,
				//将遍历出来的元素存储到这个新的位置上
				  val curKey = newData(2 * newPos) 
				  if (curKey.eq(null)) {
					newData(2 * newPos) = key
					newData(2 * newPos + 1) = value
					keepGoing = false // 结束循环,继续写一个元素的拷贝
				  } 
				  // 如果计算出来的新的位置上得到的key不等于null,说明已经存储了元素,
				  // 这里不能考虑hash冲突,直接取下一位位置,继续循环判断
				  else {
					val delta = i
					newPos = (newPos + delta) & newMask
					i += 1
				  }
				}
			  }
			  oldPos += 1 //老数组中下一个元素的拷贝
			}
			// 原来数组中的元素拷贝完以后,用新的数组替换原来的数组
			data = newData
			capacity = newCapacity //将新的容量替换原来的容量
			mask = newMask //将新的源码替换成原来的源码
			// 将新的扩容阈值替换成原来的扩容阈值
			growThreshold = (LOAD_FACTOR * newCapacity).toInt 
			//完成以上操作,也就完成的扩容操作
		  }
		```
	- nextPowerOf2(n: Int): Int用于限制用户定义AppendOnlyMap时,传的初始容量不是2的指数次幂.取传进来的n的二进制的最高位,其余位数补0,得到的新整数,如果得到的数与n相等,则结果为n,否则为新整数左移一位得到的结果.
	- rehash(h: Int) 根据元素的key计算hash值的hash函数
	- destructiveSortedIterator(keyComparator: Comparator[K]): Iterator[(K, V)] 根据指定的key的比较器,获取有序元素的迭代器.该方法是不是用额外内存,对集合中的元素进行排序.但是由于在排序过程中,底层数组data存储的元素位置都向下标0方向靠拢,破坏了原来存储进data时元素的存储位置,也就不能再继续使用update,apply,尤其是changeValue这个AppendOnlyMap特色的方法等操作,所以破坏了AppendOnlyMap的有效性.代码如下:
		```scala
			def destructiveSortedIterator(keyComparator: Comparator[K]): Iterator[(K, V)] = {
				destroyed = true //将有效性标志置为false
				// Pack KV pairs into the front of the underlying array
				var keyIndex, newIndex = 0
				while (keyIndex < capacity) {
				  // 将key不为null也即是data数组中存储了元素的位置都向0方向移动
				  // 可以理解为这个方法本身只是为了得到集合中存储的并且排好序的元素
				  if (data(2 * keyIndex) != null) {
					data(2 * newIndex) = data(2 * keyIndex)
					data(2 * newIndex + 1) = data(2 * keyIndex + 1)
					newIndex += 1
				  }
				  keyIndex += 1
				}
				//断言排null后得到的元素个数是不是与curSize这个代表集合中元素个数相等
				assert(curSize == newIndex + (if (haveNullValue) 1 else 0))

				//执行比较,排序,Sorter可自行研究
				new Sorter(new KVArraySortDataFormat[K, AnyRef]).sort(data, 0, newIndex, keyComparator)

				new Iterator[(K, V)] { //生成一个迭代访问data数组中元素的迭代器
				  var i = 0
				  var nullValueReady = haveNullValue

				  def hasNext: Boolean = (i < newIndex || nullValueReady)

				  def next(): (K, V) = {
					if (nullValueReady) {
					  nullValueReady = false
					  (null.asInstanceOf[K], nullValue)
					} else {
					  val item = (data(2 * i).asInstanceOf[K], data(2 * i + 1).asInstanceOf[V])
					  i += 1
					  item
					}
				  }
				}
			  }
		```
	- 在AppendOnlyMap的伴生对象中定义了data数组能扩容到的最大的容量.`val MAXIMUM_CAPACITY = (1 << 29)`
#### SizeTrackingAppendOnlyMap
`重要程度***`
   - SizeTrackingAppendOnlyMap继承了AppendOnlyMap,并实现了SizeTracker接口.所以具有AppendOnlyMap对元素进行聚合,返回有序元素的迭代器等特性,也有采样估算集合本身大小的特性.
   - SizeTrackingAppendOnlyMap的方法:
```scala
   /**
    * 重写父类AppendOnlyMap的update方法
    * @param key
    * @param value
    */
  override def update(key: K, value: V): Unit = {
    // 直接调用父类AppendOnlyMap的update方法
    super.update(key, value)
	// 更新集合以后,回调用SizeTracker的方法afterUpdate,判断是不是需要采样
    super.afterUpdate()
  }
  
   /**
    * 重写父类AppendOnlyMap的changeValue方法
    * 对key对应的value进行聚合
    * updateFunc:聚合函数
    *
    */
  override def changeValue(key: K, updateFunc: (Boolean, V) => V): V = {
    val newValue = super.changeValue(key, updateFunc)
    super.afterUpdate()
    newValue
  }
  
   /**
    * 在扩容后,调用resetSamples()将样本数据进行重置,以便于对AppendOnlyMap的估算更加准确
    */
  override protected def growTable(): Unit = {
    super.growTable()
    resetSamples()
  }
```

#### PartitionedAppendOnlyMap
`重要程度***`
  - PartitionedAppendOnlyMap继承了SizeTrackingAppendOnlyMap,还实现了WritablePartitionedPairCollection接口,实现了WritablePartitionedPairCollection接口中的partitionedDestructiveSortedIterator抽象方法,该集合能对元素进行聚合,能返回元素有序的迭代器,能返回能将有序元素按照字节存储到磁盘的迭代器,还是采样估算自身集合的大小.
  - 在PartitionedAppendOnlyMap中定义或者重写的方法:
```scala
  /**
    * 根据给定的对key进行比较的比较器,返回集合中的数据按照分区ID的顺序进行迭代的迭代器
    *
    * @param keyComparator key比较器
    * @return
    */
  override def partitionedDestructiveSortedIterator(keyComparator: Option[Comparator[K]])
  : Iterator[((Int, K), V)] = {
    //源码:
    /*val comparator = keyComparator.map(partitionKeyComparator).getOrElse(partitionComparator)
    destructiveSortedIterator(comparator)*/
    //分析:
    //keyComparator:对key进行比较的比较器
    //由于该方法的返回值需要返回一个根据分区ID进行排序与key进行排序的迭代器,
	//所以需要将keyComparator转化成先根据分区比较,再根据key比较的比较器
    //optionPartitionKeyComparator 就是转化后的先根据分区比较,再根据key比较的比较器
    //optionPartitionKeyComparator 得到的比较器中如果没有指定key的比较器,
	//也即keyComparator没有指定,这时给一个默认的比较器:调用partitionComparator方法生成的比较器
    val optionPartitionKeyComparator: Option[Comparator[(Int, K)]] = keyComparator.map(oldKeyComparator => partitionKeyComparator(oldKeyComparator))
    val comparator = optionPartitionKeyComparator.getOrElse(partitionComparator)
    // 调用 AppendOnlyMap中定义的的 destructiveSortedIterator方法对集合中的元素进行比较,
	// 然后返回有序结果迭代器
    destructiveSortedIterator(comparator)
  }
  
   /**
    * 插入新的元素,要求插入的元素格式为key:(当前key对应的分区ID,当前key),value
    * @param partition
    * @param key
    * @param value
    */
  override def insert(partition: Int, key: K, value: V): Unit = {
    update((partition, key), value)
  }
```

#### PartitionedPairBuffer
`重要程度*****`
- PartitionedPairBuffer底层使用数组实现的将键值对缓存再内存中,并支持对元素进行排序的缓存数据结构.其实现类了WritablePartitionedPairCollection接口和SizeTracker接口,所以PartitionedPairBuffer能够对其缓存的元素进行按照所给的key比较器返回对key对应的分区进行排序,或者对key对应的分区进行排序,并对key进行排序的迭代器,能够返回将缓存中的元素按排序后照字节写入到磁盘的迭代器,同时还可以对自身进行采样并估算自身大小.
- PartitionedPairBuffer的属性:
   - capacity 当前缓存存储元素的容量,初始大小等于使用者定义的创建缓存对象时传的initialCapacity,默认64,最大值不能大于PartitionedPairBuffer伴生对象中定义的MAXIMUM_CAPACITY = Int.MaxValue / 2 
   - curSize 当前混存中元素的个数
   - data 真正用于存储数据的数组,其数组长度为capacity的两倍,存储元素的格式为k1,v1,k2,v2...
- PartitionedPairBuffer的方法:
	```scala
	   /**
		* 插入元素key:(partitionID,key)
		*
		* @param partition
		* @param key
		* @param value
		*/
	  def insert(partition: Int, key: K, value: V): Unit = {
		// 先判断当前容量是否还够,不够就进行扩容
		if (curSize == capacity) {
		  growArray()
		}
		// 将新的元素直接插入到数组中已有元素的下一个位置
		data(2 * curSize) = (partition, key.asInstanceOf[AnyRef])
		data(2 * curSize + 1) = value.asInstanceOf[AnyRef]
		curSize += 1
		// 插入后元素调用after方法进行采样监听
		afterUpdate()
	  }

		/**
		* 扩容
		*/
	  private def growArray(): Unit = {
		if (capacity >= MAXIMUM_CAPACITY) {
		  throw new IllegalStateException(s"Can't insert more than ${MAXIMUM_CAPACITY} elements")
		}
		val newCapacity =
		// 要保证容量*2不能超过最大值MAXIMUM_CAPACITY,如果操作,直接为最大值
		  if (capacity * 2 < 0 || capacity * 2 > MAXIMUM_CAPACITY) {
			MAXIMUM_CAPACITY
		  } else {
			capacity * 2
		  }
		  //定义新数组
		val newArray = new Array[AnyRef](2 * newCapacity)
		// 数组中的元素拷贝
		System.arraycopy(data, 0, newArray, 0, 2 * capacity)
		data = newArray
		capacity = newCapacity
		//重置采样
		resetSamples()
	  }

		/**
		* 对数组中的元素进行比较排序,然后返回排序后的结果的迭代器
		*
		* @param keyComparator key比较器
		* @return
		*/
	  override def partitionedDestructiveSortedIterator(keyComparator: Option[Comparator[K]])
	  : Iterator[((Int, K), V)] = {
		val comparator = keyComparator.map(partitionKeyComparator).getOrElse(partitionComparator)
		// 进行比较排序
		new Sorter(new KVArraySortDataFormat[(Int, K), AnyRef]).sort(data, 0, curSize, comparator)
		iterator()
	  }


		/**
		* 获取迭代器,用于遍历缓存中的元素
		* @return
		*/
	  private def iterator(): Iterator[((Int, K), V)] = new Iterator[((Int, K), V)] {
		var pos = 0

		override def hasNext: Boolean = pos < curSize

		override def next(): ((Int, K), V) = {
		  if (!hasNext) {
			throw new NoSuchElementException
		  }
		  val pair = (data(2 * pos).asInstanceOf[(Int, K)], data(2 * pos + 1).asInstanceOf[V])
		  pos += 1
		  pair
		}
	  }
	```
 
