# 数据质量模块问题汇总

v3.2.0

1. User class threw exception: java.lang.ClassNotFoundException: com.mysql.cj.jdbc.Driver

   ```log
   User class threw exception: java.lang.ClassNotFoundException: com.mysql.cj.jdbc.Driver
       at java.net.URLClassLoader.findClass(URLClassLoader.java:387)
       at java.lang.ClassLoader.loadClass(ClassLoader.java:418)
       at java.lang.ClassLoader.loadClass(ClassLoader.java:351)
       at org.apache.spark.sql.execution.datasources.jdbc.DriverRegistry$.register(DriverRegistry.scala:46)
       at org.apache.spark.sql.execution.datasources.jdbc.JDBCOptions.$anonfun$driverClass$1(JDBCOptions.scala:103)
       at org.apache.spark.sql.execution.datasources.jdbc.JDBCOptions.$anonfun$driverClass$1$adapted(JDBCOptions.scala:103)
       at scala.Option.foreach(Option.scala:407)
       at org.apache.spark.sql.execution.datasources.jdbc.JDBCOptions.<init>(JDBCOptions.scala:103)
       at org.apache.spark.sql.execution.datasources.jdbc.JDBCOptions.<init>(JDBCOptions.scala:41)
       at org.apache.spark.sql.execution.datasources.jdbc.JdbcRelationProvider.createRelation(JdbcRelationProvider.scala:34)
       at org.apache.spark.sql.execution.datasources.DataSource.resolveRelation(DataSource.scala:346)
       at org.apache.spark.sql.DataFrameReader.loadV1Source(DataFrameReader.scala:229)
       at org.apache.spark.sql.DataFrameReader.$anonfun$load$2(DataFrameReader.scala:211)
       at scala.Option.getOrElse(Option.scala:189)
       at org.apache.spark.sql.DataFrameReader.load(DataFrameReader.scala:211)
       at org.apache.spark.sql.DataFrameReader.load(DataFrameReader.scala:172)
       at org.apache.dolphinscheduler.data.quality.flow.batch.reader.JdbcReader.read(JdbcReader.java:73)
       at org.
   ```

   根据官网描述，当前 dolphinscheduler-data-quality-3.x.x.jar 是瘦包，不包含任何 JDBC 驱动。 如果有 JDBC 驱动需要，可以在节点设置选项参数处设置 --jars 参数， 如：--jars /lib/jars/mysql-connector-java-8.0.16.jar。

   其他类找不到的问题类似。除此之外还可以直接将对应的包放入 `${SPARK_HOME}/jars` 目录下。

   如果只在海豚工作节点的客户端上放置JAR包，需要用client或者local模式启动任务。

2. 数据质量任务节点重跑的时候，可能会失败

   Exception in thread “main” org.apache.spark.sql.AnalysisException: path hdfs://MYCLUSTER/user/dolphinscheduler/data_quality_error_data/0_10_空值检测 already exists.;

   这是因为数据质量任务执行的最后会把结果写入到HDFS上，但是写入模式是使用ERROR，因此重跑的时候遇到相同的目录，自然报错了。把 BaseFileWriter 类中的写入模式 prepare() 方法中的 SAVE_MODE 改成 `overwrite`，即可解决。

3. 当数据源采用加密策略的时候，会导致数据质量任务连接数据源失败

   这是因为在源码中，并没有对密码部分进行解密操作。因此会报错：

   ```java
   vrite(Dataset<Row> data, SparkRuntimeEnvironment env) {
       ...
       data.write()
               .format(JDBC)
               ....
               .option(PASSWORD, ParserUtils.decode(config.getString(PASSWORD)))
               ....
   }
   ```

   因为这里涉及到了 common.properties，尝试了几次没有好的办法，只能用默认密钥加密才有效。因此以下方案适用于默认密钥，或者直接将自定义密钥替换源码中的默认密钥常量。

   添加pom依赖：

   ```xml
   <groupId>org.apache.dolphinscheduler</groupId>
   <artifactId>dolphinscheduler-datasource-api</artifactId>
   ```

   读取Config：

   ```java
   vrite(Dataset<Row> data, SparkRuntimeEnvironment env) {
       ...
       data.write()
               .format(JDBC)
               ....
               .option(PASSWORD, PasswordUtils.decodePassword(ParserUtils.decode(config.getString(PASSWORD))))
               ....
   }
   ```
4. Exception in thread "main" org.apache.hadoop.security.AccessControlException: Permission denied: user=test, access=WRITE, inode="/":dolphinscheduler:supergroup:drwxr-xr-x

   在运行某个Spark Application的时候，需要向Hdfs写入文件，控制台会输出-访问HDFS报错：org.apache.hadoop.security.AccessControlException: Permission denied

   从中很容易看出是因为当前执行Spark Application的用户没有Hdfs“/”目录的写入权限。

   需要进行环境配置：

   - /worker-server/conf/dolphinscheduler_env.sh
     - SPARK_HOME2 ：配置spark安装目录
     - HADOOP_USER_NAME：增加该变量，填写hadoop集群的部署用户，
       - hadoop在访问hdfs的时候会进行权限认证，取用户名的过程是这样的：读取HADOOP_USER_NAME系统环境变量，如果不为空，那么拿它作username，如果为空；读取HADOOP_USER_NAME这个java环境变量，如果为空；从com.sun.security.auth.NTUserPrincipal或者com.sun.security.auth.UnixPrincipal的实例获取username；如果以上尝试都失败，那么抛出异常LoginException("Can’t find user name")
   - /worker-server/conf/common.properties
     - data-quality.jar.name=dolphinscheduler-data-quality-3.0.1-SNAPSHOT.jar: 保持jar包名称和编译后的名称一致，默认为dolphinscheduler-data-quality-dev-SNAPSHOT.jar
     - 给执行租户赋权，添加hadoop部署用户组，这里租户test，hadoop部署用户组为dolphinscheduler?/supergroup
       - `sudo usermod -a -G dolphinscheduler test`

   Hdfs的用户权限是与本地文件系统的用户权限绑定在一起的，根据错误提示，可以发现，Hdfs中的/目录是属于supergroup组里的dolphinscheduler用户的。

   如果是Linux环境，将执行操作的用户添加到supergroup用户组。

   ```bash
   groupadd supergroup
   usermod -a -G supergroup test
   ```

   这样，以后每次执行类似操作可以将文件写入Hdfs中属于test用户的目录内，而不会出现上面的Exception。

   编码方案：动态添加HADOOP_USER_NAME？

   ```java
   import org.apache.hadoop.conf.Configuration;
   import org.apache.hadoop.fs.FileSystem;
   import org.apache.hadoop.fs.Path;
   import java.util.Properties;

   public class TestHDFS {
       public static void main(String[] args) throws Exception{
           Properties properties = System.getProperties();
           properties.setProperty("HADOOP_USER_NAME", "root");

           Configuration conf = new Configuration();
           conf.set("fs.defaultFS", "hdfs://192.168.0.104:9000");
           FileSystem fs = FileSystem.get(conf);

           // 存在的情况下会覆盖之前的目录
           boolean success = fs.mkdirs(new Path("/xiaol"));
           System.out.println(success);
       }
   }
   ```

