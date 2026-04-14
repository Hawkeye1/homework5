# Практическая работа 6: Настройка защищённого соединения и управление доступом.

**Цель задания:** настроить защищённое SSL-соединение для кластера Apache Kafka из трёх брокеров с использованием Docker Compose, создать новый топик и протестировать отправку и получение зашифрованных сообщений.

## Задание:
1. Создайте сертификаты для каждого брокера.
2. Создайте Truststore и Keystore для каждого брокера.
3. Настройте дополнительные брокеры в режиме SSL. В первом модуле курса вы уже работали с кластером Kafka, состоящим из трёх брокеров. Используйте имеющийся docker-compose кластера и настройте для него SSL.
4. **Создать два топика:**
   * topic-1
   * topic-2
5. **Настроить права доступа:**
   * topic-1: Доступен как для продюсеров, так и для консьюмеров.
   * topic-2: Продюсеры могут отправлять сообщения. Консьюмеры не имеют доступа к чтению данных.
   

## Процесс создания сертификатов
1. Создадим новый конфигурационный файл ```ca.cnf```
```
[ policy_match ]
countryName = match
stateOrProvinceName = match
organizationName = match
organizationalUnitName = optional
commonName = supplied
emailAddress = optional


[ req ]
prompt = no
distinguished_name = dn
default_md = sha256
default_bits = 4096
x509_extensions = v3_ca


[ dn ]
countryName = BY
organizationName = Home
organizationalUnitName = Practice
localityName = Minsk
commonName = home-practice-kafka-ca


[ v3_ca ]
subjectKeyIdentifier = hash
basicConstraints = critical,CA:true
authorityKeyIdentifier = keyid:always,issuer:always
keyUsage = critical,keyCertSign,cRLSign
```
2. Создадим новый сертификатный запрос без шифрования приватного ключа:
```bash
openssl req -new -nodes -x509 -days 365 -newkey rsa:2048 -keyout ca.key -out ca.crt -config ca.cnf
```
3. Создадим файл для хранения сертификата безопасности. Объединим сертификат CA (ca.crt) и его ключ (ca.key) в один файл ca.pem:
```bash
cat ca.crt ca.key > ca.pem
```
4. Создадим файл конфигурации для брокеров.
    - Создадим новый конфигурационный файл ```kafka-0-creds/kafka-0.cnf```:
    ```
    [req]
    prompt = no
    distinguished_name = dn
    default_md = sha256
    default_bits = 4096
    req_extensions = v3_req


    [ dn ]
    countryName = BY
    organizationName = Home
    organizationalUnitName = Practice
    localityName = Minsk
    commonName = kafka-0


    [ v3_ca ]
    subjectKeyIdentifier = hash
    basicConstraints = critical,CA:true
    authorityKeyIdentifier = keyid:always,issuer:always
    keyUsage = critical,keyCertSign,cRLSign


    [ v3_req ]
    subjectKeyIdentifier = hash
    basicConstraints = CA:FALSE
    nsComment = "OpenSSL Generated Certificate"
    keyUsage = critical, digitalSignature, keyEncipherment
    extendedKeyUsage = serverAuth, clientAuth
    subjectAltName = @alt_names


    [ alt_names ]
    DNS.1 = kafka-0
    DNS.2 = kafka-0-external
    DNS.3 = localhost
    ```
    - Создадим новый конфигурационный файл ```kafka-1-creds/kafka-1.cnf```:
    ```
    [req]
    prompt = no
    distinguished_name = dn
    default_md = sha256
    default_bits = 4096
    req_extensions = v3_req


    [ dn ]
    countryName = BY
    organizationName = Home
    organizationalUnitName = Practice
    localityName = Minsk
    commonName = kafka-1


    [ v3_ca ]
    subjectKeyIdentifier = hash
    basicConstraints = critical,CA:true
    authorityKeyIdentifier = keyid:always,issuer:always
    keyUsage = critical,keyCertSign,cRLSign


    [ v3_req ]
    subjectKeyIdentifier = hash
    basicConstraints = CA:FALSE
    nsComment = "OpenSSL Generated Certificate"
    keyUsage = critical, digitalSignature, keyEncipherment
    extendedKeyUsage = serverAuth, clientAuth
    subjectAltName = @alt_names


    [ alt_names ]
    DNS.1 = kafka-1
    DNS.2 = kafka-1-external
    DNS.3 = localhost
    ```
    - Создадим новый конфигурационный файл ```kafka-2-creds/kafka-2.cnf```:
    ```
    [req]
    prompt = no
    distinguished_name = dn
    default_md = sha256
    default_bits = 4096
    req_extensions = v3_req


    [ dn ]
    countryName = BY
    organizationName = Home
    organizationalUnitName = Practice
    localityName = Minsk
    commonName = kafka-2


    [ v3_ca ]
    subjectKeyIdentifier = hash
    basicConstraints = critical,CA:true
    authorityKeyIdentifier = keyid:always,issuer:always
    keyUsage = critical,keyCertSign,cRLSign


    [ v3_req ]
    subjectKeyIdentifier = hash
    basicConstraints = CA:FALSE
    nsComment = "OpenSSL Generated Certificate"
    keyUsage = critical, digitalSignature, keyEncipherment
    extendedKeyUsage = serverAuth, clientAuth
    subjectAltName = @alt_names


    [ alt_names ]
    DNS.1 = kafka-2
    DNS.2 = kafka-2-external
    DNS.3 = localhost
    ```
5. Создадим приватные ключи и запросы на сертификат (CSR)
```bash
openssl req -new \
    -newkey rsa:2048 \
    -keyout kafka-0-creds/kafka-0.key \
    -out kafka-0-creds/kafka-0.csr \
    -config kafka-0-creds/kafka-0.cnf \
    -nodes 
    
openssl req -new \
    -newkey rsa:2048 \
    -keyout kafka-1-creds/kafka-1.key \
    -out kafka-1-creds/kafka-1.csr \
    -config kafka-1-creds/kafka-1.cnf \
    -nodes 
    
openssl req -new \
    -newkey rsa:2048 \
    -keyout kafka-2-creds/kafka-2.key \
    -out kafka-2-creds/kafka-2.csr \
    -config kafka-2-creds/kafka-2.cnf \
    -nodes
```
6. Создадим сертификаты брокеров, подписанные CA
```bash
openssl x509 -req \
    -days 3650 \
    -in kafka-0-creds/kafka-0.csr \
    -CA ca.crt \
    -CAkey ca.key \
    -CAcreateserial \
    -out kafka-0-creds/kafka-0.crt \
    -extfile kafka-0-creds/kafka-0.cnf \
    -extensions v3_req
    
openssl x509 -req \
    -days 3650 \
    -in kafka-1-creds/kafka-1.csr \
    -CA ca.crt \
    -CAkey ca.key \
    -CAcreateserial \
    -out kafka-1-creds/kafka-1.crt \
    -extfile kafka-1-creds/kafka-1.cnf \
    -extensions v3_req    
    
openssl x509 -req \
    -days 3650 \
    -in kafka-2-creds/kafka-2.csr \
    -CA ca.crt \
    -CAkey ca.key \
    -CAcreateserial \
    -out kafka-2-creds/kafka-2.crt \
    -extfile kafka-2-creds/kafka-2.cnf \
    -extensions v3_req 
```
7. Создадим PKCS12-хранилище с сертификатом брокера:
```bash
openssl pkcs12 -export \
    -in kafka-0-creds/kafka-0.crt \
    -inkey kafka-0-creds/kafka-0.key \
    -chain \
    -CAfile ca.pem \
    -name kafka-0 \
    -out kafka-0-creds/kafka-0.p12 \
    -password pass:password
    
openssl pkcs12 -export \
    -in kafka-1-creds/kafka-1.crt \
    -inkey kafka-1-creds/kafka-1.key \
    -chain \
    -CAfile ca.pem \
    -name kafka-1 \
    -out kafka-1-creds/kafka-1.p12 \
    -password pass:password    

openssl pkcs12 -export \
    -in kafka-2-creds/kafka-2.crt \
    -inkey kafka-2-creds/kafka-2.key \
    -chain \
    -CAfile ca.pem \
    -name kafka-2 \
    -out kafka-2-creds/kafka-2.p12 \
    -password pass:password
```
8. Создадим keystore для Kafka:
```bash
keytool -importkeystore \
    -deststorepass password \
    -destkeystore kafka-0-creds/kafka.kafka-0.keystore.pkcs12 \
    -srckeystore kafka-0-creds/kafka-0.p12 \
    -deststoretype PKCS12  \
    -srcstoretype PKCS12 \
    -noprompt \
    -srcstorepass password     
    
keytool -importkeystore \
    -deststorepass password \
    -destkeystore kafka-1-creds/kafka.kafka-1.keystore.pkcs12 \
    -srckeystore kafka-1-creds/kafka-1.p12 \
    -deststoretype PKCS12  \
    -srcstoretype PKCS12 \
    -noprompt \
    -srcstorepass password    
    
keytool -importkeystore \
    -deststorepass password \
    -destkeystore kafka-2-creds/kafka.kafka-2.keystore.pkcs12 \
    -srckeystore kafka-2-creds/kafka-2.p12 \
    -deststoretype PKCS12  \
    -srcstoretype PKCS12 \
    -noprompt \
    -srcstorepass password
```
9. Создадим truststore для Kafka:
```bash
keytool -import \
    -file ca.crt \
    -alias ca \
    -keystore kafka-0-creds/kafka.kafka-0.truststore.jks \
    -storepass password \
    -noprompt    
    
keytool -import \
    -file ca.crt \
    -alias ca \
    -keystore kafka-1-creds/kafka.kafka-1.truststore.jks \
    -storepass password \
    -noprompt    
    
keytool -import \
    -file ca.crt \
    -alias ca \
    -keystore kafka-2-creds/kafka.kafka-2.truststore.jks \
    -storepass password \
    -noprompt 
```
10. Сохраним пароли:
```bash
echo "password" > kafka-0-creds/kafka-0_sslkey_creds
echo "password" > kafka-0-creds/kafka-0_keystore_creds
echo "password" > kafka-0-creds/kafka-0_truststore_creds

echo "password" > kafka-1-creds/kafka-1_sslkey_creds
echo "password" > kafka-1-creds/kafka-1_keystore_creds
echo "password" > kafka-1-creds/kafka-1_truststore_creds

echo "password" > kafka-2-creds/kafka-2_sslkey_creds
echo "password" > kafka-2-creds/kafka-2_keystore_creds
echo "password" > kafka-2-creds/kafka-2_truststore_creds
```
11. Импортируем PKCS12 в JKS:
```bash
keytool -importkeystore \
    -srckeystore kafka-0-creds/kafka-0.p12 \
    -srcstoretype PKCS12 \
    -destkeystore kafka-0-creds/kafka-0.keystore.jks \
    -deststoretype JKS \
    -deststorepass password
    
keytool -importkeystore \
    -srckeystore kafka-1-creds/kafka-1.p12 \
    -srcstoretype PKCS12 \
    -destkeystore kafka-1-creds/kafka-1.keystore.jks \
    -deststoretype JKS \
    -deststorepass password    
    
keytool -importkeystore \
    -srckeystore kafka-2-creds/kafka-2.p12 \
    -srcstoretype PKCS12 \
    -destkeystore kafka-2-creds/kafka-2.keystore.jks \
    -deststoretype JKS \
    -deststorepass password
```
12. Импортируем CA в Truststore:
```bash
keytool -import -trustcacerts -file ca.crt \
    -keystore kafka-0-creds/kafka-0.truststore.jks \
    -storepass password -noprompt -alias ca     
    
keytool -import -trustcacerts -file ca.crt \
    -keystore kafka-1-creds/kafka-1.truststore.jks \
    -storepass password -noprompt -alias ca    
    
keytool -import -trustcacerts -file ca.crt \
    -keystore kafka-2-creds/kafka-2.truststore.jks \
    -storepass password -noprompt -alias ca 
```


## Выполнение задания:
1. Запустим Docker compose (локальный терминал):
```bash
docker compose up -d
```
2. Создадим топик ```topic-1``` (локальный терминал):
```bash
docker exec kafka-0 kafka-topics --bootstrap-server kafka-0:9092 \
  --create --topic topic-1 --partitions 3 --replication-factor 3 \
  --command-config /etc/kafka/secrets/adminclient-configs.conf
```
Ожидаемый результат:
```
Created topic topic-1.
```
3. Создадим топик ```topic-2``` (локальный терминал):
```bash
docker exec kafka-0 kafka-topics --bootstrap-server kafka-0:9092 \
  --create --topic topic-2 --partitions 3 --replication-factor 3 \
  --command-config /etc/kafka/secrets/adminclient-configs.conf
```
Ожидаемый результат:
```
Created topic topic-2.
```
4. Добавим права на запись в **topic-1** для ```producer``` (локальный терминал):
```bash
docker exec kafka-0 kafka-acls --bootstrap-server kafka-0:9092 \
  --command-config /etc/kafka/secrets/adminclient-configs.conf \
  --add --allow-principal "User:producer" --operation Write --operation Describe --topic topic-1
```
Ожидаемый результат:
```
Adding ACLs for resource `ResourcePattern(resourceType=TOPIC, name=topic-1, patternType=LITERAL)`: 
 	(principal=User:producer, host=*, operation=DESCRIBE, permissionType=ALLOW)
	(principal=User:producer, host=*, operation=WRITE, permissionType=ALLOW) 

Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=topic-1, patternType=LITERAL)`: 
 	(principal=User:producer, host=*, operation=DESCRIBE, permissionType=ALLOW)
	(principal=User:producer, host=*, operation=WRITE, permissionType=ALLOW)
```
5. Добавим права на запись в **topic-2** для ```producer``` (локальный терминал):
```bash
docker exec kafka-0 kafka-acls --bootstrap-server kafka-0:9092 \
  --command-config /etc/kafka/secrets/adminclient-configs.conf \
  --add --allow-principal "User:producer" --operation Write --operation Describe --topic topic-2
```
Ожидаемый результат:
```
Adding ACLs for resource `ResourcePattern(resourceType=TOPIC, name=topic-2, patternType=LITERAL)`: 
 	(principal=User:producer, host=*, operation=DESCRIBE, permissionType=ALLOW)
	(principal=User:producer, host=*, operation=WRITE, permissionType=ALLOW) 

Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=topic-2, patternType=LITERAL)`: 
 	(principal=User:producer, host=*, operation=WRITE, permissionType=ALLOW)
	(principal=User:producer, host=*, operation=DESCRIBE, permissionType=ALLOW) 
```
6. Добавим права на чтение из **topic-1** для ```consumer``` (локальный терминал):
```bash
docker exec kafka-0 kafka-acls --bootstrap-server kafka-0:9092 \
  --command-config /etc/kafka/secrets/adminclient-configs.conf \
  --add --allow-principal "User:consumer" --operation Read -operation Describe --topic topic-1    
```
Ожидаемый результат:
```
Adding ACLs for resource `ResourcePattern(resourceType=TOPIC, name=topic-1, patternType=LITERAL)`: 
 	(principal=User:consumer, host=*, operation=DESCRIBE, permissionType=ALLOW)
	(principal=User:consumer, host=*, operation=READ, permissionType=ALLOW) 

Current ACLs for resource `ResourcePattern(resourceType=TOPIC, name=topic-1, patternType=LITERAL)`: 
 	(principal=User:producer, host=*, operation=DESCRIBE, permissionType=ALLOW)
	(principal=User:producer, host=*, operation=WRITE, permissionType=ALLOW)
	(principal=User:consumer, host=*, operation=DESCRIBE, permissionType=ALLOW)
	(principal=User:consumer, host=*, operation=READ, permissionType=ALLOW)
```
7. Добавим доступ ```consumer``` к группе **test** (локальный терминал):
```bash
docker exec kafka-0 kafka-acls --bootstrap-server kafka-0:9092 \
  --command-config /etc/kafka/secrets/adminclient-configs.conf \
  --add --allow-principal "User:consumer" --operation Read -operation Describe --group test
```
Ожидаемый результат:
```    
Adding ACLs for resource `ResourcePattern(resourceType=GROUP, name=test, patternType=LITERAL)`: 
 	(principal=User:consumer, host=*, operation=DESCRIBE, permissionType=ALLOW)
	(principal=User:consumer, host=*, operation=READ, permissionType=ALLOW) 

Current ACLs for resource `ResourcePattern(resourceType=GROUP, name=test, patternType=LITERAL)`: 
 	(principal=User:consumer, host=*, operation=DESCRIBE, permissionType=ALLOW)
	(principal=User:consumer, host=*, operation=READ, permissionType=ALLOW)
```
8. Отправим сообщения в ```topic-1``` используя юзера **producer** (локальный терминал):
```bash
docker exec -it kafka-0 kafka-console-producer --bootstrap-server kafka-0:9092 --topic topic-1 --producer.config /etc/kafka/secrets/producer-configs.conf
```
Пример сообщений
```
>Test 1
>Test 2
>Super
```
9. Отправим сообщения в ```topic-2``` используя юзера **producer** (локальный терминал):
```bash
docker exec -it kafka-0 kafka-console-producer --bootstrap-server kafka-0:9092 --topic topic-2 --producer.config /etc/kafka/secrets/producer-configs.conf
```
Пример сообщений
```
>Test
>Wow
```
10. Прочитаем сообщения из ```topic-1``` используя юзера **consumer** (локальный терминал):
```bash
docker exec -it kafka-0 kafka-console-consumer --bootstrap-server kafka-0:9092 --topic topic-1 --from-beginning --consumer.config /etc/kafka/secrets/consumer-configs.conf --consumer-property group.id=test
```
Ожидаемый результат:
```
Test 1
Test 2
Super
Processed a total of 3 messages
```
11. Прочитаем сообщения из ```topic-2``` используя юзера **consumer** (локальный терминал):
```bash
docker exec -it kafka-0 kafka-console-consumer --bootstrap-server kafka-0:9092 --topic topic-2 --from-beginning --consumer.config /etc/kafka/secrets/consumer-configs.conf --consumer-property group.id=test
```
Ожидаемый результат:
```
[2026-04-13 23:02:18,256] WARN [Consumer clientId=console-consumer, groupId=test] Error while fetching metadata with correlation id 2 : {topic-2=TOPIC_AUTHORIZATION_FAILED} (org.apache.kafka.clients.NetworkClient)
[2026-04-13 23:02:18,257] ERROR [Consumer clientId=console-consumer, groupId=test] Topic authorization failed for topics [topic-2] (org.apache.kafka.clients.Metadata)
[2026-04-13 23:02:18,257] ERROR Error processing message, terminating consumer process:  (kafka.tools.ConsoleConsumer$)
org.apache.kafka.common.errors.TopicAuthorizationException: Not authorized to access topics: [topic-2]
Processed a total of 0 messages
```
