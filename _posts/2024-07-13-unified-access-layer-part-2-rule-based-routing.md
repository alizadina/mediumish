---
layout: post
title: "Unified access layer for Kafka resources: Part 2: Rule Based Routing Layer"
authors: p6,raghav
categories: [Platform Engineering, Infrastructure, Kafka, Unified Access Layer]
image: assets/blog-images/unified_access_layer/rule_based_routing_layer.png
featured: false
hidden: false
teaser: Unified access layer for Kafka resources, Part 2, Rule Based Routing Layer
toc: true
---

# Introduction

This is the Part 2 which is a follow up of [Part 1](https://platformatory.io/blog/unified-access-layer-kafka-resources-part-1) of the Unified Access Layer for Kafka Resources where we dive deep into the design and implementation of discovering Kafka resources via a rule based routing layer.

# Approach 1: Discover topics through a rule based routing layer

Just to recap, here we are trying to build the pattern where a language specific SDK handles the Kafka resources discovery by talking to a catalog service.

Let’s refer the overall architecture diagram above.

## Implementation

### Catalog Service

Catalog Service. This is implemented in the Kong API Gateway with 1/ A service that represents the Catalog Service upstream, 2/ A route “/service-locator” on the service, 3/ A custom plugin (or a pre-functions/post-functions plugin) that has a set of rules configured to retrieve the right Kafka bootstrap servers and topic information based on a set of headers (Channel, ServiceType, Organization). All these calls are protected using an API key.


### Java Implementation for Kafka Producer and Consumer

### Kafka Producer
Like we highlighted above, we need a custom consumer that can call the Catalog Service passing in the required information to locate the bootstrap servers and the topic. These will then be used to produce records (instead of the static configuration of the bootstrap servers and the topic). Here, we create a new ServiceLocatorProduder which is a facade around the KafkaProduder (routes all the calls to the internal Kafka Producer object). It first calls the Catalog Service passing the information needed and obtains the bootstrap servers and the kafka topic.

```java
public class ServiceLocatorProducer<K,V> implements Producer {
   private OkHttpClient client = new OkHttpClient();
   public static final String X_KAFKA_TOPIC_HEADER = "x-kafka-topic";
   public static final String KAFKA_TOPIC_KEY = "kafka_topic";
   public static final String X_KAFKA_BOOTSTRAP_SERVERS_HEADER = "x-kafka-bootstrap-servers";
   public static final String SERVICE_LOCATOR_URL = "http://localhost:8000/service-locator";
   public static final String APIKEY_HEADER = "apikey";
   public static final String CHANNEL_HEADER = "Channel";
   public static final String SERVICE_TYPE_HEADER = "Service-Type";
   public static final String ORGANIZATION_HEADER = "Organization";

   private String topic;

   private Map<String, String> kafkaServiceLocationMap = new HashMap<>();

   private KafkaProducer kafkaProducer;

   ServiceLocatorProducer(Properties properties) {
       try {
           this.kafkaServiceLocationMap = getServiceConfiguration();


           if (StringUtils.isEmpty(this.kafkaServiceLocationMap.get(BOOTSTRAP_SERVERS_CONFIG))) {
               throw new Exception("Unable to obtain bootstrap servers configuration.");
           }
           properties.put(BOOTSTRAP_SERVERS_CONFIG, this.kafkaServiceLocationMap.get(BOOTSTRAP_SERVERS_CONFIG));
           System.out.println(properties.get(BOOTSTRAP_SERVERS_CONFIG));

       } catch (Exception ex) {
           System.out.println("Caught exception:" + ex.getMessage() + " " + Arrays.toString(ex.getStackTrace()));
       }

       kafkaProducer = new KafkaProducer(properties);
   }


   String getTopic() {
       return this.kafkaServiceLocationMap.get(KAFKA_TOPIC_KEY);
   }


   public Map<String, String> getServiceConfiguration() throws IOException {
       Map<String, String> kafkaSvcLocMap = new HashMap<>();

       Request request = new Request.Builder()
               .url(SERVICE_LOCATOR_URL)
               .addHeader(APIKEY_HEADER, "rkey")
               .addHeader(CHANNEL_HEADER, "topic")
               .addHeader(SERVICE_TYPE_HEADER, "kafka")
               .addHeader(ORGANIZATION_HEADER, "billing")
               .build();

       Response response = client.newCall(request).execute();
       if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

       Headers responseHeaders = response.headers();
       for (int i = 0; i < responseHeaders.size(); i++) {
           System.out.println("Header: " + responseHeaders.name(i) + ": " + responseHeaders.value(i));
       }
       System.out.println(response.body().string());


       if (StringUtils.isEmpty(responseHeaders.get(X_KAFKA_TOPIC_HEADER))) throw new IOException("Could not retrieve topic name." + response);
       kafkaSvcLocMap.put(KAFKA_TOPIC_KEY, responseHeaders.get(X_KAFKA_TOPIC_HEADER));

       if (StringUtils.isEmpty(responseHeaders.get(X_KAFKA_BOOTSTRAP_SERVERS_HEADER))) throw new IOException("Could not retrieve bootstrap servers." + response);
       kafkaSvcLocMap.put(BOOTSTRAP_SERVERS_CONFIG, responseHeaders.get(X_KAFKA_BOOTSTRAP_SERVERS_HEADER));

       return kafkaSvcLocMap;
   }

```

### Kafka Producer App

Typically, the producer application would include all the properties needed to successfully publish records to the Kafka cluster (especially the 2 key things: 1/ Bootstrap servers, 2/ Topic). Like shown below, the ServiceLocatorProducer does the magic to get the right bootstrap servers and topic (instead of the commented out static information).

```java
public class App {

   public static void main(String[] args ) throws UnknownHostException {
      
       Properties config = new Properties();

       // Typically, the line below is uncommented
       //config.put(BOOTSTRAP_SERVERS_CONFIG, "localhost:9092,localhost:9093");
       config.put(ACKS_CONFIG, "all");
       config.put(KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getCanonicalName());
       config.put(VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getCanonicalName());

       // Typically, a topic is supplied through a static configuration
       //String topic = "users";
       String [] keys = {"joe", "jill", "justie"};
       String [] values = {"crease", "myers", "hill"};

       try (final ServiceLocatorProducer<String, String> producer = new ServiceLocatorProducer<>(config)) {
           final Random rnd = new Random();
           final int numMessages = 10;
           for (int i = 0; i < numMessages; i++) {
               String user = keys[rnd.nextInt(keys.length)];
               String item = values[rnd.nextInt(values.length)];
               String topic = producer.getTopic();

               producer.send(
                       new ProducerRecord<>(topic, user, item),
                       (event, ex) -> {
                           if (ex != null)
                               ex.printStackTrace();
                           else
                               System.out.printf("Produced event to topic %s: key = %-10s value = %s%n", topic, user, item);
                       });
           }
           System.out.printf("%s events were produced to topic %s%n", numMessages, producer.getTopic());
       }
   }
}

```

### Kafka Producer in Action
```
C:\Users\nragh\.jdks\corretto-11.0.22\bin\java.exe "-javaagent:C:\Program Files\JetBrains\IntelliJ IDEA Community Edition 2023.2.2\lib\idea_rt.jar=56323:C:\Program Files\JetBrains\IntelliJ IDEA Community Edition 2023.2.2\bin" -Dfile.encoding=UTF-8 -classpath C:\Users\nragh\IdeaProjects\kafka-producer\target\classes;C:\Users\nragh\.m2\repository\org\apache\kafka\kafka-clients\7.0.1-ccs\kafka-clients-7.0.1-ccs.jar;C:\Users\nragh\.m2\repository\com\github\luben\zstd-jni\1.5.0-2\zstd-jni-1.5.0-2.jar;C:\Users\nragh\.m2\repository\org\lz4\lz4-java\1.7.1\lz4-java-1.7.1.jar;C:\Users\nragh\.m2\repository\org\xerial\snappy\snappy-java\1.1.8.1\snappy-java-1.1.8.1.jar;C:\Users\nragh\.m2\repository\org\slf4j\slf4j-api\1.7.30\slf4j-api-1.7.30.jar;C:\Users\nragh\.m2\repository\org\apache\httpcomponents\httpclient\4.5.14\httpclient-4.5.14.jar;C:\Users\nragh\.m2\repository\org\apache\httpcomponents\httpcore\4.4.16\httpcore-4.4.16.jar;C:\Users\nragh\.m2\repository\commons-logging\commons-logging\1.2\commons-logging-1.2.jar;C:\Users\nragh\.m2\repository\commons-codec\commons-codec\1.11\commons-codec-1.11.jar;C:\Users\nragh\.m2\repository\com\squareup\okhttp\okhttp\2.7.5\okhttp-2.7.5.jar;C:\Users\nragh\.m2\repository\com\squareup\okio\okio\1.6.0\okio-1.6.0.jar;C:\Users\nragh\.m2\repository\com\squareup\okhttp3\okhttp\3.14.9\okhttp-3.14.9.jar;C:\Users\nragh\.m2\repository\org\apache\commons\commons-lang3\3.14.0\commons-lang3-3.14.0.jar com.platformatory.App
Header: Content-Type: application/json
Header: Content-Length: 787
Header: Connection: keep-alive
Header: RateLimit-Remaining: 3
Header: RateLimit-Reset: 51
Header: RateLimit-Limit: 5
Header: Server: gunicorn/19.9.0
Header: Date: Tue, 09 Jul 2024 07:46:09 GMT
Header: Access-Control-Allow-Origin: *
Header: Access-Control-Allow-Credentials: true
Header: X-Kafka-Bootstrap-Servers: localhost:9092,localhost:9093,localhost:9094
Header: X-Kafka-Topic: users
Header: X-Rllm: 5
Header: X-Rlrm: 3
Header: My-Custom-Proxy-Latency: 3
Header: My-Custom-Upstream-Latency: 30
Header: X-Kong-Upstream-Latency: 30
Header: X-Kong-Proxy-Latency: 3
Header: Via: kong/3.6.1
Header: X-Kong-Request-Id: 7716083bb8586e3c2e952a1ce463bc60
Header: OkHttp-Sent-Millis: 1720511169779
Header: OkHttp-Received-Millis: 1720511169817
{
  "args": {}, 
  "data": "", 
  "files": {}, 
  "form": {}, 
  "headers": {
    "Accept-Encoding": "gzip", 
    "Apikey": "rkey", 
    "Channel": "topic", 
    "Connection": "close", 
    "Content-Length": "0", 
    "Host": "host.docker.internal", 
    "Organization": "billing", 
    "Service-Type": "kafka", 
    "User-Agent": "okhttp/2.7.5", 
    "X-Consumer-Id": "f8c11202-f83a-4b32-b29b-4645cbd79a1f", 
    "X-Consumer-Username": "*******", 
    "X-Credential-Identifier": "83cdfb75-1538-4b1a-b5ad-2aca27f91764", 
    "X-Forwarded-Host": "localhost", 
    "X-Forwarded-Path": "/service-locator", 
    "X-Kong-Request-Id": "7716083bb8586e3c2e952a1ce463bc60"
  }, 
  "json": null, 
  "method": "GET", 
  "origin": "172.1.1.1", 
  "url": "http://localhost/anything/service-locator"
}

10 events were produced to topic users
Produced event to topic users: key = joe        value = hill
Produced event to topic users: key = joe        value = hill
Produced event to topic users: key = jill       value = hill
Produced event to topic users: key = jill       value = crease
Produced event to topic users: key = jill       value = hill
Produced event to topic users: key = jill       value = crease
Produced event to topic users: key = justie     value = crease
Produced event to topic users: key = justie     value = crease
Produced event to topic users: key = justie     value = crease
Produced event to topic users: key = justie     value = hill

```

### Kafka Consumer
Just like the facade created by the Kafka Producer, we do the same here called the ServiceLocatorConsumer, which routes all the calls to the internal Kafka Consumer object. Also, just like the ServiceLocatorProducer, it calls the Catalog Service and obtains the same 2 key properties needed to consume data.

```java
public class ServiceLocatorConsumer<K, V> implements Consumer<K, V> {
   private OkHttpClient client = new OkHttpClient();
   public static final String X_KAFKA_TOPIC_HEADER = "x-kafka-topic";
   public static final String KAFKA_TOPIC_KEY = "kafka_topic";
   public static final String X_KAFKA_BOOTSTRAP_SERVERS_HEADER = "x-kafka-bootstrap-servers";
   public static final String SERVICE_LOCATOR_URL = "http://localhost:8000/service-locator";
   public static final String APIKEY_HEADER = "apikey";
   public static final String CHANNEL_HEADER = "Channel";
   public static final String SERVICE_TYPE_HEADER = "Service-Type";
   public static final String ORGANIZATION_HEADER = "Organization";

   private Map<String, String> kafkaServiceLocationMap = new HashMap<>();

   private KafkaConsumer kafkaConsumer;

   public ServiceLocatorConsumer(Map<String, Object> configs) {
       kafkaConsumer = new KafkaConsumer<>(configs);
   }

   private String topic;

   public ServiceLocatorConsumer(Properties properties) {
       try {
           this.kafkaServiceLocationMap = getServiceConfiguration();


           if (StringUtils.isEmpty(this.kafkaServiceLocationMap.get(BOOTSTRAP_SERVERS_CONFIG))) {
               throw new Exception("Unable to obtain bootstrap servers configuration.");
           }
           properties.put(BOOTSTRAP_SERVERS_CONFIG, this.kafkaServiceLocationMap.get(BOOTSTRAP_SERVERS_CONFIG));
           System.out.println(properties.get(BOOTSTRAP_SERVERS_CONFIG));

       } catch (Exception ex) {
           System.out.println("Caught exception:" + ex.getMessage() + " " + Arrays.toString(ex.getStackTrace()));
       }
       kafkaConsumer = new KafkaConsumer<>(properties);
   }
   public Map<String, String> getServiceConfiguration() throws IOException {
       Map<String, String> kafkaSvcLocMap = new HashMap<>();

       Request request = new Request.Builder()
               .url(SERVICE_LOCATOR_URL)
               .addHeader(APIKEY_HEADER, "rkey")
               .addHeader(CHANNEL_HEADER, "topic")
               .addHeader(SERVICE_TYPE_HEADER, "kafka")
               .addHeader(ORGANIZATION_HEADER, "billing")
               .build();

       Response response = client.newCall(request).execute();
       if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);


       Headers responseHeaders = response.headers();
       for (int i = 0; i < responseHeaders.size(); i++) {
           System.out.println("Header: " + responseHeaders.name(i) + ": " + responseHeaders.value(i));
       }
       System.out.println(response.body().string());

       if (StringUtils.isEmpty(responseHeaders.get(X_KAFKA_TOPIC_HEADER))) throw new IOException("Could not retrieve topic name." + response);
       kafkaSvcLocMap.put(KAFKA_TOPIC_KEY, responseHeaders.get(X_KAFKA_TOPIC_HEADER));


       if (StringUtils.isEmpty(responseHeaders.get(X_KAFKA_BOOTSTRAP_SERVERS_HEADER))) throw new IOException("Could not retrieve bootstrap servers." + response);
       kafkaSvcLocMap.put(BOOTSTRAP_SERVERS_CONFIG, responseHeaders.get(X_KAFKA_BOOTSTRAP_SERVERS_HEADER));

       return kafkaSvcLocMap;
   }

   public String getTopic() {
       return this.kafkaServiceLocationMap.get(KAFKA_TOPIC_KEY);
   }

```

### Kafka Consumer App
Just like the Kafka Producer App, the Consumer App instantiates the ServiceLocatorConsumer and seamlessly starts consuming from the obtained topic. You can see how the commented out code for the bootstrap servers and the topic show the dynamic nature of the whole system.

```java
public class App
{
   public static void main( String[] args ) {
       Properties config = new Properties();
       try {
           config.put("client.id", InetAddress.getLocalHost().getHostName());
       } catch (UnknownHostException e) {
           throw new RuntimeException(e);
       }
       // Typically, this is supplied via a configuration property at build time
       //config.put(BOOTSTRAP_SERVERS_CONFIG, "localhost:9092,localhost:9093,localhost:9094");
       config.put(AUTO_OFFSET_RESET_CONFIG, "earliest");
       config.put(GROUP_ID_CONFIG, "kafka-java-consumer");
       config.put(KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getCanonicalName());
       config.put(VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getCanonicalName());


       // Typically, the topic is supplied at build time
       //final String topic = "users";

       try (final ServiceLocatorConsumer consumer = new ServiceLocatorConsumer<>(config)) {
           consumer.subscribe(Arrays.asList(consumer.getTopic()));
           while (true) {
               ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
               for (ConsumerRecord<String, String> record : records) {
                   String key = record.key();
                   String value = record.value();
                   String topic = record.topic();
                   System.out.println(
                           String.format("Consumed event from topic %s: key = %-10s value = %s", topic, key, value));
               }
           }
       }
   }

```

### Kafka Consumer in Action
```
C:\Users\nragh\.jdks\corretto-11.0.22\bin\java.exe "-javaagent:C:\Program Files\JetBrains\IntelliJ IDEA Community Edition 2023.2.2\lib\idea_rt.jar=56307:C:\Program Files\JetBrains\IntelliJ IDEA Community Edition 2023.2.2\bin" -Dfile.encoding=UTF-8 -classpath C:\Users\nragh\IdeaProjects\kakfa-consumer\target\classes;C:\Users\nragh\.m2\repository\org\apache\kafka\kafka-clients\7.6.1-ce\kafka-clients-7.6.1-ce.jar;C:\Users\nragh\.m2\repository\io\confluent\telemetry-events-api\7.6.1-ce\telemetry-events-api-7.6.1-ce.jar;C:\Users\nragh\.m2\repository\com\github\luben\zstd-jni\1.5.5-1\zstd-jni-1.5.5-1.jar;C:\Users\nragh\.m2\repository\org\lz4\lz4-java\1.8.0\lz4-java-1.8.0.jar;C:\Users\nragh\.m2\repository\org\xerial\snappy\snappy-java\1.1.10.5\snappy-java-1.1.10.5.jar;C:\Users\nragh\.m2\repository\org\slf4j\slf4j-api\1.7.36\slf4j-api-1.7.36.jar;C:\Users\nragh\.m2\repository\org\apache\httpcomponents\httpclient\4.5.14\httpclient-4.5.14.jar;C:\Users\nragh\.m2\repository\org\apache\httpcomponents\httpcore\4.4.16\httpcore-4.4.16.jar;C:\Users\nragh\.m2\repository\commons-logging\commons-logging\1.2\commons-logging-1.2.jar;C:\Users\nragh\.m2\repository\commons-codec\commons-codec\1.11\commons-codec-1.11.jar;C:\Users\nragh\.m2\repository\com\fasterxml\jackson\core\jackson-databind\2.11.0\jackson-databind-2.11.0.jar;C:\Users\nragh\.m2\repository\com\fasterxml\jackson\core\jackson-annotations\2.11.0\jackson-annotations-2.11.0.jar;C:\Users\nragh\.m2\repository\com\fasterxml\jackson\core\jackson-core\2.11.0\jackson-core-2.11.0.jar;C:\Users\nragh\.m2\repository\org\apache\httpcomponents\client5\httpclient5\5.0.1\httpclient5-5.0.1.jar;C:\Users\nragh\.m2\repository\org\apache\httpcomponents\core5\httpcore5\5.0.1\httpcore5-5.0.1.jar;C:\Users\nragh\.m2\repository\org\apache\httpcomponents\core5\httpcore5-h2\5.0.1\httpcore5-h2-5.0.1.jar;C:\Users\nragh\.m2\repository\com\squareup\okhttp3\okhttp\3.14.9\okhttp-3.14.9.jar;C:\Users\nragh\.m2\repository\com\squareup\okio\okio\1.17.2\okio-1.17.2.jar;C:\Users\nragh\.m2\repository\com\squareup\retrofit2\retrofit\2.7.2\retrofit-2.7.2.jar;C:\Users\nragh\.m2\repository\com\squareup\retrofit2\converter-jackson\2.7.2\converter-jackson-2.7.2.jar;C:\Users\nragh\.m2\repository\com\squareup\okhttp\okhttp\2.7.5\okhttp-2.7.5.jar;C:\Users\nragh\.m2\repository\org\apache\commons\commons-lang3\3.14.0\commons-lang3-3.14.0.jar com.platformatory.App
Header: Content-Type: application/json
Header: Content-Length: 787
Header: Connection: keep-alive
Header: RateLimit-Remaining: 4
Header: RateLimit-Reset: 58
Header: RateLimit-Limit: 5
Header: Server: gunicorn/19.9.0
Header: Date: Tue, 09 Jul 2024 07:46:02 GMT
Header: Access-Control-Allow-Origin: *
Header: Access-Control-Allow-Credentials: true
Header: X-Kafka-Bootstrap-Servers: localhost:9092,localhost:9093,localhost:9094
Header: X-Kafka-Topic: users
Header: X-Kong-Upstream-Latency: 27
Header: X-Kong-Proxy-Latency: 3
Header: Via: kong/3.6.1
Header: X-Kong-Request-Id: b00de779f26f42343588ee444afd0020
Header: OkHttp-Sent-Millis: 1720511162677
Header: OkHttp-Received-Millis: 1720511162714
{
  "args": {}, 
  "data": "", 
  "files": {}, 
  "form": {}, 
  "headers": {
    "Accept-Encoding": "gzip", 
    "Apikey": "********", 
    "Channel": "topic", 
    "Connection": "close", 
    "Content-Length": "0", 
    "Host": "host.docker.internal", 
    "Organization": "billing", 
    "Service-Type": "kafka", 
    "User-Agent": "okhttp/2.7.5", 
    "X-Consumer-Id": "f8c11202-f83a-4b32-b29b-4645cbd79a1f", 
    "X-Consumer-Username": "*******", 
    "X-Credential-Identifier": "83cdfb75-1538-4b1a-b5ad-2aca27f91764", 
    "X-Forwarded-Host": "localhost", 
    "X-Forwarded-Path": "/service-locator", 
    "X-Kong-Request-Id": "b00de779f26f42343588ee444afd0020"
  }, 
  "json": null, 
  "method": "GET", 
  "origin": "172.1.1.1", 
  "url": "http://localhost/anything/service-locator"
}

Consumed event from topic users: key = justie     value = crease
Consumed event from topic users: key = justie     value = crease
Consumed event from topic users: key = justie     value = crease
Consumed event from topic users: key = justie     value = hill
Consumed event from topic users: key = jill       value = hill
Consumed event from topic users: key = jill       value = crease
Consumed event from topic users: key = jill       value = hill
Consumed event from topic users: key = jill       value = crease
Consumed event from topic users: key = joe        value = hill
Consumed event from topic users: key = joe        value = hill

```

## Python Implementation
Coming soon...


# Conclusion

Creating a system that helps a centralized Kafka infrastructure team to easily create, label and vend information reduces common problems and dependencies. Same benefits are passed on to the producers and consumers thus creating a scalable system/organization. 
