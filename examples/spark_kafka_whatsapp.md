
# Spark_Elastic_WhatsApp Example

Extract User input data from WEB API and send them to WhatsApp with Spark-Kafka Streaming



## Tech Stack

* Logstash
* Kafka
* Spark Structured Streaming


## Installation

Install ElasticSearch 7.x on CentOS 7

Step 1: Update CentOS 7 Linux

```bash
sudo yum -y update
```

Step 2: Install Java on CentOS 7

```bash
sudo yum -y install java-1.8.0-openjdk  java-1.8.0-openjdk-devel
```

Step 3: Add ElasticSearch Yum repository
Add the repository for downloading ElasticSearch 7 packages to your CentOS 7 system.

```bash
cat <<EOF | sudo tee /etc/yum.repos.d/elasticsearch.repo
[elasticsearch-7.x]
name=Elasticsearch repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/oss-7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
```

If you want to install Elasticsearch 6, replace all occurrences of 7 with 6. Once the repository is added, clear and update your YUM package index.

```bash
sudo yum clean all
sudo yum makecache
```    

Step 4: Install ElasticSearch 7 on CentOS 7
Finally install ElasticSearch 7.x on CentOS 7 machine. 

```bash
sudo yum -y install elasticsearch-oss
```

Let's confirm ElasticSearch 7 installation on CentOS 7:

```bash
# rpm -qi elasticsearch-oss
Name        : elasticsearch-oss
Epoch       : 0
Version     : 7.10.2
Release     : 1
Architecture: x86_64
Install Date: Tue Jun  7 18:33:58 2022
Group       : Application/Internet
Size        : 420252496
License     : ASL 2.0
Signature   : RSA/SHA512, Wed Jan 13 03:45:21 2021, Key ID d27d666cd88e42b4
Source RPM  : elasticsearch-oss-7.10.2-1-src.rpm
Build Date  : Wed Jan 13 00:54:36 2021
Build Host  : packer-virtualbox-iso-1600176624
Relocations : /usr
Packager    : Elasticsearch
Vendor      : Elasticsearch
URL         : https://www.elastic.co/
Summary     : Distributed RESTful search engine built for the cloud
Description :
Reference documentation can be found at
  https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html
  and the 'Elasticsearch: The Definitive Guide' book can be found at
  https://www.elastic.co/guide/en/elasticsearch/guide/current/index.html
```

Step 4: Configure Java memory Limits
You can set JVM options like memory limits by editing the file: /etc/elasticsearch/jvm.options

Example below sets initial/maximum size of total heap space

```bash
$ sudo vi /etc/elasticsearch/jvm.options
.....
-Xms1g
-Xmx1g
```

Step 5: Start and enable elasticsearch service on boot

```bash
sudo systemctl enable --now elasticsearch
```

Confirm that the service is running.

```bash
sudo systemctl status elasticsearch
```

Check if you can connect to ElasticSearch Service.

```bash
curl http://node4.example.com:9200
{
  "name" : "node4.example.com",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "Ubudz1MzRhyxf_Xdsjdge",
  "version" : {
    "number" : "7.10.2",
    "build_flavor" : "oss",
    "build_type" : "rpm",
    "build_hash" : "747e1cc71def077253878a59143c1f785afa92b9",
    "build_date" : "2021-01-13T00:42:12.435326Z",
    "build_snapshot" : false,
    "lucene_version" : "8.7.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

You should be able to create an index with curl.

```bash
curl -X PUT "http://node4.example.com:9200/my_test_index"
{"acknowledged":true,"shards_acknowledged":true,"index":"my_test_index"}
```

Step 6: Install Kibana 7 on CentOS 7

```bash
sudo yum install kibana-oss logstash
```

After a successful installation, configure Kibana:

```bash
$ sudo vi /etc/kibana/kibana.yml
server.host: "http://node4.example.com"
server.name: "kibana.example.com"
elasticsearch.url: ["http://node4.example.com:9200"]
```

Change other settings as desired then start kibana service:

```bash
sudo systemctl enable --now kibana
```

If you have an active firewall, you’ll need to allow access to Kibana port:

```bash
sudo firewall-cmd --add-port=5601/tcp --permanent
sudo firewall-cmd --reload
```

Access http://ip-address:5601 to open Kibana Dashboard:

![KibanaUI](https://github.com/deepakpanda93/SparkStreaming-Kafka-ElasticSearch-WhatsApp/blob/master/src/main/assets/KibanaUI_new.png?raw=true) 
## Usage/Examples

1) Create a Kafka topic.

```bash
kafka-topics --bootstrap-server [HOST1:PORT1] --create --topic [TOPIC] --partitions <no_of_partitions> --replication-factor <replication_factor>
```

2) Create a Logstash config file to hit the WEB API every second and retrieve the user input data.

```javascript
## cat api_kafka_logstash.conf
input {
  http_poller {
    urls => {
      urlname => "https://test.ekhool.com/checking/user_fetch"
    }
    request_timeout => 60
    schedule => { every => "1s" }
    codec => "line"
  }
}

output {
elasticsearch{
 hosts => ["<elasticSearch_server>:9200"]
 index => "<index_name>"
}
kafka {
    bootstrap_servers => "<bootstrap_servers>:9092"
    codec => json
    topic_id => "<topic_name>"
}
  stdout {
    codec => rubydebug
  }
}
```

3) Create a Spark Streaming Code file.

```javascript
## cat Spark_Kafka_WhatsApp.scala

package com.whatsapp.spark.kafka

import org.apache.spark.sql.{DataFrame, Dataset, SparkSession}
import org.apache.spark.sql.types.{StringType, StructType}
import org.apache.spark.sql.functions.{col, from_json, window}
case class Employee_Cloudera(name : String)

object Spark_Kafka_WhatsApp {

  private val whatsapp: SendWhatsApp = new SendWhatsApp()

  private def sendtoWhatsapp(batchDF : Dataset[Employee_Cloudera], batchId : Long): Unit = {
    batchDF.show()
    batchDF.collect().foreach(employee => {
      whatsapp.sendmessage( "Hello  " + employee.name)
    })
  }

  def main(args: Array[String]): Unit = {

  /*  if(args.length > 2 ) {
      System.err.println("Usage : SparkElasticWhatsappIntegrationApp <KAFKA_BOOTSTRAP_SERVERS> <KAFKA_TOPIC_NAME>");
      System.exit(0);
    } */

    val appName = "Spark Elastic Whatsapp Integration"

    whatsapp.init()

    // Creating the SparkSession object
    val spark: SparkSession = SparkSession.builder().master("local").appName(appName).getOrCreate()
    import spark.implicits._

    val kafkaBootstrapServers = "bootstrap_server" //args(0)
    val inputTopicNames = "topic_name" //args(1)

    val schema = new StructType()
      .add("message", StringType, true)
      .add("@timestamp", StringType, true)
      .add("@version", StringType, true)


    val inputDf = spark.
      readStream.
      format("kafka").
      option("kafka.bootstrap.servers", kafkaBootstrapServers).
      option("subscribe", inputTopicNames).
      option("startingOffsets", "latest").
      option("kafka.security.protocol","PLAINTEXT").
      load().selectExpr("CAST(value AS STRING)").as[String]

    inputDf.printSchema()

    val dfJSON = inputDf.withColumn("jsonData",from_json(col("value"),schema)).select("jsonData.*")

    val messageData : DataFrame = dfJSON.select("message").filter(col("message") =!= "" && col("message") =!= "[\"empty_data\"]")

    messageData.printSchema()

    val nameDF = messageData.map(value => {
      val nameData : String = value.toString().split(":")(1).replace("}]]", "").replaceAll("^\"|\"$", "")
      Employee_Cloudera(nameData)
    })

    nameDF.printSchema()

    val outputDF = nameDF.writeStream.foreachBatch((batchDF: Dataset[Employee_Cloudera], batchId : Long) => sendtoWhatsapp(batchDF, batchId))
      .outputMode("append")

    outputDF.start().awaitTermination()

  }
}

```

4) Create a Java code file to send to whatsapp using Twilio API.

```javascript
## cat SendWhatsApp.java

package com.whatsapp.spark.kafka;

import com.twilio.Twilio;
import com.twilio.rest.api.v2010.account.Message;

public class SendWhatsApp {
   public void sendmessage(String yourmessage) {
        Message message = Message.creator(
                new com.twilio.type.PhoneNumber("whatsapp:+YOUR_WHATSAPP_PHONE_NUMBER"),
                new com.twilio.type.PhoneNumber("whatsapp:+14155238886"),
                yourmessage)
                .create();

        System.out.println(message.getSid());
    }

    public void init() {
        String ACCOUNT_SID = "TWILIO_ACCOUNT_SID";
        String AUTH_TOKEN =  "TWILIO_AUTH_TOKEN";

        Twilio.init(ACCOUNT_SID, AUTH_TOKEN);
    }
}

```

5) Prepare a pom.xml file for your project

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>Spark_Kafka</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <scala.version>2.11.8</scala.version>
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
            <groupId>com.twilio.sdk</groupId>
            <artifactId>twilio</artifactId>
            <version>8.9.0</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.dataformat</groupId>
            <artifactId>jackson-dataformat-xml</artifactId>
            <version>2.12.1</version>
        </dependency>

        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-core</artifactId>
            <version>2.12.1</version>
        </dependency>

        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.12.1</version>
        </dependency>

        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-annotations</artifactId>
            <version>2.12.1</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.module</groupId>
            <artifactId>jackson-module-scala_2.11</artifactId>
            <version>2.12.1</version>
        </dependency>
    </dependencies>
</project>
```



## Run Locally

Start the logstash

```bash
  /usr/share/logstash/bin/logstash -f api_kafka_logstash.conf
```

Start the spark-kafka streaming

```bash
  I used Intelij to run the project
```

Send some data from the URL : https://test.ekhool.com/checking/user_input

Check your whatsapp AND BOOM !!!!




## Screenshots

### Logstash Ingested Data

![App Screenshot](https://github.com/deepakpanda93/SparkStreaming-Kafka-ElasticSearch-WhatsApp/blob/master/src/main/assets/logstashData_1.png?raw=true)
![App Screenshot](https://github.com/deepakpanda93/SparkStreaming-Kafka-ElasticSearch-WhatsApp/blob/master/src/main/assets/logstashData_2.png?raw=true)
![App Screenshot](https://github.com/deepakpanda93/SparkStreaming-Kafka-ElasticSearch-WhatsApp/blob/master/src/main/assets/logstashData_3.png?raw=true)
![App Screenshot](https://github.com/deepakpanda93/SparkStreaming-Kafka-ElasticSearch-WhatsApp/blob/master/src/main/assets/logstashData_4.png?raw=true)
![App Screenshot](https://github.com/deepakpanda93/SparkStreaming-Kafka-ElasticSearch-WhatsApp/blob/master/src/main/assets/logstashData_5.png?raw=true)
![App Screenshot](https://github.com/deepakpanda93/SparkStreaming-Kafka-ElasticSearch-WhatsApp/blob/master/src/main/assets/logstashData_6.png?raw=true)


### Spark Streaming Ingested Data

![App Screenshot](https://github.com/deepakpanda93/SparkStreaming-Kafka-ElasticSearch-WhatsApp/blob/master/src/main/assets/SparkStreamingData_1.png?raw=true)
![App Screenshot](https://github.com/deepakpanda93/SparkStreaming-Kafka-ElasticSearch-WhatsApp/blob/master/src/main/assets/SparkStreamingData_2.png?raw=true)
![App Screenshot](https://github.com/deepakpanda93/SparkStreaming-Kafka-ElasticSearch-WhatsApp/blob/master/src/main/assets/SparkStreamingData_3.png?raw=true)
![App Screenshot](https://github.com/deepakpanda93/SparkStreaming-Kafka-ElasticSearch-WhatsApp/blob/master/src/main/assets/SparkStreamingData_4.png?raw=true)
![App Screenshot](https://github.com/deepakpanda93/SparkStreaming-Kafka-ElasticSearch-WhatsApp/blob/master/src/main/assets/SparkStreamingData_5.png?raw=true)


### Final output in WhatsApp

![App Screenshot](https://github.com/deepakpanda93/SparkStreaming-Kafka-ElasticSearch-WhatsApp/blob/master/src/main/assets/whatsapp.jpg?raw=true)

## Demo

To be uploaded


## 🚀 About Me

## Hi, I'm Deepak! 👋

I'm a Big Data Engineer...




## 🔗 Links
[![portfolio](https://img.shields.io/badge/my_portfolio-000?style=for-the-badge&logo=ko-fi&logoColor=white)](https://github.com/deepakpanda93)
[![linkedin](https://img.shields.io/badge/linkedin-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/deepakpanda93)
