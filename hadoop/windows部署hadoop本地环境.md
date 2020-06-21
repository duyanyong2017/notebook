## windows10环境下安装hadoop 3.2.1
1. 安装JDK1.8，并配置环境变量。注意：JAVA_HOME环境变量配置的路径不要包含空格（C盘中的Program Files 可以使用PROGRA~1 代替，D盘路径经测试不行）

2. 下载Hadoop安装文件 https://mirror.bit.edu.cn/apache/hadoop/common/hadoop-3.2.1/hadoop-3.2.1.tar.gz

3. 解压文件至

4.配置HADOOP_HOME环境变量，并在系统环境变量（Path）中添加Hadoop环境变量

5.打开cmd窗口，输入hadoop version命令验证
  备注：若出现 Error: JAVA_HOME is incorrectly set. Please update F:\hadoop\conf\hadoop-env.cmd的报错，则是因为JAVA_HOME环境变量配置的路径含有空格的原因

6.Hadoop伪分布式部署配置
  下载windows专用二进制文件和工具类依赖库: hadoop在windows上运行需要winutils支持和hadoop.dll等文件
  链接: https://pan.baidu.com/s/1NuuNR0TvBCrIcCFMbSPUAA 提取码: 39a9
  注意:  hadoop.dll等文件不要与hadoop冲突，若出现依赖性错误可以将hadoop.dll放到C:\Windows\System32下一份

7.修改etc目录下的core-site.xml文件 

   <configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>/E:/tools/hadoop-3.1.2/hadoop-3.1.2/data/dfs/namenode</value>
    </property>
    <property>
      <name>dfs.datanode.data.dir</name>
      <value>/E:/tools/hadoop-3.1.2/hadoop-3.1.2/data/dfs/datanode</value>
    </property>
  </configuration>

  注意：windows目录路径要改成使用正斜杠，且磁盘名称最前面也需要一个正斜杠

8.修改hdfs-site.xml配置文件
  <configuration>
   <property>
    <name>hadoop.tmp.dir</name>
    <value>/E:/tools/hadoop-3.1.2/hadoop-3.1.2/data</value>
    <description>存放临时数据的目录，即包括NameNode的数据</description>
    </property>
   <property>
    <name>fs.defaultFS</name>
    <value>hdfs://localhost:9000</value>
   </property>
  </configuration>
    注意：windows目录路径要改成使用正斜杠，且磁盘名称最前面也需要一个正斜杠

9.节点格式化
  在cmd窗口执行命令：hdfs namenode -format

10.启动&关闭Hadoop
     a.进入Hadoop的sbin目录下执行start-dfs.cmd启动Hadoop     
     b.Web界面查看HDFS信息，在浏览器输入http://localhost:9870/，可访问NameNode
     备注：hadoop 2.7.7访问http://localhost:50070 ，可访问NameNode，http://localhost:8088 访问yarn的web界面，有就表明已经成功。可通过jps - 查看运行的所有结点
     c.stop-all 停止运行的所有结点
