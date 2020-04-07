# springboot简单集成kafka的配置

* 引入依赖
    ```
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
            <version>2.4.3.RELEASE</version>
        </dependency>
    ```
* 配置
    ```
    spring.kafka:
  producer:
    bootstrap-servers: 128.196.1.45:9092
    batch-size: 16785                                   #一次最多发送数据量
    retries: 1                                          #发送失败后的重复发送次数
    buffer-memory: 33554432                             #32M批处理缓冲区
    linger: 1
    properties.max.requst.size: 2097152
    key-serializer: org.apache.kafka.exceptions.serialization.StringSerializer
    value-serializer: org.apache.kafka.exceptions.serialization.StringSerializer
  consumer:
    bootstrap-servers: 128.196.1.45:9092
    group-id: group0 #设置一个默认组
    auto-offset-reset: latest                           #最早未被消费的offset earliest
    max-poll-records: 3100                              #批量消费一次最大拉取的数据量
    enable-auto-commit: false                           #是否开启自动提交
    auto-commit-interval: 1000                          #自动提交的间隔时间
    session-timeout: 20000                              #连接超时时间
    max-poll-interval: 15000                            #手动提交设置与poll的心跳数,如果消息队列中没有消息，等待毫秒后，调用poll()方法。如果队列中有消息，立即消费消息，每次消费的消息的多少可以通过max.poll.records配置。
    max-partition-fetch-bytes: 15728640                 #设置拉取数据的大小,15M
    #set comsumer max fetch.byte 2*1024*1024
    properties.max.partition.fetch.bytes: 2097152
    #key-value序列化反序列化
    key-deserializer: org.apache.kafka.exceptions.serialization.StringDeserializer
    value-deserializer: org.apache.kafka.exceptions.serialization.StringDeserializer
  listener:
    batch-listener: true                                #是否开启批量消费，true表示批量消费
    concurrencys: 2,4                                     #设置消费的线程数
    poll-timeout: 1500                                  #只限自动提交，
    topics: TranLog,test
    ```
* 生产者
    ```
    @Configuration
    @EnableKafka
    public class KafkaProducerConfig {

        @Value("${spring.kafka.producer.bootstrap-servers}")
        private String bootstrapServers;

        @Value("${spring.kafka.producer.retries}")
        private Integer retries;

        @Value("${spring.kafka.producer.batch-size}")
        private Integer batchSize;

        @Value("${spring.kafka.producer.buffer-memory}")
        private Integer bufferMemory;

        @Value("${spring.kafka.producer.linger}")
        private Integer linger;

        private Map<String, Object> producerConfigs() {
            Map<String, Object> props = new HashMap<>(7);
            props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
            props.put(ProducerConfig.RETRIES_CONFIG, retries);
            props.put(ProducerConfig.BATCH_SIZE_CONFIG, batchSize);
            props.put(ProducerConfig.LINGER_MS_CONFIG, linger);
            props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, bufferMemory);
            props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
            props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
            return props;
        }

        private ProducerFactory<String, String> producerFactory() {
            DefaultKafkaProducerFactory<String, String> producerFactory = new DefaultKafkaProducerFactory<>(producerConfigs());

            producerFactory.transactionCapable();
            producerFactory.setTransactionIdPrefix("hous-");

            return producerFactory;
        }


        @Bean
        public KafkaTransactionManager transactionManager() {
            KafkaTransactionManager manager = new KafkaTransactionManager(producerFactory());
            return manager;
        }


        @Bean
        public KafkaTemplate<String, String> kafkaTemplate() {
            return new KafkaTemplate<>(producerFactory());
        }

        @Bean
        public KafkaAdmin kafkaAdmin() {
            return new KafkaAdmin(producerConfigs());
        }

        @Bean
        public AdminClient adminClient() {
            return AdminClient.create(kafkaAdmin().getConfig());
        }
    }

    ```
    ---
    ```
    @Component
    @Slf4j
    public class KafkaSender {

        private final KafkaTemplate<String, String> KAFKA_TEMPLATE;

        @Autowired
        public KafkaSender(KafkaTemplate<String, String> kafkaTemplate) {
            this.KAFKA_TEMPLATE = kafkaTemplate;
        }

        public void sendMessage(String topic, String message){

            ListenableFuture<SendResult<String, String>> sender = KAFKA_TEMPLATE.send(new ProducerRecord<>(topic, message));
    //        //发送成功
    //        SuccessCallback successCallback = result -> log.info("数据发送成功!");
    //        //发送失败回调
    //        FailureCallback failureCallback = ex -> log.error("数据发送失败!");

            sender.addCallback(result -> {}, ex -> log.error("数据发送失败!"));
        }

    }
    ```

* 消费者
    ```
    @Configuration
    @EnableKafka
    public class KafkaConsumerConfig {

        @Value("${spring.kafka.consumer.bootstrap-servers}")
        private String bootstrapServers;

        @Value("${spring.kafka.consumer.group-id}")
        private String groupId;

        @Value("${spring.kafka.consumer.enable-auto-commit}")
        private Boolean autoCommit;

        @Value("${spring.kafka.consumer.auto-commit-interval}")
        private Integer autoCommitInterval;

        @Value("${spring.kafka.consumer.max-poll-records}")
        private Integer maxPollRecords;

        @Value("${spring.kafka.consumer.auto-offset-reset}")
        private String autoOffsetReset;

        @Value("${spring.kafka.consumer.concurrency}")
        private Integer concurrency;

        @Value("${spring.kafka.listener.poll-timeout}")
        private Long pollTimeout;

        @Value("${spring.kafka.consumer.session-timeout}")
        private String sessionTimeout;

        @Value("${spring.kafka.listener.batch-listener}")
        private Boolean batchListener;

        @Value("${spring.kafka.consumer.max-poll-interval}")
        private Integer maxPollInterval;

        @Value("${spring.kafka.consumer.max-partition-fetch-bytes}")
        private Integer maxPartitionFetchBytes;


        /**
        * 多少个并发数
        *
        * @return
        */

        @Bean
        @ConditionalOnMissingBean(name = "kafkaBatchListener")
        public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<String, String>> kafkaBatchListener3() {
            ConcurrentKafkaListenerContainerFactory<String, String> factory = kafkaListenerContainerFactory();
            factory.setConcurrency(concurrency);
            return factory;
        }

        private ConcurrentKafkaListenerContainerFactory<String, String> kafkaListenerContainerFactory() {
            ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
            factory.setConsumerFactory(consumerFactory());
            //批量消费
            factory.setBatchListener(batchListener);
            //如果消息队列中没有消息，等待timeout毫秒后，调用poll()方法。
            // 如果队列中有消息，立即消费消息，每次消费的消息的多少可以通过max.poll.records配置。
            //手动提交无需配置
            factory.getContainerProperties().setPollTimeout(pollTimeout);
            //设置提交偏移量的方式， MANUAL_IMMEDIATE 表示消费一条提交一次；MANUAL表示批量提交一次
            factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
            return factory;
        }

        private ConsumerFactory<String, String> consumerFactory() {
            return new DefaultKafkaConsumerFactory<>(consumerConfigs());
        }

        private Map<String, Object> consumerConfigs() {
            Map<String, Object> props = new HashMap<>(11);
            props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, autoCommitInterval);
            props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
            props.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
            props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, autoCommit);
            props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, maxPollRecords);
            props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, autoOffsetReset);
            props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, sessionTimeout);
            props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, maxPollInterval);
            props.put(ConsumerConfig.MAX_PARTITION_FETCH_BYTES_CONFIG, maxPartitionFetchBytes);
            props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
            props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
            return props;
        }
    }
    ```
    ---
    ```
    @Slf4j
    @Component
    public class KafkaListeners {

        @KafkaListener(containerFactory = "kafkaBatchListener",topics = {"#{'${spring.kafka.listener.topics}'.split(',')[0]}"})
        public void batchListener(List<ConsumerRecord<?,?>> records, Acknowledgment ack){
            try {
                records.forEach(record -> log.info("record.value()={}",record.value())
                );
            } catch (Exception e) {
                log.error("Kafka监听异常"+e.getMessage(),e);
            } finally {
                ack.acknowledge();//手动提交偏移量
            }

        }

    }
    ```