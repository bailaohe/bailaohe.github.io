---
title: Use SparkSQL to build a OLAP database across different datasources
date: 2016-04-20 13:20:35
tags:
---

[Spark](http://spark.apache.com) is a large-scale data processing engine. [SparkSQL](https://spark.apache.org/docs/latest/sql-programming-guide.html), one of its important component, can access the Hive metastore service to handle Hive tables directly. Furthermore, SparkSQL also provides approach to use data from other external datasources (JDBC to RDB, Mongo, HBase, etc).

## Original Target

In my work, I need to handle data from different datasources (mostly Mysql & Mongo) to generate the final OLAP query result. Our goal is to establish a universal data platform to access, especially to process `JOIN` operation across schema on multiple datasources.

## Approach-1: Pandas ETL engine

We originally used [pandas](http://pandas.pydata.org/) to load required schemas as (pandas) Dataframes and then process all data operations within memory. This approach, however, is

- **Time Consuming**: requires great efforts to load dataframes into memory
- **Lack of Scalability**: cannot handle large-scale data well since the entire platform is resided in single node.
- **Difficult to Access**: needs pandas APIs to process all the data operations. There are methods to use SQL to handle pandas Dataframe (e.g., [sql4pandas](https://github.com/keeganmccallum/sql4pandas)), but the supported sql syntax is limited.

At last, we come to Spark. In SparkSQL, the basic operational data unit is also `DataFrame`, no matter a table in RDB, a collection in MongoDB, or a document in ElasticSearch. Moreover, its `lazy evaluation` of Dataframe enable it to process ETL job until the time we really need to access it, which makes it efficient in data handling and aware of change of external datasource.

## Approach-2: PySpark Jupyter Notebook

The idea is very easy, we register all Dataframes as temporary tables at first. Then we can use sql via Spark SQLContext to operate multiple datasources directly. Its easy to setup the jupyter notebook environment using PySpark. You can check the following demo notebook at my github repository ([here](https://github.com/bailaohe/spark-notebook)). I post the source code as follows.

### Initialize pySpark Environment

```python
import os
import sys

# Add support to access mysql
SPARK_CLASSPATH = "./libs/mysql-connector-java-5.1.38-bin.jar"
# Add support to access mongo (from official)
SPARK_CLASSPATH += ":./libs/mongo-hadoop-core-1.5.2.jar"
SPARK_CLASSPATH += ":./libs/mongo-java-driver-3.2.2.jar"
# Add support to access mongo (from stratio) based on casbah libs
SPARK_CLASSPATH += ":./libs/casbah-commons_2.10-3.1.1.jar"
SPARK_CLASSPATH += ":./libs/casbah-core_2.10-3.1.1.jar"
SPARK_CLASSPATH += ":./libs/casbah-query_2.10-3.1.1.jar"
SPARK_CLASSPATH += ":./libs/spark-mongodb_2.10-0.11.1.jar"

# Set the environment variable SPARK_CLASSPATH
os.environ['SPARK_CLASSPATH'] = SPARK_CLASSPATH

# Add pyspark to sys.path
spark_home = os.environ.get('SPARK_HOME', None)
sys.path.insert(0, spark_home + "/python")

# Add the py4j to the path.
# You may need to change the version number to match your install
sys.path.insert(0, os.path.join(spark_home, 'python/lib/py4j-0.9-src.zip'))

from pyspark import SparkContext
from pyspark import SparkConf
from pyspark.sql import SQLContext

# Initialize spark conf/context/sqlContext
conf = SparkConf().setMaster("local[*]").setAppName('spark-etl')
sc = SparkContext(conf=conf)
sqlContext = SQLContext(sc)
```

### Initial Data Access Drivers (Mysql/Mongo/...)

```python
# 1. Initialize the mysql driver
mysql_host = "YOUR_MYSQL_HOST"
mysql_port = 3306
mysql_db = "YOUR_MYSQL_DB"
mysql_user = "YOUR_MYSQL_USER"
mysql_pass = "YOUR_MYSQL_PASS"
mysql_driver = "com.mysql.jdbc.Driver"

mysql_prod = sqlContext.read.format("jdbc").options(
    url="jdbc:mysql://{host}:{port}/{db}".format(host=mysql_host, port=mysql_port, db=mysql_db),
    driver = mysql_driver,
    user=mysql_user,
    password=mysql_pass)

# 2. Initalize the official mongo driver
mongo_user = "YOUR_MONGO_USER"
mongo_pass = "YOUR_MONGO_PASSWORD"
mongo_host = "127.0.0.1"
mongo_port = 27017
mongo_db = "test"
```

### Register Temporary Tables from datasources (Mysql/Mongo/...)

```python
# 1. Register mysql temporary tables
df_deal = mysql_prod.load(dbtable = "YOUR_MYSQL_TABLE")
df_deal.registerTempTable("mysql_table")

# 2. Register mongo temporary tables
sqlContext.sql("CREATE TEMPORARY TABLE mongo_table USING com.stratio.datasource.mongodb OPTIONS (host '{host}:{port}', database '{db}', collection '{table}')".format(
    host=mongo_host,
    port=mongo_port,
    db=mongo_db,
    table="demotbl"
))
```

Then We can use SparkSQL as follows:

```python
df_mongo = sqlContext.sql("SELECT * FROM mongo_table limit 10")
df_mongo.collect()
```

## Approach-3: OLAP SQL Database on SparkSQL Thrift

We take our step furthermore, we want to make our platform as a **database**, facilitate us to access it in our program via JDBC driver, and to support different legacy BI application (e.g., Tableau, QlikView).

As mentioned above, SparkSQL can use Hive metastore directly. Thus, we want to start the SparkSQL thriftserver accompy with Hive metastore service, establish the environment with some SparkSQL DDL statements to create the `symbol-links` to external datasources.

The work is also very easy, just share the same hive-site.xml between Hive metastore service and SparkSQL thriftserver. We post the content of hive-site.xml as follows. It's only a toy settings without any Hadoop/HDFS/Mapreduce stuff to highlight the key points, you can adapt it quickly for production use.

### Config hive-site.xml

```xml
<configuration>
<property>
  <name>javax.jdo.option.ConnectionURL</name>
  <value>jdbc:mysql://localhost:3306/hive?createDatabaseIfNotExist=true</value>
  <description>JDBC connect string for a JDBC metastore</description>
</property>

<property>
  <name>javax.jdo.option.ConnectionDriverName</name>
  <value>com.mysql.jdbc.Driver</value>
  <description>Driver class name for a JDBC metastore</description>
</property>

<property>
  <name>javax.jdo.option.ConnectionUserName</name>
  <value>hive</value>
  <description>username to use against metastore database</description>
</property>

<property>
  <name>javax.jdo.option.ConnectionPassword</name>
  <value>Bh@840922</value>
  <description>password to use against metastore database</description>
</property>

<property>
  <name>hive.metastore.uris</name>
  <value>thrift://localhost:9083</value>
  <description>Thrift URI for the remote metastore. Used by metastore client to connect to remote metastore.</description>
</property>

<property>
  <name>hive.server2.thrift.port</name>
  <value>10000</value>
</property>

<property>
  <name>hive.server2.thrift.bind.host</name>
  <value>localhost</value>
</property>
</configuration>
```

### Start the SparkSQL thriftserver with required jars

```bash
#!/bin/sh

${SPARK_HOME}/sbin/start-thriftserver.sh \
  --jars ${WORKDIR}/libs/mongo-java-driver-3.2.2.jar, \
  ${WORKDIR}/libs/casbah-commons_2.10-3.1.1.jar, \
  ${WORKDIR}/libs/casbah-core_2.10-3.1.1.jar, \
  ${WORKDIR}/libs/casbah-query_2.10-3.1.1.jar, \
  ${WORKDIR}/libs/spark-mongodb_2.10-0.11.1.jar, \
  ${WORKDIR}/libs/mysql-connector-java-5.1.38-bin.jar

```

OK, everything is done! Now you can do the same thing as approach-2 to create a symbol-link to external mongo table as follows in your beeline client:

```sql
CREATE TEMPORARY TABLE mongo_table USING com.stratio.datasource.mongodb OPTIONS (host 'localhost:27017', database 'test', collection 'demotbl');
```

Then you can access it via normal query statement:

<pre>
0: jdbc:hive2://localhost:10000> show tables;
+--------------+--------------+--+
|  tableName   | isTemporary  |
+--------------+--------------+--+
| mongo_table  | false        |
+--------------+--------------+--+
1 row selected (0.108 seconds)
0: jdbc:hive2://localhost:10000> select * from mongo_table;
+------+----+---------------------------+--+
|  x   | y  |            _id            |
+------+----+---------------------------+--+
| 1.0  | a  | 5715f227d2f82889971df7f1  |
| 2.0  | b  | 57170b5e582cb370c48f085c  |
+------+----+---------------------------+--+
2 rows selected (0.38 seconds)
</pre>
