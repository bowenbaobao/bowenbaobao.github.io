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
