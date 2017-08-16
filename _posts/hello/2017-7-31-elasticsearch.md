---
layout: post
title: elasticsearch 笔记
permalink: /:categories/elasticsearch_url/
date: 2017-7-31 09:30:15 +0800
category: elasticsearch
tags: [elasticsearch]
---



### elasticsearch 5.5.0

#### elasticsearch安装

下载elasticsearch 5.5.0  Windows版本

Run bin/elasticsearch

http://localhost:9200   

#### kibana安装

下载kibana 5.5.0  Windows版本

bin\kibana.bat

http://localhost:5601  elastic  changeme


#### x-pack安装

#### Logstash 安装

bin/logstash.bat -f  ../config/logstash.conf

logstash.conf如下

```
 input { 
  file {
  path => "E:/logstash-5.5.0/tmp/*.log"
    type => "system"
    start_position => "beginning"
    codec=>plain{charset=>"UTF-8"}
    }
}

output {
  elasticsearch {
  
  user => elastic
  password => changeme
  
  hosts => ["localhost:9200"] 
    index => "nginx"
    template_overwrite => true
  }
  stdout { codec => rubydebug }
}

```

### java api

#### ESUtils

```
package com.bowen.es_5_5;

import java.net.InetAddress;

import org.elasticsearch.client.Client;
import org.elasticsearch.client.transport.TransportClient;
import org.elasticsearch.common.settings.Settings;
import org.elasticsearch.common.transport.InetSocketTransportAddress;
import org.elasticsearch.transport.client.PreBuiltTransportClient;
import org.elasticsearch.xpack.client.PreBuiltXPackTransportClient;


/** @getEsClient() 创建客户端，所有的操作都由客户端开始，这个就好像是JDBC的Connection对象
 *@closeClient() 用完记得要关闭
 **/
public class ESUtils {
	public static final String INDEX_NAME="esindex";
	public static String getIndexName(){
	return INDEX_NAME;
	}
	public static final String TYPE_NAME="estype";
	public static String getTypeName(){
	return TYPE_NAME;
	}

	
	 public static Client getEsClient(){
		 TransportClient client=null;
     try {
   	  
    	 
    	 Settings settings = Settings.builder().put("cluster.name", "elasticsearch")
                 .put("client.transport.sniff", true)
                 .put("xpack.security.transport.ssl.enabled", false)
                 .put("xpack.security.user", "elastic:changeme")
                 .build();
         
    	   client = new PreBuiltXPackTransportClient(settings)
                 .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("127.0.0.1"), Integer.valueOf(9300)));

         
         
   	  
//		  //设置集群名称
//         Settings settings = Settings.builder().put("cluster.name", "elasticsearch").build();
//         //创建client
//         client = new PreBuiltTransportClient(settings)
//                 .addTransportAddress(new InetSocketTransportAddress(InetAddress.getByName("127.0.0.1"), 9300));
         
         
         
         
/*         //搜索数据
         GetResponse response = client.prepareGet("blog", "article", "1").execute().actionGet();
         //输出结果
         System.out.println(response.getSourceAsString());
         //关闭client
         client.close();
         */
     } catch (Exception e) {
    	 System.out.println("--------Create Client------error--------");
         e.printStackTrace();
     }
     System.out.println("--------Create Client------OK--------");

     return client;
	  }
	  
	  //关闭
	  public static void closeClient(Client esClient){
		  if(esClient !=null){
			  esClient.close();
		  }
	  }

}

```


#### TestEsClient

```
package com.bowen.es_5_5;

import java.net.InetAddress;
import java.util.*;

import org.elasticsearch.action.delete.DeleteResponse;
import org.elasticsearch.action.get.GetResponse;
import org.elasticsearch.action.index.IndexResponse;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.action.update.UpdateResponse;
import org.elasticsearch.client.Client;
import org.elasticsearch.client.transport.TransportClient;
import org.elasticsearch.common.settings.Settings;
import org.elasticsearch.common.transport.InetSocketTransportAddress;
import org.elasticsearch.index.query.QueryBuilder;
import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.search.SearchHit;
import org.elasticsearch.search.SearchHits;


public class TestEsClient {

	  public static void main(String[] args) {

//		  add();//添加
		  del();//删除
//		  upd();//修改
//		  sel();//查询
    }
	 	  //---------------------------------------------------------------------------------------------------------
	  
	  public static void add(){
		  Client client = ESUtils.getEsClient();
		  IndexResponse indexResponse =
		  client.prepareIndex().setIndex(ESUtils.getIndexName())
		  .setType(ESUtils.getTypeName())
		  .setSource("{\"prodId\":1,\"prodName\":\"ipad5\",\"prodDesc\":\"比你想的更强大\",\"catId\":1}")
		  .setId("1")
		  .execute()
		  .actionGet();
		  System.out.println("添加成功,isCreated="+indexResponse.toString());
		  ESUtils.closeClient(client);
	  }
	  
	  public static void del(){
		  Client client = ESUtils.getEsClient();

		  DeleteResponse delResponse =
				  client.prepareDelete().setIndex(ESUtils.getIndexName())
				  .setType(ESUtils.getTypeName())
				  .setId("1")
				  .execute()
				  .actionGet();
				  System.out.println("del is found="+delResponse.toString());
	  }
	  
	  public static void upd(){
		  Client client = ESUtils.getEsClient();

		  GetResponse getResponse =
				  client.prepareGet().setIndex(ESUtils.getIndexName())
				  .setType(ESUtils.getTypeName())
				  .setId("1")
				  .execute()
				  .actionGet();
				  System.out.println("berfore update version="+getResponse.getVersion());
				  UpdateResponse updateResponse =
				  client.prepareUpdate().setIndex(ESUtils.getIndexName())
				  .setType(ESUtils.getTypeName())
				  .setDoc("{\"prodId\":1,\"prodName\":\"ipad5\",\"prodDesc\":\"比你想的更强大噢耶\",\"catId\":1}")
				  .setId("1")
				  .execute()
				  .actionGet();
				  System.out.println("更新成功，isCreated="+updateResponse.toString());
				  getResponse =
				  client.prepareGet().setIndex(ESUtils.getIndexName())
				  .setType(ESUtils.getTypeName())
				  .setId("1")

				   
				  .execute()
				  .actionGet();
				  System.out.println("get version="+getResponse.getVersion());
				  System.out.println("-----upd----ok-----");
	  }
	  
	  public static void sel(){
		  Client client = ESUtils.getEsClient();
		  //初始化查询条件
		  QueryBuilder query = QueryBuilders.matchQuery("prodName", "ipad5");
		  
		  SearchResponse SearchResponseresponse = client.prepareSearch(ESUtils.INDEX_NAME)
		  //设置查询条件,
		  .setQuery(query)
		  .setFrom(0).setSize(60)
		  .execute()
		  .actionGet();
		  /**
		  * SearchHits是SearchHit的复数形式，表示这个是一个列表
		  */
		  SearchHits shs = SearchResponseresponse.getHits();
		  System.out.println("总共有："+shs.hits().length);
		  for(SearchHit hit : shs){
		  System.out.println(hit.getSourceAsString());
		  }
		  client.close();
	  }

}


```

#### pom配置

```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.bowen.es_5_5</groupId>
  <artifactId>es_5_5</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <packaging>war</packaging>
  
  
  
   <repositories>
        <!-- add the elasticsearch repo -->
        <repository>
            <id>elasticsearch-releases</id>
            <url>https://artifacts.elastic.co/maven</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>
    
    
    
<dependencies>
    
         <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>x-pack-transport</artifactId>
            <version>5.5.0</version>
        </dependency> 
     
		<dependency>
		<groupId>org.elasticsearch.client</groupId>
		<artifactId>x-pack-transport</artifactId>
		<version>5.5.0</version>
		</dependency>
     
     
        
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>transport</artifactId>
            <version>5.5.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-api</artifactId>
            <version>2.7</version>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-core</artifactId>
            <version>2.7</version>
        </dependency>
        
        
        
        
        <dependency>
      <groupId>org.elasticsearch.plugin</groupId>
      <artifactId>transport-netty3-client</artifactId>
      <version>5.5.0</version>
      <scope>runtime</scope>
    </dependency>
    <dependency>
      <groupId>org.elasticsearch.plugin</groupId>
      <artifactId>transport-netty4-client</artifactId>
      <version>5.5.0</version>
      <scope>runtime</scope>
    </dependency>
    <dependency>
      <groupId>com.unboundid</groupId>
      <artifactId>unboundid-ldapsdk</artifactId>
      <version>3.2.0</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.bouncycastle</groupId>
      <artifactId>bcprov-jdk15on</artifactId>
      <version>1.55</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.bouncycastle</groupId>
      <artifactId>bcpkix-jdk15on</artifactId>
      <version>1.55</version>
      <scope>compile</scope>
      <exclusions>
        <exclusion>
          <groupId>org.bouncycastle</groupId>
          <artifactId>bcprov-jdk15on</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
    <dependency>
      <groupId>com.googlecode.owasp-java-html-sanitizer</groupId>
      <artifactId>owasp-java-html-sanitizer</artifactId>
      <version>r239</version>
      <scope>compile</scope>
      <exclusions>
        <exclusion>
          <groupId>com.google.guava</groupId>
          <artifactId>guava</artifactId>
        </exclusion>
        <exclusion>
          <groupId>com.google.code.findbugs</groupId>
          <artifactId>jsr305</artifactId>
        </exclusion>
        <exclusion>
          <groupId>com.google.errorprone</groupId>
          <artifactId>error_prone_annotations</artifactId>
        </exclusion>
        <exclusion>
          <groupId>com.google.j2objc</groupId>
          <artifactId>j2objc-annotations</artifactId>
        </exclusion>
        <exclusion>
          <groupId>org.codehaus.mojo</groupId>
          <artifactId>animal-sniffer-annotations</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
    <dependency>
      <groupId>com.google.guava</groupId>
      <artifactId>guava</artifactId>
      <version>16.0.1</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>com.sun.mail</groupId>
      <artifactId>javax.mail</artifactId>
      <version>1.5.3</version>
      <scope>compile</scope>
      <exclusions>
        <exclusion>
          <groupId>javax.activation</groupId>
          <artifactId>activation</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
    <dependency>
      <groupId>javax.activation</groupId>
      <artifactId>activation</artifactId>
      <version>1.1</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.elasticsearch.client</groupId>
      <artifactId>rest</artifactId>
      <version>5.5.0</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.elasticsearch.client</groupId>
      <artifactId>sniffer</artifactId>
      <version>5.5.0</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>net.sf.supercsv</groupId>
      <artifactId>super-csv</artifactId>
      <version>2.4.0</version>
      <scope>compile</scope>
    </dependency>
    <dependency>
      <groupId>org.elasticsearch</groupId>
      <artifactId>elasticsearch</artifactId>
      <version>5.5.0</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>org.locationtech.spatial4j</groupId>
      <artifactId>spatial4j</artifactId>
      <version>0.6</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>com.vividsolutions</groupId>
      <artifactId>jts</artifactId>
      <version>1.13</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>org.apache.logging.log4j</groupId>
      <artifactId>log4j-api</artifactId>
      <version>2.8.2</version>
      <scope>provided</scope>
    </dependency>
    <dependency>
      <groupId>org.apache.logging.log4j</groupId>
      <artifactId>log4j-core</artifactId>
      <version>2.8.2</version>
      <scope>provided</scope>
      <exclusions>
        <exclusion>
          <groupId>org.apache.logging.log4j</groupId>
          <artifactId>log4j-api</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
    <dependency>
      <groupId>org.elasticsearch</groupId>
      <artifactId>jna</artifactId>
      <version>4.4.0</version>
      <scope>provided</scope>
    </dependency>
    
    
    
    </dependencies>
  
</project>
```
