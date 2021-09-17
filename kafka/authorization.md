# Kafka authorization 建立

啟動zookeeper，先不要啟用kafka server。
1. 創建使用者
```=
$ bin/kafka-configs.sh --zookeeper host:port --alter --add-config 'SCRAM-SHA-256=[iterations=8192,password=belstart123],SCRAM-SHA-512=[password=belstar123]' --entity-type users --entity-name mia 
```

2. Broker配置
建立jaas檔案ex. kafka_server_jaas.conf
```=
KafkaServer {
    
org.apache.kafka.common.security.scram.ScramLoginModule required
username="admin"
password="admin";
};
KafkaServer {
    org.apache.kafka.common.security.scram.ScramLoginModule required
    username="admin"
    password="belstar123"
};
```

配置server.properties，加入以下配置：
```
listeners=SASL_PLAINTEXT://localhost:9092
security.inter.broker.protocol=SASL_PLAINTEXT
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
sasl.enabled.mechanisms=SCRAM-SHA-512
```

將JAAS配置檔案作為參數傳遞給kafak
```
$ KAFKA_OPTS=-Djava.security.auth.login.config=/PATH/kafka_server_jaas.conf bin/kafka-server-start.sh config/server.properties
```

3. KafkaJs 使用
建立kafka instance時透過sasl加入驗證
```javascript=
new Kafka({
  clientId: 'my-app',
  brokers: ['localhost:9092'],
  sasl: {
    mechanism: 'SCRAM-SHA-512',
    username: 'admin',
    password: 'belstar123',
  }
})
```

4. 額外設置用戶權限
在server.properties修改
```
allow.everyone.if.no.acl.found=false
```
新增super user
```
super.users=User:admin
```

透過kafka-acls.sh增加/刪除 ACL
```
$ bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:Bob --allow-principal User:Alice --allow-host 198.51.100.0 --allow-host 198.51.100.1 --operation Read --operation Write --topic Test-topic
```
[詳細設置請參考](https://kafka.apache.org/documentation/#security_authz_examples)