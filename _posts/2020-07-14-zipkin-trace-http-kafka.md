---
layout: post
title: "zipkin跟踪Httpclient,OkHttp3,Kafka方式"
subtitle: ''
author: "Zerosbg"
header-style: text
tags:
  - zipkin
  - HttpClient
  - Okhttp3
  - kafka
---

zipkin针对三种类别的代码做跟踪,沿用brave的instrument系列

* Apache httpClient

  针对`HttpClient` 形式我们需要用到如下依赖

  ```yml
  <dependency>
      <groupId>io.zipkin.brave</groupId>
      <artifactId>brave-instrumentation-httpclient</artifactId>
  </dependency>
  ```

  在创建HttpClient对象的时候我们可以用`TracingHttpClientBuilder`来嵌套创建CloseableHttpClient

  非brave版本:

  ```java
  CloseableHttpClient client = HttpClientBuilder.create().build();
  ```

  使用brave的版本:

  ```java
  HttpClientBuilder builder = TracingHttpClientBuilder.create(tracing).build();
  ```

* OkHttp3

  在使用OkHttp3的情况下我们需要添加依赖如下:

  ```yml
  <dependency>
       <groupId>io.zipkin.brave</groupId>
       <artifactId>brave-instrumentation-okhttp3</artifactId>
  </dependency>
  ```

  在创建OkHttpClient的时候我们需要添加dispatcher和拦截器TracingInterceptor

  ```java
  new OkHttpClient.Builder()
                .dispatcher(new Dispatcher(httpTracing.tracing().currentTraceContext().executorService(new Dispatcher().executorService())))
                .addNetworkInterceptor(TracingInterceptor.create(httpTracing))
                .build();
  ```

* kafka

  在使用kafka的时候我们一般都是批量获取数据`poll()`然后循环解析数据.brave也提供了对应的工具包

  ```yml
   <dependency>
       <groupId>io.zipkin.brave</groupId>
       <artifactId>brave-instrumentation-kafka-clients</artifactId>
   </dependency>
  ```

  `brave-instrumentation-kafka-clients` 包中提供了`KafkaTracing`用于修饰`KafkaConsumer` 和`KafkaProducer`

  ```java
  /** 封装 Kafka producer **/
  Producer<String, String> producer = kafkaTracing.producer(new KafkaProducer<>(props));
  /** 封装 Kafka Consumer **/
  Consumer<String, String> consumer = new KafkaConsumer<>(properties);
  Consumer<String, String> tracingConsumer = kafkaTracing.consumer(consumer);
  ```

  同时由于在消费Kafka的时候是批量消费的,所以我们针对每一个子流程都需要创建一个`span`, 做如下封装

  ```java
  <K, V> void process(ConsumerRecords<K, V> record) {
    // Grab any span from the record. The topic and key are automatically tagged
    Span span = kafkaTracing.nextSpan(record).name("process").start();
  
    // Below is the same setup as any synchronous tracing
    try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) { // so logging can see trace ID
      return doProcess(record); // do the actual work
    } catch (RuntimeException | Error e) {
      span.error(e); // make sure any error gets into the span before it is finished
      throw e;
    } finally {
      span.finish(); // ensure the span representing this processing completes.
    }
  }
  ```

  在这个方法中我们需要根据当前的record创建一个新的span.

