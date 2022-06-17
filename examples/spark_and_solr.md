
# Spark Solr Integration on a Secured & Unsecured Cluster

Access document from Solr collection (Read + Write)


## Tech Stack

* Spark
* Solr

## Usage/Examples

### Pre-requisites:
- Login via SSH to any of the Infra Solr nodes.

- If using Kerberos, set the SOLR_PROCESS_DIR environment variable

```bash
$ export SOLR_PROCESS_DIR=$(ls -1dtr /var/run/cloudera-scm-agent/process/*SOLR_SERVER | tail -1)
```

### Steps to perform:

1. Create a Solr collection config directory on local.

```bash
$ solrctl instancedir --generate /tmp/testLocal
```

You can modify the schema file **(/tmp/testLocal/conf/managed-schema)** according to the requirement.

2. Upload the local config files to the Zookeeper.

```bash
$ solrctl instancedir --create testConfig /tmp/testLocal
$ solrctl --jaas jaas.conf instancedir --create testConfig /tmp/testLocal (for Secured Cluster)
```

3. Create the collection.

```bash
$ solrctl collection --create test -c testConfig
```

4. List the collections.

```bash
$ solrctl collection --list
test (5)
```

5. Insert few records into the collection.

```bash
$ curl -k --negotiate -u : -X POST -d '{"add":{ "doc":{"id":"101", "name":"Deepak"}}}' -H "Content-Type: application/json" https://$(hostname -f):8985/solr/test/update?commit=true

$ curl -k --negotiate -u : -X POST -d '{"add":{ "doc":{"id":"102", "name":"James"}}}' -H "Content-Type: application/json" https://$(hostname -f):8985/solr/test/update?commit=true

$ curl -k --negotiate -u : -X POST -d '{"add":{ "doc":{"id":"103", "name":"John"}}}' -H "Content-Type: application/json" https://$(hostname -f):8985/solr/test/update?commit=true
```
On successful insert you will see below response.

![App Screenshot](https://github.com/deepakpanda93/Spark_Solr_SSL_Example/blob/master/src/main/resources/assets/solr_success_insert.png?raw=true)

6. Display the records of your collection.

```bash
$ curl -k --negotiate -u : "https://$(hostname -f):8985/solr/test/select?q=*:*"
```
![App Screenshot](https://github.com/deepakpanda93/Spark_Solr_SSL_Example/blob/master/src/main/resources/assets/Solr_initial_data.png?raw=true)

7. Add an user on each machine.

```bash
$ useradd dpanda -g hadoop
```

8. Add a principal and create a keytab

```bash
$ kadmin.local addprinc dpanda
$ kadmin.local ktadd -k /tmp/dpanda.keytab  dpanda
```

9. Add the user to Ranger and grant it with valid permissions to access the Collection.

- ***Ranger Home***
![App Screenshot](https://github.com/deepakpanda93/Spark_Solr_SSL_Example/blob/master/src/main/resources/assets/Ranger_Home.png?raw=true)
- ***Add user to "all-collection"***
![App Screenshot](https://github.com/deepakpanda93/Spark_Solr_SSL_Example/blob/master/src/main/resources/assets/Ranger_addUser.png?raw=true)

10. Create the JAAS file.

```bash
$ cat solr_jaas.conf

SolrJClient {
  com.sun.security.auth.module.Krb5LoginModule required
  useTicketCache=false
  useKeyTab=true
  principal="dpanda@<principal>"
  keyTab="dpanda.keytab"
  renewTicket=true
  storeKey=true;
};
Client {
  com.sun.security.auth.module.Krb5LoginModule required
  useTicketCache=false
  useKeyTab=true
  principal="dpanda@<principal>"
  keyTab="dpanda.keytab"
  renewTicket=true
  storeKey=true
  serviceName="zookeeper";
};
```

11. Execute the below command on the terminal.

```bash
$ export SPARK_OPTS="-Djavax.net.ssl.trustStore=./cm-auto-global_truststore.jks -Djavax.net.ssl.trustStorePassword=${TRUSTSTORE_PASSWORD} -Djava.security.auth.login.config=./solr_jaas.conf"
```

12. Run spark-shell with spark-solr connector, truststore and jaas file.

```bash
$ spark-shell --deploy-mode client --jars spark-solr-3.9.0.7.1.7.1000-142-shaded.jar --files cm-auto-global_truststore.jks,./solr_jaas.conf#solr_jaas.conf,dpanda.keytab#dpanda.keytab \
--conf "spark.executor.extraJavaOptions=-Djava.security.auth.login.config=./solr_jaas.conf \
-DsolrJaasAuthConfig=./solr_jaas.conf \
-Djavax.net.ssl.trustStore=./cm-auto-global_truststore.jks \
-Djavax.net.ssl.trustStorePassword=${TRUSTSTORE_PASSWORD} -Djavax.security.auth.useSubjectCredsOnly=false" --driver-java-options="-Djava.security.auth.login.config=./solr_jaas.conf \
-DsolrJaasAuthConfig=./solr_jaas.conf \
-Djavax.net.ssl.trustStore=./cm-auto-global_truststore.jks \
-Djavax.net.ssl.trustStorePassword=${TRUSTSTORE_PASSWORD} -Djavax.security.auth.useSubjectCredsOnly=false" -DsolrJaasAuthConfig=solr_jaas.conf

Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.4.7.7.1.7.1000-142
      /_/

Using Scala version 2.11.12 (OpenJDK 64-Bit Server VM, Java 1.8.0_232)
Type in expressions to have them evaluated.
Type :help for more information.

scala> val options = Map("collection" -> "test", "zkhost" -> "node1.example.com:2181/solr")
options: scala.collection.immutable.Map[String,String] = Map(collection -> test, zkhost -> node1.example.com:2181/solr)

scala> val solrDF = spark.read.format("solr").options(options).load
solrDF: org.apache.spark.sql.DataFrame = [id: string, name: string]

scala> solrDF.printSchema()
root
 |-- id: string (nullable = false)
 |-- name: string (nullable = true)


scala> solrDF.show(false)
+---+--------+
|id |name    |
+---+--------+
|102|"James" |
|101|"Deepak"|
|103|"John"  |
+---+--------+


scala> val columns = Seq("id","name")
columns: Seq[String] = List(id, name)

scala> val data = Seq((104, "Clark"), (105, "Liang"), (106, "Aditya"))
data: Seq[(Int, String)] = List((104,Clark), (105,Liang), (106,Aditya))

scala> val employeeRDD = spark.sparkContext.parallelize(data)
employeeRDD: org.apache.spark.rdd.RDD[(Int, String)] = ParallelCollectionRDD[7] at parallelize at <console>:25

scala> val employeeDF = spark.createDataFrame(employeeRDD).toDF(columns:_*)
employeeDF: org.apache.spark.sql.DataFrame = [id: int, name: string]

scala> employeeDF.write.format("solr").options(options).save

scala> 22/06/16 12:19:03 WARN sql.CommandsHarvester$: Missing unknown leaf node: ExternalRDD [obj#13]

22/06/16 12:19:03 WARN sql.CommandsHarvester$: Missing output entities: SaveIntoDataSourceCommand solr.DefaultSource@32c3f813, Map(collection -> test, zkhost -> node1.example.com:2181/solr), ErrorIfExists
   +- Project [_1#14 AS id#18, _2#15 AS name#19]
      +- SerializeFromObject [assertnotnull(assertnotnull(input[0, scala.Tuple2, true]))._1 AS _1#14, staticinvoke(class org.apache.spark.unsafe.types.UTF8String, StringType, fromString, assertnotnull(assertnotnull(input[0, scala.Tuple2, true]))._2, true, false) AS _2#15]
         +- ExternalRDD [obj#13]

22/06/16 12:19:03 WARN producer.ProducerConfig: The configuration 'key.deserializer' was supplied but isn't a known config.
22/06/16 12:19:03 WARN producer.ProducerConfig: The configuration 'value.deserializer' was supplied but isn't a known config.
22/06/16 12:19:03 WARN producer.ProducerConfig: The configuration 'max.poll.records' was supplied but isn't a known config.
22/06/16 12:19:03 WARN producer.ProducerConfig: The configuration 'zookeeper.connection.timeout.ms' was supplied but isn't a known config.
22/06/16 12:19:03 WARN producer.ProducerConfig: The configuration 'zookeeper.session.timeout.ms' was supplied but isn't a known config.
22/06/16 12:19:03 WARN producer.ProducerConfig: The configuration 'enable.auto.commit' was supplied but isn't a known config.
22/06/16 12:19:03 WARN producer.ProducerConfig: The configuration 'zookeeper.connect' was supplied but isn't a known config.
22/06/16 12:19:03 WARN producer.ProducerConfig: The configuration 'zookeeper.sync.time.ms' was supplied but isn't a known config.
22/06/16 12:19:03 WARN producer.ProducerConfig: The configuration 'session.timeout.ms' was supplied but isn't a known config.
22/06/16 12:19:03 WARN producer.ProducerConfig: The configuration 'auto.offset.reset' was supplied but isn't a known config.


scala> val resultDF = spark.read.format("solr").options(options).load
resultDF: org.apache.spark.sql.DataFrame = [id: string, name: string]

scala> resultDF.show(false)
+---+--------+
|id |name    |
+---+--------+
|102|"James" |
|104|"Clark" |
|105|"Liang" |
|106|"Aditya"|
|101|"Deepak"|
|103|"John"  |
+---+--------+
```

### On UnSecured Cluster 

- We can access the solr collection from IDE

1. Create a Spark Code file to read/write data from/to Solr Collection.

```bash
$ cat Spark_Solr_Example.scala

package com.deepak.spark.solr

import org.apache.spark.sql.SparkSession

object Spark_Solr_Example {

  def main(args: Array[String]): Unit = {

    val appName = "Spark Solr Integration"

    // Creating the SparkSession object
    var spark: SparkSession = SparkSession.builder().master("local").appName(appName).getOrCreate()

    val options = Map("collection" -> "testCollection", "zkhost" -> "node3.example.com:2181/solr")

    // Read from Solr Collection
    val solrDF = spark.read.format("solr").options(options).load

    solrDF.printSchema()

    solrDF.show(100,false)

    val columns = Seq("id","name")
    val data = Seq((104, "Clark"), (105, "Mathew"), (106, "Chandra"))

    val employeeRDD = spark.sparkContext.parallelize(data)
    val employeeDF = spark.createDataFrame(employeeRDD).toDF(columns:_*)

    val writeOptions = Map("collection" -> "testCollection", "zkhost" -> "node3.example.com:2181/solr", "soft_commit_secs" -> "1")

    // Write to Solr Collection
    employeeDF.write.format("solr").options(writeOptions).save

    val resultDF = spark.read.format("solr").options(options).load
    resultDF.show(100,false)

    spark.stop()

  }
}
```

2. Prepare a pom.xml file for your project

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>Spark_Solr_SSL</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.build.targetEncoding>UTF-8</project.build.targetEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>

        <maven.compiler.plugin.version>3.8.1</maven.compiler.plugin.version>
        <maven-shade-plugin.version>3.2.3</maven-shade-plugin.version>
        <scala-maven-plugin.version>3.2.2</scala-maven-plugin.version>
        <scalatest-maven-plugin.version>2.0.0</scalatest-maven-plugin.version>

        <scala.version>2.11.8</scala.version>
        <scala.binary.version>2.11</scala.binary.version>
        <spark.version>2.4.7</spark.version>
        <jackson.version>2.11.0</jackson.version>
        <spark.scope>provided</spark.scope>
    </properties>

    <!-- Developers -->
    <developers>
        <developer>
            <id>deepakpanda93</id>
            <name>Deepak Panda</name>
            <email>deepakpanda93@gmail.com</email>
            <url>https://github.com/deepakpanda93</url>
        </developer>
    </developers>

    <!-- Repositories -->
    <repositories>
        <repository>
            <id>central</id>
            <name>Maven Central</name>
            <url>https://repo1.maven.org/maven2</url>
        </repository>

        <repository>
            <id>cldr-repo</id>
            <name>Cloudera Public Repo</name>
            <url>https://repository.cloudera.com/artifactory/cloudera-repos/</url>
        </repository>

        <repository>
            <id>hdp-repo</id>
            <name>Hortonworks Public Repo</name>
            <url>https://repo.hortonworks.com/content/repositories/releases/</url>
        </repository>
    </repositories>

    <dependencies>
        <dependency>
            <groupId>org.scala-lang</groupId>
            <artifactId>scala-library</artifactId>
            <version>${scala.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-sql-kafka-0-10_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>com.lucidworks.spark</groupId>
            <artifactId>spark-solr</artifactId>
            <version>1.0</version>
            <scope>system</scope>
            <systemPath>/Users/dpanda/Downloads/spark-solr-3.9.0.7.1.7.1000-141-shaded.jar</systemPath>
        </dependency>
    </dependencies>

    <build>
        <plugins>

            <!-- Maven Compiler Plugin -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>${maven.compiler.plugin.version}</version>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                    <compilerArgs>
                        <arg>-Xlint:all,-serial,-path</arg>
                    </compilerArgs>
                </configuration>
            </plugin>

            <!-- Scala Maven Plugin -->
            <plugin>
                <groupId>net.alchim31.maven</groupId>
                <artifactId>scala-maven-plugin</artifactId>
                <version>${scala-maven-plugin.version}</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>compile</goal>
                            <goal>testCompile</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <scalaVersion>${scala.version}</scalaVersion>
                    <scalaCompatVersion>${scala.binary.version}</scalaCompatVersion>
                    <checkMultipleScalaVersions>true</checkMultipleScalaVersions>
                    <failOnMultipleScalaVersions>true</failOnMultipleScalaVersions>
                    <recompileMode>incremental</recompileMode>
                    <charset>${project.build.sourceEncoding}</charset>
                    <args>
                        <arg>-unchecked</arg>
                        <arg>-deprecation</arg>
                        <arg>-feature</arg>
                    </args>
                    <jvmArgs>
                        <jvmArg>-Xss64m</jvmArg>
                        <jvmArg>-Xms1024m</jvmArg>
                        <jvmArg>-Xmx1024m</jvmArg>
                        <jvmArg>-XX:ReservedCodeCacheSize=1g</jvmArg>
                    </jvmArgs>
                    <javacArgs>
                        <javacArg>-source</javacArg>
                        <javacArg>${java.version}</javacArg>
                        <javacArg>-target</javacArg>
                        <javacArg>${java.version}</javacArg>
                        <javacArg>-Xlint:all,-serial,-path</javacArg>
                    </javacArgs>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```



## Run Locally

Run the Spark job from IDE

```bash
I used InteliJ to run the project. But one can build the project, deploy the JAR on the cluster and execute using spark-submit
```




## Screenshots

### Initial records of Solr Collection (testCollection)

![App Screenshot](https://github.com/deepakpanda93/Spark_Solr_SSL_Example/blob/master/src/main/resources/assets/testCollection_initialData.png?raw=true)

### Initial records of Solr Collection (Fetched using Spark)

![App Screenshot](https://github.com/deepakpanda93/Spark_Solr_SSL_Example/blob/master/src/main/resources/assets/testCollection_spark_initialData.png?raw=true)

### Final output in Solr Collection (testCollection)

![App Screenshot](https://github.com/deepakpanda93/Spark_Solr_SSL_Example/blob/master/src/main/resources/assets/testCollection_finaldata.png?raw=true)

### Remove all data from a collection

![App Screenshot](https://github.com/deepakpanda93/Spark_Solr_SSL_Example/blob/master/src/main/resources/assets/testCollection_spark_finalData.png?raw=true)

## 🚀 About Me

## Hi, I'm Deepak! 👋

I'm a Big Data Engineer...




## 🔗 Links
[![portfolio](https://img.shields.io/badge/my_portfolio-000?style=for-the-badge&logo=ko-fi&logoColor=white)](https://github.com/deepakpanda93)
[![linkedin](https://img.shields.io/badge/linkedin-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/deepakpanda93)
