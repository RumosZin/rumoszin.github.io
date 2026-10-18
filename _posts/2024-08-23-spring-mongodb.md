---
title: "Spring에서 MongoDB 사용하기"
author:
date: 2024-08-23 00:12:00 +0900
categories: [Backend, Database]
tags: [Database, NoSQL, MongoDB, Spring]
---

<br>

## **MongoDB Docker 세팅**

Springboot 프로젝트에서 MongoDB를 사용하기 위해, Docker로 mongodb, mongo-express 컨테이너를 띄우자! [TodoChallengers-DB 레포지토리](https://github.com/TodoChallengers/TodoChallengers-DB)를 만들어서 동료 백엔드들과 같은 환경을 가질 수 있도록 했다! (두 명 뿐이지만...)

여러 문서들을 참고했지만 [이 테크블로그](https://www.sktenterprise.com/bizInsight/blogDetail/dev/2652)가 가장 잘 설명하고 있다! 참고하기 바란다.

{: file='docker-compose.yml'}

```yaml
version: "3.8"
services:
  mongodb:
    image: mongo
    container_name: mongodb
    restart: always
    ports:
      - 27017:27017
    volumes:
      - ~/mongodb:/data/db
    env_file: .env

  mongo-express:
    image: mongo-express
    restart: always
    ports:
      - 8081:8081
    env_file: .env
```

<br>

## **Springboot 프로젝트 세팅**

### **build.gradle**

{: file='build.gradle'}

```yaml
implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
```

### **application.yml**

mongodb docker를 세팅할 때 `.env`의 `ME_CONFIG_MONGODB_ADMINUSERNAME`, `ME_CONFIG_MONGODB_ADMINPASSWORD`에 설정한 값을 넣는다.

{: file='application.yml'}

```yaml
spring:
  data:
    mongodb:
      uri: mongodb://{ME_CONFIG_MONGODB_ADMINUSERNAME}:{ME_CONFIG_MONGODB_ADMINPASSWORD}@localhost:27017/todochallengers?authSource=admin
```

### **Spring data MongoDB**

Spring data JPA를 그동안 익숙하게 사용해왔는데, 이상하게 Spring data MongoDB는 한번에 연결이 잘 안되고 찾을 수 없다는 오류가 계속 발생했다.

**✅ [Connecting to MongoDB](https://docs.spring.io/spring-data/mongodb/reference/mongodb/configuration.html)에서 연결하는 방법을 찾을 수 있었다.**

> One of the first tasks when using MongoDB and Spring is to create a MongoClient object using the IoC container.

문서에 따르면, Mongo Client가 만들어지고 이것을 이용해서 MongoClientDatabaseFactory를 만들어야 한다!

<br>

**✅ MongoClientDatabaseFactory를 관련해서는 Spring boot 3.0부터 지원되기 시작한 [SimpleMongoClientDatabaseFactory](https://docs.spring.io/spring-data/mongodb/docs/current/api/org/springframework/data/mongodb/core/SimpleMongoClientDatabaseFactory.html)를 참고했다!**

> Factory to create MongoDatabase instances from a MongoClient instance.

이후 구글링을 통해 트랜잭션을 위한 MongoTransactionManager이 필요함을 알게 되어 같이 추가했다.

### **MongoConfig.java**

{: file='/config/MongoConfig.java'}

```java
@Configuration
@EnableMongoAuditing
public class MongoDbConfig {

    @Bean
    @ConfigurationProperties("spring.data.mongodb")
    public MongoProperties properties() {
        return new MongoProperties();
    }

    /**
     *
     * @param properties
     * @return
     */

    @Bean
    public SimpleMongoClientDatabaseFactory mongoClientDatabaseFactory(MongoProperties properties) {
        return new SimpleMongoClientDatabaseFactory(properties.getUri());
    }

    @Bean
    public MongoTransactionManager transactionManager(MongoDatabaseFactory databaseFactory) {
        return new MongoTransactionManager(databaseFactory);
    }
}
```

<br>

### **SimpleMongoClientDatabaseFactory의 연결 설정**

`application.yml`에서 mongodb 컨테이너와 관련된 정보를 입력했었다. `MongoProperties` 빈으로 등록된 정보를 생성자로 주입했다.그리고 여기에서 MongoDB uri를 `SimpleMongoClientDatabaseFactory` 생성자의 인자로 넘긴다.

[Connecting to MongoDB](https://docs.spring.io/spring-data/mongodb/reference/mongodb/configuration.html)에 나와있던 대로, `MongoClient`(여기서는 MongoDB uri)로 `MongoDatabaseFactory`를 만든다.

{: file='SimpleMongoClientDatabaseFactory.java'}

```java
public class SimpleMongoClientDatabaseFactory
    extends MongoDatabaseFactorySupport<MongoClient> implements DisposableBean {
        public SimpleMongoClientDatabaseFactory(String connectionString) {
        this(new ConnectionString(connectionString));
    }
}
```

<br>

MongoDb의 uri가 Springboot가 Connection을 위해 찾는 주소가 된다!

`ConnectionString.java는` MongoDB 클라이언트와 데이터베이스 서버 간의 연결을 설정하기 위해 사용되는 MongoDB URI(Connection String)를 파싱하고 관리하는 역할을 한다. 주어진 MongoDB URI 문자열을 받아서, 이를 통해 연결 설정에 필요한 다양한 정보(호스트, 포트, 데이터베이스, 인증 정보, SSL 설정 등)를 추출하고, 이러한 정보를 내부적으로 관리할 수 있게 한다!

{: file='ConnectionString.java'}

```java
public class ConnectionString {
    private static final String MONGODB_PREFIX = "mongodb://";
    private static final String MONGODB_SRV_PREFIX = "mongodb+srv://";
    private static final Set<String> ALLOWED_OPTIONS_IN_TXT_RECORD = new HashSet(Arrays.asList("authsource", "replicaset", "loadbalanced"));
    private static final Logger LOGGER = Loggers.getLogger("uri");
    private final MongoCredential credential;
    private final boolean isSrvProtocol;
    private final List<String> hosts;
    private final String database;
    private final String collection;
    private final String connectionString;
    ...
}
```

여기까지 작성했다면 Springboot에서 MongoDB를 사용할 준비는 끝났다! 이제 신나게 개발하면서 만나는 문제들과 고민거리들을 잘 정리해보자!!!

<br>
<br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
