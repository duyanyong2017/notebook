前提：先部署jdk

# 1.下载kafka
版本选择地址：http://kafka.apache.org/releases.html 
选择需要下载的版本：使用wget 下载kafka_2.12-2.5.0.tgz /usr/local/software

# 2.解压
~~~bash
tar -xzvf /usr/local/software/kafka_2.12-2.5.0.tgz -C /usr/local/program
~~~

# 3.修改配置
配置kafka环境变量，在/etc/profile 中添加以下参数配置
~~~shell
export KAFKA_HOME=/usr/local/program/kafka_2.12-2.5.0
export PATH=$PATH:$KAFKA_HOME/bin
~~~
修改kafka配置config/server.properties，主要修改如下字段：
~~~shell
# The id of the broker. This must be set to a unique integer for each broker.
broker.id=1

# Switch to enable topic deletion or not, default value is false
delete.topic.enable=true

listeners=PLAINTEXT://10.17.156.73:9092

log.dirs=/usr/local/data/kafka/logs

zookeeper.connect=localhost:2181
~~~

# 4.启动
## 4.1 启动zookeeper，这里使用kafka自带zookeeper
切换到/usr/local/program/kafka_2.12-2.5.0 路径下，启动zookeeper
~~~shell
bin/zookeeper-server-start.sh config/zookeeper.properties
~~~

## 4.2 启动kafka
切换到/usr/local/program/kafka_2.12-2.5.0 路径下，启动kafka
常规模式启动kafka
~~~shell
bin/kafka-server-start.sh config/server.properties
~~~

进程守护模式启动kafka
~~~shell
nohup bin/kafka-server-start.sh config/server.properties >/dev/null 2>&1 &
~~~

~~~shell
#创建一个主题
./kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic ARF

#创建一个消费者
./kafka-console-consumer.sh --zookeeper localhost:2181 --topic ARF --from-beginning

#创建一个新消费者（支持0.9版本+）
./kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic ARF --new-consumer --from-beginning --consumer.config ../config/consumer.properties

#创建一个生产者
./kafka-console-producer.sh --broker-list localhost:9092 --topic ARF

#创建一个新生产者（支持0.9版本+）
./kafka-console-producer.sh --broker-list localhost:9092 --topic ARF --producer.config ../config/producer.properties
~~~
