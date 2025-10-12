# 切换元数据库

1. 流程概述

   在将dolphinscheduler的元数据库从默认的H2切换为MySQL时，需要以下步骤：

   操作
   1. 配置MySQL数据库
   2. 修改dolphinscheduler的配置文件
   3. 初始化MySQL数据库
   4. 启动dolphinscheduler

2. 具体步骤及操作

   1. 配置MySQL数据库

      首先，我们需要在MySQL数据库中创建一个新的数据库以存储dolphinscheduler的元数据信息。

   2. 修改dolphinscheduler的配置文件

      找到dolphinscheduler的配置文件application-datasources.properties，修改其中的配置信息，指向MySQL数据库的相关信息。

      ```properties
      # MySQL配置
      spring.datasource.url=jdbc:mysql://localhost:3306/dolphinscheduler?useUnicode=true&characterEncoding=UTF-8&useSSL=false&serverTimezone=UTC
      spring.datasource.username=root
      spring.datasource.password=root
      spring.datasource.driver-class-name=com.mysql.jdbc.Driver
      ```
   3. 初始化MySQL数据库

      接下来，我们需要初始化MySQL数据库，执行以下命令来创建表结构：

      ```sql
      mysql -uroot -proot dolphinscheduler < /path/to/dolphinscheduler/sql/dolphinscheduler.sql
      ```
   4. 启动dolphinscheduler

      最后，重新启动dolphinscheduler服务，使配置生效：

      ```bash
      sh start.sh
      ```

