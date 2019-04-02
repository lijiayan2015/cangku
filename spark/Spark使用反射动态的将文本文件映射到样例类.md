### Spark使用反射动态的将文本数据映射到样例类
假如现在有一个tsv或者csv文件,文件中每条数据包含100+个字段.使用Spark读取这个文件.我看有些人的做法是直接创建一个类,然后类的字段一个一个的传.wdmy.要是有100多个字段,这不是很耗时?好吧,暂且不说耗时不好时,万一一个不小心,写错了一个字段,那该怎么办?反正我比较喜欢偷懒,像这种的情况,一般使用偷奸耍滑的方法.

当然,使用反射的前提是:
1. 不考虑反射时带来的性能消耗.
2. csv中字段数最好小于等于样例类中的属性个数.等于的不用管,小于的添加几个空的占位字段就好.
3. 样例类中字段的顺序与csv中字段的顺序一一对应.


### ShowTime
```scala
package com.rm1024

import java.lang.reflect.Constructor

import org.apache.spark.sql.{DataFrame, SparkSession}

object ReflectReadCsv {
  // 反射获取类的构造方法
  val cnstructor: Constructor[_] = classOf[Bean].getConstructors()(0)

  def main(args: Array[String]): Unit = {
    val spark = SparkSession.builder()
      .master("local[*]")
      .appName(this.getClass.getSimpleName)
      .getOrCreate()
    val csvPath = "path"
    readCSV(spark, csvPath).rdd.map(row => {
      // row中的字段,给他转化成Seq
      var fields: Seq[String] = row.toSeq.map(_.asInstanceOf[String])
      // 假设样例类的属性与csv的字段一一对应
      // 笨方法
      Bean(fields(0), fields(1), fields(2), fields(3), fields(4), fields(5), fields(6), fields(7), fields(8), fields(9),
        fields(10), fields(11), fields(12), fields(13), fields(14), fields(15), fields(16), fields(17), fields(18), fields(19),
        fields(20), fields(21), fields(22), fields(23), fields(24), fields(25), fields(26), fields(27), fields(28), fields(29),
        fields(30), fields(31), fields(32), fields(33), fields(34),
      )

      // 反射创建
      cnstructor.newInstance(fields: _*)

      // 假设scv的字段没有Bean的字段多,怎么办?补充几个,使之相等.
      fields = fields ++ Seq("", "", "", "", "", "")
      // 然后再反射创建
      cnstructor.newInstance(fields: _*)
    })
    spark.stop()
  }


  /**
    * 读取csv格式的数据
    *
    * @return
    * @param path   csv路径
    * @param spark  SparkSession
    * @param header 有些csv是有头文件的,有些没有,看是否需要头文件,true 表示需要头文件,会将头文件读取进来,
    *               false表示不需要头文件,则会吧第一行头文件去掉.
    * @param sep    分隔符 不管是csv还是其他文件,字段之间的分隔符
    * @return
    */
  def readCSV(spark: SparkSession, path: String, header: Boolean = true, sep: String = "\t"): DataFrame = {
    spark.read.format("csv")
      .option("sep", sep)
      .option("inferSchema", "false") // 去掉类型推断,都使用String类型
      .option("header", header)
      .load(path)
  }

}

case class Bean(
                 field0: String,
                 field1: String,
                 field2: String,
                 field3: String,
                 field4: String,
                 field5: String,
                 field6: String,
                 field7: String,
                 field8: String,
                 field9: String,
                 field10: String,
                 field11: String,
                 field12: String,
                 field13: String,
                 field14: String,
                 field15: String,
                 field16: String,
                 field17: String,
                 field18: String,
                 field19: String,
                 field20: String,
                 field21: String,
                 field22: String,
                 field23: String,
                 field24: String,
                 field25: String,
                 field26: String,
                 field27: String,
                 field28: String,
                 field29: String,
                 field30: String,
                 field31: String,
                 field32: String,
                 field33: String,
                 field34: String
               )

```

### The End
可能有很多追求性能的大神们不喜欢我这种偷奸耍滑的程序员,但是我觉得代码都好看多了,只是稍微付出那么一点点性能,不好吗?

免费赠送spark读取文件时怎么去掉第一行的解决方案.