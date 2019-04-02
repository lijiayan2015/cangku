### Spark 中动态的给Row新增字段

我们知道,在Spark中,我们读取csv或者MySQL等关系型数据库时,可以直接得到DataFrame.我们要想新增一个字段,可以通过DataFrame的API或者注册一个临时表,通过SQL语句能很方便的实现给增加一个或多个字段.

但是,当我们将DataFrame转化成RDD的时候,RDD里面的类型就是Row,如果此时,要想再增加一个字段,该怎么办呢?

### Show Time
```scala
package com.emmm.test.scala

import com.yss.remake.util.BasicUtils
import org.apache.spark.SparkConf
import org.apache.spark.sql.catalyst.expressions.GenericRowWithSchema
import org.apache.spark.sql.types.{StringType, StructType}
import org.apache.spark.sql.{Row, SparkSession}

object Emmm {
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf()
    conf.setMaster(BasicUtils.masterType)
    conf.setAppName(this.getClass.getSimpleName)
    conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
    conf.set("spark.kryo.registrationRequired", "true")
    conf.registerKryoClasses(Array(
      Class.forName("scala.collection.mutable.WrappedArray$ofRef"),
      Class.forName("org.apache.spark.sql.types.StringType$"),
      classOf[TPerson],
      classOf[org.apache.spark.sql.catalyst.expressions.GenericRowWithSchema],
      classOf[org.apache.spark.sql.types.StructType],
      classOf[org.apache.spark.sql.types.StructField],
      classOf[org.apache.spark.sql.types.Metadata],
      classOf[Array[TPerson]],
      classOf[Array[org.apache.spark.sql.Row]],
      classOf[Array[org.apache.spark.sql.types.StructField]],
      classOf[Array[Object]]
    ))
    val spark = SparkSession.builder()
      .config(conf)
      .getOrCreate()
    import spark.implicits._
    // 使用样例类创建RDD并转化成DF后又回到RDD
    spark.sparkContext.parallelize(Seq(TPerson("zs", "21"), TPerson("ls", "25"))).toDF().rdd
      .map(row => {
        // 打印schema
        println(row.schema)
        // 得到Row中的数据并往其中添加我们要新增的字段值
        val buffer = Row.unapplySeq(row).get.map(_.asInstanceOf[String]).toBuffer
        buffer.append("男") //增加一个性别
        buffer.append("北京") //增肌一个地址

        // 获取原来row中的schema,并在原来Row中的Schema上增加我们要增加的字段名以及类型.
        val schema: StructType = row.schema
          .add("gender", StringType)
          .add("address", StringType)
        // 使用Row的子类GenericRowWithSchema创建新的Row
        val newRow: Row = new GenericRowWithSchema(buffer.toArray, schema)
        // 使用新的Row替换成原来的Row
        newRow
      }).map(row => {
      // 打印新的schema
      println(row.schema)
      // 测试我们新增的字段
      val gender = row.getAs[String]("gender")
      // 获取原本就有的字段
      val name = row.getAs[String]("name")
      val age = row.getAs[String]("age")
      // 获取新的字段
      val address = row.getAs[String]("address")
      // 输出查看结果
      println(s"$name-$age-$gender-$address")
      row
    }).collect()
    spark.stop()
  }

  /**
    * 样例类
    *
    * @param name name属性
    * @param age  age属性
    */
  case class TPerson(name: String, age: String)

}
```

### TODO
PS:不要问为什么生成RDD又转化成DataFrame又转化成RDD,因为确实在实际中用到了Row新增字段的需求,这么转只是为了测试.

最后,免费赠送**kyro**的使用.