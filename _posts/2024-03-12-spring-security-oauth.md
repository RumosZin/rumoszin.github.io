---
title: "[Spring] Spring security, OAuth 2.0으로 구글 로그인 구현하기"
author:
date: 2024-03-12 18:00:00 +0900
categories: [Backend, Auth]
image: https://github.com/RumosZin/spring-security-oauth/assets/81238093/c7a3f0d6-5152-461d-831a-91b709439597
tags: [Spring, 개발, OAuth, Spring Security]
---

spring boot 3.2.3 버전 프로젝트에서 spring security와 oauth를 이용해 구글 로그인을 구현해보자. 전체 코드는 [여기](https://github.com/RumosZin/spring-security-oauth)에 있다.

- Spring boot 3.2.3
- Spring security 6.1.x
- MySQL 8.3.0

## **GCP에서 구글 로그인 설정**

### **새 프로젝트**

1. GCP에 들어가서 spring-security-oauth 이름의 프로젝트를 생성한다. 이름은 자유롭게 정한다.
2. 만든 프로젝트에 접속 후, `API 및 서비스 / 사용자 인증 정보`로 이동한다.
3. `사용자 인증 정보 만들기 / OAuth 클라이언트 ID 항목`을 선택한다.
4. client ID를 생성하기 전에 동의 화면을 구성한다.
   - User Type : 외부
   - 앱 이름 : 구글 로그인 화면에서 뜰 앱의 이름
   - 범위 추가 또는 삭제 : email, profile, openid 선택
5. `OAuth 클라이언트 ID 만들기`로 이동해서 애플리케이션 유형을 `웹 애플리케이션`으로 설정한다.
6. 승인된 리디렉션 URL 주소를 등록한다. 인증에 성공한 경우 구글에서 리다이렉트할 URL이다.
   - spring boot 3 버전에서, spring security가 `{domain}/login/oauth2/code/google`로 리다이렉트 URL을 지원하기 때문에, 별도로 URL을 지원하는 컨트롤러를 만들지 않아도 된다.
   - 현재 로컬에서 개발 중이기 때문에 `http://localhost:8080/login/oauth2/code/google`만 등록하는데, 서버에 배포하면 주소를 추가해야 한다.
7. 생성 버튼을 누르면 클라이언트 ID, 클라이언트 보안 비밀번호를 알 수 있다. 노출되지 않게 복사해서 저장해둔다.

<br>

## **Spring boot 프로젝트**

### **build.gradle 설정**

```bash
# spring boot
implementation 'org.springframework.boot:spring-boot-starter-web'
implementation 'org.springframework.boot:spring-boot-starter'
testImplementation 'org.springframework.boot:spring-boot-starter-test'

# spring security + oauth + social login
implementation 'org.springframework.boot:spring-boot-starter-security'
implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'
implementation 'org.springframework.boot:spring-boot-starter-validation'
implementation 'com.auth0:java-jwt:4.2.1'

# annotation
implementation 'org.projectlombok:lombok:1.18.28'
compileOnly 'org.projectlombok:lombok'
annotationProcessor 'org.projectlombok:lombok'
testCompileOnly 'org.projectlombok:lombok'
testAnnotationProcessor 'org.projectlombok:lombok'

# jpa
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'

# template engine
implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'

# query dsl for jakarta
implementation 'com.querydsl:querydsl-jpa:5.0.0:jakarta'
annotationProcessor "com.querydsl:querydsl-apt:${dependencyManagement.importedProperties['querydsl.version']}:jakarta"
annotationProcessor "jakarta.annotation:jakarta.annotation-api"
annotationProcessor "jakarta.persistence:jakarta.persistence-api"

# mysql
implementation 'mysql:mysql-connector-java:8.0.23'
```

### **application-oauth.yml**

application.properties 혹은 application.yml이 있는 위치에 `application-oauth.yml` 파일을 생성한다. application-oauth.yml에 아래 코드를 입력한다.

들여쓰기를 주의하고, 앞서 저장해둔 클라이언트 ID, 클라이언트 보안 비밀번호를 입력한다.

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: { Client ID }
            client-secret: { Client Secret }
            scope:
              - email
              - profile
```

### **MySQL 설정**

구글 로그인을 통해 회원가입/로그인한 사용자의 정보는 `users` 테이블에 저장된다. `(user_id, email, name, picture, role)`를 칼럼으로 가지는데, jpa가 users 테이블이 없다면 생성할 것이므로 직접 테이블을 만들 필요는 없다.

하지만 application.yml에 접속할 mysql 경로를 알려줘야 하므로 `schema`는 만들어져 있어야 한다.

### **application.yml**

개인 정보로 인해 application.yml의 원본은 올리지 않았고, `application-example.yml`을 참고해서 `application.yml`을 작성한다.

mysql 관련한 정보를 직접 입력해야 한다.

application.yml에서, application-oauth.yml을 포함하도록 설정한다.

```yaml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    # 각자 PC에 만들어놓은 Database이름을 써야 합니다.
    url: jdbc:mysql://{ local host ip address }:{ mysql port }/{ schema_name }?useSSL=false&characterEncoding=UTF-8&serverTimezone=UTC&allowPublicKeyRetrieval=true&useSSL=false
    username: { username }
    password: { password }
  jpa:
    database: mysql
    database-platform: org.hibernate.dialect.MySQLDialect
    show-sql: true
    hibernate:
      ddl-auto: update
    properties:
      hibernate:
        format_sql: true
  profiles:
    include: oauth
```

### **.gitignore**

application-oauth.yml에는 클라이언트 ID와 클라이언트 보안 비밀번호가 존재해서, 노출되면 안되기 때문에 `.gitignore`에 추가해야 한다.

실수로 업로드 했다면 cache까지 삭제해 올라간 파일을 확실하게 삭제하도록 한다.

<br>

## **결과 화면**

`localhost:8080`으로 접속 후 `google login`을 누르면 아래 화면을 통해 구글 로그인에 성공한다!

![image](https://github.com/RumosZin/spring-security-oauth/assets/81238093/c7a3f0d6-5152-461d-831a-91b709439597)

### **참고자료**

- [https://docs.spring.io/spring-authorization-server/reference/guides/how-to-social-login.html](https://docs.spring.io/spring-authorization-server/reference/guides/how-to-social-login.html)
- [https://spring.io/guides/tutorials/spring-boot-oauth2](https://spring.io/guides/tutorials/spring-boot-oauth2)
- [https://velog.io/@99mon/Spring-%EC%8A%A4%ED%94%84%EB%A7%81-%EC%8B%9C%ED%81%90%EB%A6%AC%ED%8B%B0-%EA%B5%AC%EA%B8%80-%EB%A1%9C%EA%B7%B8%EC%9D%B8](https://velog.io/@99mon/Spring-%EC%8A%A4%ED%94%84%EB%A7%81-%EC%8B%9C%ED%81%90%EB%A6%AC%ED%8B%B0-%EA%B5%AC%EA%B8%80-%EB%A1%9C%EA%B7%B8%EC%9D%B8)
- [https://gyuwon95.tistory.com/167](https://gyuwon95.tistory.com/167)

<br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
