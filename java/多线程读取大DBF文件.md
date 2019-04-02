### Java多线程读取大文件

#### 需求
需要将DBF文件解析后存储到HBase 或者HDFS.起初打算使用Kettle读取,然后转存到HBase,小文件还好,一下子就ok来,但是,遇到一个1G大小(测试阶段,实际生产远远大于1G)的时候,Kettle输出到HBase时实在太慢,可能由于HBase的技术水平有限,再怎么优化,还是很慢.于是想着自己写一个程序解决一下,结果还是和kettle的差不多,就有点尴尬(手动尬笑).不管怎么样,还是把实现逻辑贴出来吧.

#### DBF文件
DBF是一种特殊的文件格式.主要由头部和文件体构成.其中头部包含整个文件的描述信息.主要包含:
  - 文件头的长度
  - 记录数(类似mysql的一条一条记录)
  - 每条记录的长度
  - 每条记录包含的字段
  - 每个字段的长度
  - 每个字段的类型
  - 等等

文件体主要由一条一条的记录组成.

|   字段1  |    字段2 |  字段3   |   字段4  |  字段5   |   字段6  |
| --- | --- | --- | --- | --- | --- |
|   字段1对应的值(10字节)  |  字段2对应的值(10字节)    |  字段3对应的值(20字节)    |  字段4对应的值(10字节)    |  字段6对应的值(10字节)    |  字段7对应的值(15字节)    |
|   字段1对应的值(10字节)  |  字段2对应的值(10字节)    |  字段3对应的值(20字节)    |  字段4对应的值(10字节)    |  字段6对应的值(10字节)    |  字段7对应的值(15字节)    |
|   字段1对应的值(10字节)  |  字段2对应的值(10字节)    |  字段3对应的值(20字节)    |  字段4对应的值(10字节)    |  字段6对应的值(10字节)    |  字段7对应的值(15字节)    |

大概就是上面这个样子的.

由于比较规整,所以解析起来还算比较方便.

#### 多线程读取DBF文件
当一个DBF文件很大的时候,使用普通的单线程读取DBF文件的时候,会很耗时,所以我使用多线程并行读取dbf文件是个不错的选择.

实现步骤大概如下:
 1. 读取dbf的头文件,获取到上面列出来的头文件长度,记录数,字段名以及该字段对应的所占字节大小等信息.
 2. 计算每个线程要读取的记录数(一个线程读取多少行)
 3. 使用java nio包下的RandomAccessFile的FileChannel来负责每个线程的读取任务.
 4. 定一个输出接口,每解析一条记录,将读取后解析的一条记录对应的数据使用接口回调出去,方便处理(可以存储到HDFS,也可以存储为tsv文件,或者存储到HBase).

#### 读取头文件
```java
/**
     * 读取头文件信息
     *
     * @throws IOException
     */
    private void readHeader() throws IOException {
        DataInputStream dataInput = new DataInputStream(IOUtils.genFileInputStream(new File(file)));
        dbfHeader.read(dataInput, null, false);
        Charset charset = this.dbfHeader.getDetectedCharset();
        if (charset != null) {
            this.userCharset = charset;
        } else {
            this.userCharset = Charset.forName("GBK");
        }
        this.fields = dbfHeader.getFieldArray();
        // 头文件长度
        this.headerLength = dbfHeader.getHeaderLength();
        // 每条记录长度
        this.recordLength = dbfHeader.getRecordLength();
        // 记录数
        this.numberOfRecords = dbfHeader.numberOfRecords;
        dataInput.close();
    }

```

#### 读取DBF文件的Task
```java
private final class ReadTask implements Runnable {
        // 要读取的文件
        private String file;
        // 当前线程要读取文件的开始位置
        private long startPos;
        // 当前线程要读取的字节数
        private long readByteSize;
        // 每行的记录长度
        private int recordLength;

        private DBFFieldUtils[] fields;

        private Charset charset;

        private ReadListener listener;

        private Output output;

        private int num = 0;

        ReadTask(String file, long startPos, long readByteSize, int recordLength, DBFFieldUtils[] fields, Charset charset, ReadListener listener) {
            this.file = file;
            this.startPos = startPos;
            this.readByteSize = readByteSize;
            this.recordLength = recordLength;
            this.fields = fields;
            this.charset = charset;
            this.listener = listener;
        }

        public Output getOutput() {
            return output;
        }

        public void setOutput(Output output) {
            this.output = output;
        }

        @Override
        public void run() {
            this.read();
        }

        private void read() {
            RandomAccessFile randomAccessFile = null;
            FileChannel channel = null;
            try {
                randomAccessFile = new RandomAccessFile(file, "r");
                channel = randomAccessFile.getChannel();
                channel.position(startPos);
                // 结束位置
                long endPos = startPos + readByteSize;
                boolean isEnd = false;
                int buffLine = (int) (readByteSize / recordLength);
                buffLine = buffLine < Config.getReadLineSize() ? buffLine : Config.getReadLineSize();
                ByteBuffer buffer = ByteBuffer.allocate(this.recordLength * buffLine);
                while ((channel.read(buffer) != -1 && startPos < endPos) && !isEnd) {
                    byte[] bytes = new byte[buffer.position()];
                    buffer.flip();
                    buffer.get(bytes);
                    // 获取此次读取的行数
                    int readedLines = bytes.length / this.recordLength;
                    // 这里需要验证是不是文件的最后一行(文件的最后一行没有换行符,少一个字节)
                    if (readedLines * this.recordLength < bytes.length) {
                        isEnd = true;
                        readedLines += 1;
                    }
                    int start = 0;
                    for (int i = 0; i < readedLines; i++) {
                        // 获取每一行的数据
                        Field[] yssfields = new Field[fields.length];
                        for (int j = 0; j < fields.length; j++) {
                            byte[] fieldBytes = new byte[fields[j].getLength()];
                            System.arraycopy(bytes, start, fieldBytes, 0, fieldBytes.length);
                            String fieldContent = new String(fieldBytes, 0, fieldBytes.length, this.charset).trim();
                            start += fieldBytes.length;
                            Field yssField = new Field();
                            yssField.setFieldName(fields[j].getName());
                            yssField.setContent(fieldContent);
                            yssfields[j] = yssField;
                        }
                        start = start + 1;// 最后一个换行符的位置
                        if (this.output != null) {
                            output.write(yssfields, false);
                            num++;
                        }
                    }
                    this.startPos += bytes.length;
                    channel.position(startPos);
                    // 计算此次读完后剩下的字节数
                    // 此次读完后剩下的行数
                    buffLine = (int) ((readByteSize - recordLength * num) / recordLength);
                    buffLine = buffLine < Config.getReadLineSize() ? buffLine : Config.getReadLineSize();
                    buffer = ByteBuffer.allocate(this.recordLength * buffLine);
                }
            } catch (IOException e) {
                e.printStackTrace();

            } finally {
                IOUtils.close(randomAccessFile, channel);
                if (this.output != null) {
                    this.output.write(null, true);
                    this.output.onDestory();
                }
                if (this.listener != null) {
                    this.listener.finish(true, num);
                    System.out.println(Thread.currentThread() + "读了" + num + "行");
                }
            }
        }
    }
```

#### 根据事先设置的线程池大小,计算每个线程要读取的线程任务量
```java
/**
     * read start
     */
    public void start() {
        this.startTime = System.currentTimeMillis();
        try {
            readHeader();
            if (this.numberOfRecords <= Config.getReadLineSize()) {
                poolSize = 1;
            }
            // 计算每个线程应该读取的行数
            int readLines = this.numberOfRecords / poolSize;
            // 从头的位置开始读
            long start = headerLength + 1;

            // 已经读取的行数
            int line = 0;
            for (int i = 0; i < poolSize; i++) {
                // 每个线程读取的字节数 = 要读取的行数 * 每行的长度
                long readByteSize = readLines * this.recordLength;
                if (i < poolSize - 1) {
                    line += readLines;
                    ReadTask task = new ReadTask(file, start, readByteSize, this.recordLength, this.fields, userCharset, this);
                    Output output = genOutput();
                    if (output != null) {
                        task.setOutput(output);
                        output.onCreate();
                    }
                    this.threadPool.submit(task);
                    start = start + readByteSize;
                    System.out.println("线程" + i + "需要读" + readLines + "行");
                } else {
                    // 最后一个线程包揽剩下的所有数据
                    int lastLines = numberOfRecords - line;
                    readByteSize = lastLines * this.recordLength;
                    ReadTask task = new ReadTask(file, start, readByteSize, this.recordLength, this.fields, userCharset, this);
                    Output output = genOutput();
                    if (output != null) {
                        task.setOutput(output);
                        output.onCreate();
                    }
                    this.threadPool.submit(task);
                    System.out.println("线程" + i + "需要读" + lastLines + "行");
                }

            }

        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

#### 定义向外输出的接口
```java
@SuppressWarnings("all")
public abstract class Output {

    /**
     * 用于解析后向外输出每条记录
     * @param fields 每条记录包含的字段(字段名,字段值)
     * @param isFinish  是否处理已经处理完
     */
    public abstract void write(Field[] fields, boolean isFinish);

    public void onCreate() {
    }

    public void onDestory() {
    }
}
```

#### 完成的处理过程
```java
package com.yss.dbf;

import java.io.DataInputStream;
import java.io.File;
import java.io.IOException;
import java.io.RandomAccessFile;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.charset.Charset;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

@SuppressWarnings("all")
public class ReadDBF implements ReadListener {

    private DBFHeaderUtils dbfHeader;
    // 编码
    private Charset userCharset;

    // 头长度
    private int headerLength;

    // 记录数
    private int numberOfRecords;

    // 每条记录长度
    private int recordLength;

    private int poolSize = Config.getThreadPoolSize();

    private ExecutorService threadPool;


    private String file;

    private DBFFieldUtils[] fields;

    // 已经结束线程的数量
    private int finishCount = 0;

    private long startTime = 0;

    private int totalLines = 0;

    private ReadStatusCallBack readStatusCallBack;
    //执行成功的线程数
    private int successThreadCount = 0;

    public ReadDBF(String file) {
        this.dbfHeader = new DBFHeaderUtils();
        this.file = file;
        this.threadPool = Executors.newFixedThreadPool(this.poolSize);
    }

    private Output genOutput() {
        Output output = null;
        if (Config.getOutputClassName() == null) return null;
        try {
            Class<?> clz = Class.forName(Config.getOutputClassName());
            Object obj = clz.newInstance();
            if (obj instanceof Output) {
                output = ((Output) obj);
            } else {
                System.err.println("输出类需要实现com.yss.dbf.Output");
            }
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        }
        return output;
    }

    public void start(ReadStatusCallBack readStatusCallBack) {
        this.readStatusCallBack = readStatusCallBack;
        this.start();
    }

    /**
     * read start
     */
    public void start() {
        this.startTime = System.currentTimeMillis();
        try {
            readHeader();
            if (this.numberOfRecords <= Config.getReadLineSize()) {
                poolSize = 1;
            }
            // 计算每个线程应该读取的行数
            int readLines = this.numberOfRecords / poolSize;
            // 从头的位置开始读
            long start = headerLength + 1;

            // 已经读取的行数
            int line = 0;
            for (int i = 0; i < poolSize; i++) {
                // 每个线程读取的字节数 = 要读取的行数 * 每行的长度
                long readByteSize = readLines * this.recordLength;
                if (i < poolSize - 1) {
                    line += readLines;
                    ReadTask task = new ReadTask(file, start, readByteSize, this.recordLength, this.fields, userCharset, this);
                    Output output = genOutput();
                    if (output != null) {
                        task.setOutput(output);
                        output.onCreate();
                    }
                    this.threadPool.submit(task);
                    start = start + readByteSize;
                    System.out.println("线程" + i + "需要读" + readLines + "行");
                } else {
                    // 最后一个线程包揽剩下的所有数据
                    int lastLines = numberOfRecords - line;
                    readByteSize = lastLines * this.recordLength;
                    ReadTask task = new ReadTask(file, start, readByteSize, this.recordLength, this.fields, userCharset, this);
                    Output output = genOutput();
                    if (output != null) {
                        task.setOutput(output);
                        output.onCreate();
                    }
                    this.threadPool.submit(task);
                    System.out.println("线程" + i + "需要读" + lastLines + "行");
                }

            }

        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 读取头文件信息
     *
     * @throws IOException
     */
    private void readHeader() throws IOException {
        DataInputStream dataInput = new DataInputStream(IOUtils.genFileInputStream(new File(file)));
        dbfHeader.read(dataInput, null, false);
        Charset charset = this.dbfHeader.getDetectedCharset();
        if (charset != null) {
            this.userCharset = charset;
        } else {
            this.userCharset = Charset.forName("GBK");
        }
        this.fields = dbfHeader.getFieldArray();
        // 头文件长度
        this.headerLength = dbfHeader.getHeaderLength();
        // 每条记录长度
        this.recordLength = dbfHeader.getRecordLength();
        // 记录数
        this.numberOfRecords = dbfHeader.numberOfRecords;
        dataInput.close();
    }

    @Override
    public void finish(boolean success, int readLines) {
        this.finishCount++;
        totalLines += readLines;
        this.successThreadCount = success ? (++this.successThreadCount) : this.successThreadCount;
        if (this.finishCount == poolSize) {
            this.threadPool.shutdown();
            System.out.println("共花费:" + ((System.currentTimeMillis() - startTime) * 1.0 / 1000) + "秒,总计:" + totalLines + "行");
            if (readStatusCallBack != null) {
                readStatusCallBack.finish(this.successThreadCount == this.poolSize);
            }
        }

    }


    private final class ReadTask implements Runnable {
        // 要读取的文件
        private String file;
        // 当前线程要读取文件的开始位置
        private long startPos;
        // 当前线程要读取的字节数
        private long readByteSize;
        // 每行的记录长度
        private int recordLength;

        private DBFFieldUtils[] fields;

        private Charset charset;

        private ReadListener listener;

        private Output output;

        private int num = 0;

        ReadTask(String file, long startPos, long readByteSize, int recordLength, DBFFieldUtils[] fields, Charset charset, ReadListener listener) {
            this.file = file;
            this.startPos = startPos;
            this.readByteSize = readByteSize;
            this.recordLength = recordLength;
            this.fields = fields;
            this.charset = charset;
            this.listener = listener;
        }

        public Output getOutput() {
            return output;
        }

        public void setOutput(Output output) {
            this.output = output;
        }

        @Override
        public void run() {
            this.read();
        }

        private void read() {
            RandomAccessFile randomAccessFile = null;
            FileChannel channel = null;
            try {
                randomAccessFile = new RandomAccessFile(file, "r");
                channel = randomAccessFile.getChannel();
                channel.position(startPos);
                // 结束位置
                long endPos = startPos + readByteSize;
                boolean isEnd = false;
                int buffLine = (int) (readByteSize / recordLength);
                buffLine = buffLine < Config.getReadLineSize() ? buffLine : Config.getReadLineSize();
                ByteBuffer buffer = ByteBuffer.allocate(this.recordLength * buffLine);
                while ((channel.read(buffer) != -1 && startPos < endPos) && !isEnd) {
                    byte[] bytes = new byte[buffer.position()];
                    buffer.flip();
                    buffer.get(bytes);
                    // 获取此次读取的行数
                    int readedLines = bytes.length / this.recordLength;
                    // 这里需要验证是不是文件的最后一行(文件的最后一行没有换行符,少一个字节)
                    if (readedLines * this.recordLength < bytes.length) {
                        isEnd = true;
                        readedLines += 1;
                    }
                    int start = 0;
                    for (int i = 0; i < readedLines; i++) {
                        // 获取每一行的数据
                        Field[] yssfields = new Field[fields.length];
                        for (int j = 0; j < fields.length; j++) {
                            byte[] fieldBytes = new byte[fields[j].getLength()];
                            System.arraycopy(bytes, start, fieldBytes, 0, fieldBytes.length);
                            String fieldContent = new String(fieldBytes, 0, fieldBytes.length, this.charset).trim();
                            start += fieldBytes.length;
                            Field yssField = new Field();
                            yssField.setFieldName(fields[j].getName());
                            yssField.setContent(fieldContent);
                            yssfields[j] = yssField;
                        }
                        start = start + 1;// 最后一个换行符的位置
                        if (this.output != null) {
                            output.write(yssfields, false);
                            num++;
                        }
                    }
                    this.startPos += bytes.length;
                    channel.position(startPos);
                    // 计算此次读完后剩下的字节数
                    // 此次读完后剩下的行数
                    buffLine = (int) ((readByteSize - recordLength * num) / recordLength);
                    buffLine = buffLine < Config.getReadLineSize() ? buffLine : Config.getReadLineSize();
                    buffer = ByteBuffer.allocate(this.recordLength * buffLine);
                }
            } catch (IOException e) {
                e.printStackTrace();

            } finally {
                IOUtils.close(randomAccessFile, channel);
                if (this.output != null) {
                    this.output.write(null, true);
                    this.output.onDestory();
                }
                if (this.listener != null) {
                    this.listener.finish(true, num);
                    System.out.println(Thread.currentThread() + "读了" + num + "行");
                }
            }
        }
    }

}
```

我使用的是通过配置文件设置线程池的大小,一次读取多少行记录,输出接口的实现类等.
配置文件如下:
```
#输出文件分隔符
output.fields.delimited=#
#一次读取多少行
read.line.size=100
#线程池数量
thread.pool.size=7
#输出类
output.class.name=com.yss.dbf.out.HBaseOutput
#往HBase里面批量提交的数量
hbase.output.batch=50000
#读取dbf文件并输出成文本文件,供mr转换成HFile的路径
dbf.textfile.output.path=
#将dbf.textfile.output.path的文件上传到hdfs
dbf.textfile.hdfs.tmp.path=
#将转化成hfile文件的hdfs路径
hfile.hdfs.path=
#==========================================
#HBase配置
hbase.zookeeper.property.clientPort=2181
#Zookeeper 集群的地址列表，用逗号分割 如:vhost1,vhost1,vhost3 其中vhost1为zk所在的主机地址
hbase.zookeeper.quorum=vhost1,vhost1,vhost3
#HBase在ZK中的节点路径
zookeeper.znode.parent=/hbase-unsecure
hbase.table.name=ktr
hbase.table.family=cf
```

#### 输出到HBase的输出类.
```java
package com.yss.dbf.out;

import com.yss.dbf.Config;
import com.yss.dbf.Field;
import com.yss.dbf.IOUtils;
import com.yss.dbf.Output;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.*;
import org.apache.hadoop.hbase.util.Bytes;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.Executors;

@SuppressWarnings("all")
public class HBaseOutput1 extends Output {

    private Connection conn;
    private final int BATCHSIZE = Config.getHbaseOutputBatch();
    private BufferedMutator mutator;
    private List<Put> list;

    @Override
    public void onCreate() {
        try {
            conn = ConnectionFactory.createConnection(Config.getConf());
            BufferedMutatorParams params = new BufferedMutatorParams(TableName.valueOf(Config.getHbaseTableName()));
            params.writeBufferSize(1024 * 1024 * 10);
            params.pool(Executors.newFixedThreadPool(2));
            mutator = conn.getBufferedMutator(params);
            list = new ArrayList<>();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void onDestory() {
        IOUtils.close(conn, mutator);
    }

    @Override
    public void write(Field[] fields, boolean isFinish) {
        if (!isFinish) {
            if (fields.length > 4) {
                String rowKey = genRowKey(fields[3].getContent(),
                        fields[0].getContent(),
                        fields[1].getContent(),
                        fields[2].getContent());
                Put put = new Put(Bytes.toBytes(rowKey));
                for (int i = 4; i < fields.length; i++) {
                    put.addColumn(Bytes.toBytes(Config.getHbaseTableFamily()),
                            Bytes.toBytes(fields[i].getFieldName()),
                            Bytes.toBytes(fields[i].getContent()));
                }
                this.list.add(put);

                if (this.list.size() == BATCHSIZE) {
                    commit();
                }
            }
        } else {
            commit();
            System.out.println(Thread.currentThread().getName() + "完成!");
        }
    }

    private void commit() {
        try {
            mutator.mutate(list);
            mutator.flush();
            this.list.clear();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private String genRowKey(String... fields) {
        StringBuilder sb = new StringBuilder();
        for (String field : fields) {
            sb.append(field)
                    .append(Config.getOutPutDelimited());
        }
        sb.delete(sb.length() - 1, sb.length());
        return sb.toString();
    }

}

```

在输出到HBase的类型,由于我只针对特定的文件,做了一个简单的rowKey设计.当然也可以定一个rowKey的设计接口.比如:
```java
package com.yss.dbf.out;

import com.yss.dbf.Field;

public interface HBaseRowKey {
    String genRowKey(Field[] fields);
}
```

以上就是大概的整个DBF文件解析,并存储到HBase的过程.由于直接存储到HBase,还是太慢,所以我想着就是使用MapReduce生成HFile,然后使用命令加载将HFile加载到HBase,这样按理来说,应该会快很多.
#### MR之Map端代码如下:
```java
package com.yss.mr;

import com.yss.dbf.Config;
import org.apache.hadoop.hbase.client.Put;
import org.apache.hadoop.hbase.io.ImmutableBytesWritable;
import org.apache.hadoop.hbase.util.Bytes;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;

import java.io.IOException;

@SuppressWarnings("all")
public class WriteToHFile extends Mapper<LongWritable, Text, ImmutableBytesWritable, Put> {

    private String[] colums;

    @Override
    protected void setup(Context context) throws IOException, InterruptedException {
        colums = context.getConfiguration().get("colums.name").split(Config.getOutPutDelimited());
    }

    @Override
    protected void map(LongWritable key, Text value, Context context) throws IOException, InterruptedException {
        String[] fields = value.toString().split(Config.getOutPutDelimited());
        String ROWKEY = genRowKey(fields[3], fields[0], fields[1], fields[2]);
        Put put = new Put(Bytes.toBytes(ROWKEY));
        for (int i = 4; i < colums.length; i++) {
            put.addColumn(Bytes.toBytes("cf"), Bytes.toBytes(colums[i]), Bytes.toBytes(fields[i]));
        }
        ImmutableBytesWritable rowkey = new ImmutableBytesWritable(Bytes.toBytes(ROWKEY));
        context.write(rowkey, put);
    }

    private String genRowKey(String... fields) {
        StringBuilder sb = new StringBuilder();
        for (String field : fields) {
            sb.append(field)
                    .append(Config.getOutPutDelimited());
        }
        sb.delete(sb.length() - 1, sb.length());
        return sb.toString();
    }
}

```

#### MR之Driver端代码:
```java
package com.yss.mr;

import com.yss.dbf.Config;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.hbase.TableName;
import org.apache.hadoop.hbase.client.*;
import org.apache.hadoop.hbase.io.ImmutableBytesWritable;
import org.apache.hadoop.hbase.mapreduce.HFileOutputFormat2;
import org.apache.hadoop.hbase.mapreduce.LoadIncrementalHFiles;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.util.Tool;
import org.apache.hadoop.util.ToolRunner;

@SuppressWarnings("all")
public class Driver implements Tool {
    private final static String INPUT_PATH = "file:///Users/lijiayan/Desktop/tempDir/";
    private final static String OUTPUT_PATH = "hdfs://host:port/tmp/ljy/out/";

    private Configuration conf;

    @Override
    public int run(String[] strings) throws Exception {
        Connection connection = ConnectionFactory.createConnection(conf);
        Table table = connection.getTable(TableName.valueOf("ktr"));
        Job job = Job.getInstance(conf);
        job.setJarByClass(Driver.class);
        job.setMapperClass(WriteToHFile.class);
        job.setMapOutputKeyClass(ImmutableBytesWritable.class);
        job.setMapOutputValueClass(Put.class);
        job.setOutputFormatClass(HFileOutputFormat2.class);
        HFileOutputFormat2.configureIncrementalLoad(job, table, connection.getRegionLocator(TableName.valueOf("ktr")));
        FileInputFormat.addInputPath(job, new Path(INPUT_PATH));
        FileOutputFormat.setOutputPath(job, new Path(OUTPUT_PATH));
        return job.waitForCompletion(true) ? 0 : 1;
    }

    @Override
    public void setConf(Configuration configuration) {
        configuration.set("colums.name", genColums(
                "GDDM",
                "GDXM",
                "BCRQ",
                "CJBH",
                "GSDM",
                "CJSL",
                "BCYE",
                "ZQDM",
                "SBSJ",
                "CJSJ",
                "CJJG",
                "CJJE",
                "SQBH",
                "BS",
                "MJBH"
        ));
        configuration.set("hbase.zookeeper.property.clientPort", "2181");
        configuration.set("hbase.zookeeper.quorum", "ip");
        configuration.set("zookeeper.znode.parent", "/hbase-unsecure");
        configuration.set("hbase.fs.tmp.dir","hdfs://ip:port/tmp/ljy/temp/");
        this.conf = configuration;
    }

    @Override
    public Configuration getConf() {
        return conf;
    }

    private String genColums(String... fields) {
        StringBuilder sb = new StringBuilder();
        for (String field : fields) {
            sb.append(field)
                    .append(Config.getOutPutDelimited());
        }
        sb.delete(sb.length() - 1, sb.length());
        return sb.toString();
    }

    public static void main(String[] args) {
        try {
            Driver driver = new Driver();
            if (ToolRunner.run(driver, null) == 0) {//解析完成后,使用如下代码将HFile加载到HBase
                Connection connection = ConnectionFactory.createConnection(driver.conf);
                Admin admin = connection.getAdmin();
                Table table = connection.getTable(TableName.valueOf("ktr"));
                LoadIncrementalHFiles load = new LoadIncrementalHFiles(driver.conf);
                load.doBulkLoad(new Path(OUTPUT_PATH), admin, table, connection.getRegionLocator(TableName.valueOf("ktr")));
            } else {
                System.out.println("运行错误");
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}

```

#### End
好了,大概就是这样.主要代码注释都很清楚.由于结果不是太理想,所以这个过程还处于测试阶段.并没有上线使用.一方面记录多线程读取DBF文件的处理方式,当然通过这个方式,也可以拓展到多线程读取大文件的方式.处理逻辑大同小异.另一方便,记录一下写数据到HBase的方式.再一个就是通过MR生成HFile,然后加载HFile到HBase的方式.
若有借鉴者,不懂的地方,欢迎交流.
完整的代码已经上传[完整的代码](www.baidu.com).
后面会上传到github,有需要的也可以 lijiayan_mail@163.com