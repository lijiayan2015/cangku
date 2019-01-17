---
title: Spark Shuffle 阶段用到的重要的数据结构
tags: Spark,Shuffle,AppendOnlyMap
grammar_cjkRuby: true
---

#### SizeTracker
- SizeTracker  定义了对集合已经采样和集合所占内存字节大小的估算.
- SizeTracker的结构图:
   <img src='https://www.github.com/lijiayan2015/cangku/raw/master/小书匠/1547629661466.png' width='300'>
- SizeTracker重要的属性:
    - SAMPLE_GROWTH_RATE:Double 采样增长的速率,固定值1.1
    - samples:mutable.Queue 用于存储采集的样本,在该队列中,对多只能保存两个样本,且是最近采集的样本.
    - bytesPerUpdate: Double 平均每次更新集合的字节数,这里的更新主要是指插入新元素或者修改新元素.
    - numUpdates: Long 从初始化样本或者重置样本开始,更新集合的总次数
    - nextSampleNum: Long 达到下次采集样本时对集合的操作总次数
 - SizeTracker重要的方法:
	- resetSamples() 初始化或者重置样本采集评判标准,将numUpdates nextSampleNum 的值都初始化为1,同时对集合当前状态进行一次采样.
	- afterUpdate() 对集合进行更新操作以后,会调用该方法,判断是不是需要进行下一次采样.
	- takeSample() 样本采集,估算当前集合的大小,并将集合的大小和当前更新集合操作的总次数封装到Sample样本类中,放入到样本队列,放入后需要判断当前样本队列中样本数量,如果>2,需要将最先加入到队列中的样本移除队列,以保证样本队列中最多只保存最近采集的两个样本.代码:
		```scala
			private def takeSample(): Unit = {
			// 对集合当前的状态进行大小估算,并将估算的大小以及当前对集合操作的总次数封装到样本对象Sample中并加样本添加到样本队列
			samples.enqueue(Sample(SizeEstimator.estimate(this), numUpdates))
			// 保证样本队列存的最多是最近采集的两个样本
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
   - WritablePartitionedPairCollection 接口定义了将带有分区信息的key-value键值对插入到集合,并提供了获取可以将集合的内容按照字节写入磁盘的WritablePartitionedIterator迭代器.在其伴生类中,还提供了根据指定的key比较器获取默认的根据分区ID进行比较,或者既根据分区ID进行比较,也根据key-value键值对的key进行比较的比较器.
   - WritablePartitionedPairCollection的结构图:
     <img src='https://www.github.com/lijiayan2015/cangku/raw/master/小书匠/1547687864602.png' width=700 height=300>
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
				 // 实现了将元素按照字节写入到磁盘的方法,这里写只是写元素的key和value,并没有写分区信息
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
	 
	