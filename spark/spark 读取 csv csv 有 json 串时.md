###  Spark 读取 csv 时,当 csv 的字段值中有 JSON 串 

需求:统计 csv 中 有 json 串的 key 个数

csv 数据:
![csv](https://www.github.com/lijiayan2015/cangku/raw/master/小书匠/1555295890333.png)

代码:
``` scala
package com.rm1024.scala

import com.alibaba.fastjson.JSON
import org.apache.spark.sql.SparkSession

import scala.collection.mutable.ArrayBuffer

object JsonParserTest {
  def main(args: Array[String]): Unit = {
    Logger.getLogger("org").setLevel(Level.ERROR)
    val spark = SparkSession.builder()
      .appName(this.getClass.getSimpleName)
      .master("local[*]")
      .getOrCreate()
    val path = "/Users/lijiayan/Desktop/bad7b804-6de6-4aa0-ba74-e7f485fbf41c.csv"
   
    spark.read.format("com.databricks.spark.csv")
      .option("inferSchema", value = false)
      .option("header", value = true)
      .option("nullValue", "\\N")
      .option("escape", "\"") // 设置用于在已引用的值内转义引号的单个字符。详情见 spark 读取 csv 官网介绍 https://spark.apache.org/docs/2.0.2/api/java/org/apache/spark/sql/DataFrameReader.html#option(java.lang.String,%20boolean)
      .option("quoteAll", "true")
      .option("sep", ",")
      .csv(path).createTempView("test")
    
    val sql =
      """
        |select * from test
      """.stripMargin

    spark.sql(sql).rdd.map(row => {
      val jsonStr = row.getAs[String]("stat_json")
      val jObj = JSON.parseObject(jsonStr)
      val list = ArrayBuffer[String]()
      val it = jObj.entrySet().iterator()
      while (it.hasNext) {
        val key: String = it.next().getKey
        list.append(key)
      }
      list
    }).flatMap(it => it.map((_, 1)))
      .reduceByKey(_ + _)
      .foreach(println)
    
    spark.stop()
  }
}

```

输出结果:
(kqid,11)
(event,100)
(stime,60)
(start,100)
(buynum,46)
(pn,8)
(source,32)
(type,100)
(skuid,46)
(price,46)
(code,21)
(end,100)
(filmid,53)
(kq,6)
(url,49)
(stoptime,60)
(binded,1)
(id,60)
(eventType,3)
(browser,17)
(eventName,3)
(ltime,60)
(receive,2)
(couponId,1)

