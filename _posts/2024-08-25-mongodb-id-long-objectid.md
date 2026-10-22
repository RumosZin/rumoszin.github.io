---
title: "MongoDB를 이용해 Id 필드의 타입 고르기 : Auto Increment vs UUID vs ObjectId"
author:
date: 2024-08-25 00:12:00 +0900
categories: [Backend, Database]
image: /assets/img/240826-4.png
tags: [Database, NoSQL, MongoDB, Spring]
---

<br>

## **Auto Increment 대신 UUID를 선택했다**

데이터베이스 스키마를 선택할 때 타입도 생각하지 않을 수 없다. 이전에 SQL 계열 데이터베이스를 사용했을 때 id pk 칼럼에 Auto increment가 자동 설정 되어 있는 경우가 많았다.

하지만 커머스 서비스의 경우, 상품 데이터의 id pk에 auto increment를 사용하면 API 하나만으로 상품 데이터들을 긁어갈 수 있다. 예를 들어서, `https://some-commerce/products/102341`의 경우 상품 데이터는 1(정확히 1은 아니겠지만)부터 102341까지 있음을 유추할 수 있다.

**✅ id pk는 데이터베이스 collection (or RDB라면 table)에서 식별하는 역할을 하기 때문에, 이 값을 통해 다른 어떠한 내용도 유추되어서는 안된다. 따라서 쉽게 고유한 키를 생성해서 사용할 수 있는 UUID를 사용하기로 했다. [마침 Java 17의 경우에 UUID class를 지원한다.](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/UUID.html)**

<br>

## **Spring data MongoDB에서 UUID는 id 필드에서 바로 쓸 수 없었다**

Spring data MongoDB를 이용해 UserRepository 인터페이스를 만들고, User collection id 필드의 타입을 `UUID`로 구현하고 로그인하니, 오류가 발생했다.

`org.springframework.core.convert.ConverterNotFoundException: No converter found capable of converting from type [org.bson.types.ObjectId] to type [java.util.UUID]`

![Untitled](/assets/img/240826-1.png){: width="100%"}

살펴보면 `ObjectId` 타입을 `UUID`로 바꿀 Converter를 찾을 수 없다는 오류이다. 여기서 의문이 생겼다.

- `ObjectId` 타입은 무엇인가?
- `UUID`를 필드로 사용하기 위해서는 Converter를 두어야 하는가?

<br>

## **MongoDB의 id 타입, ObjectId**

Spring data MongoDB에서 타입을 매핑하는 방식에 대해 찾아보았는데, [공식 문서](https://docs.spring.io/spring-data/mongodb/docs/1.2.0.RELEASE/reference/html/mapping-chapter.html)에서 명쾌하게 설명하고 있다.

> 7.1.1 How the '\_id' field is handled in the mapping layer

> If a field named 'id' is declared as a String or BigInteger in the Java class it will be converted to and stored as an ObjectId if possible. ObjectId as a field type is also valid. If you specify a value for 'id' in your application, the conversion to an ObjectId is delected to the MongoDBdriver. If the specified 'id' value cannot be converted to an ObjectId, then the value will be stored as is in the document's \_id field.

**✅ 즉, String 혹은 BigInteger로 타입을 지정하면 MongoDriver가 `ObjectId` 필드로 convert하여 저장하는데, `UUID`로 타입을 지정하면 ObjectId로 변환할 수 없어서 spring converter 단에서 ConverterNotFoundException 오류가 발생했던 것이다!**

<br>

## **ObjectId는 UUID처럼 고유성을 보장할까?**

처음 스키마를 설계할 때 Auto increment가 아닌 UUID를 선택한 것도, 쉽게 고유한 키를 만들 수 있기 때문에 택했었다. 그렇다면 ObjectId는 고유성, 유일성을 보장할까?

아까 마주한 오류를 다시 한 번 보자. `ObjectId`는 `org.bson.types`에서 제공한다.

`org.springframework.core.convert.ConverterNotFoundException: No converter found capable of converting from type [org.bson.types.ObjectId] to type [java.util.UUID]`

[ObjectId는 BSON의 데이터 타입 중 하나이고, BSON은 MongoDB 내부에서 데이터를 저장하는 형식이다.](https://www.mongodb.com/ko-kr/docs/manual/reference/bson-types/)

[MongoDB 공식 문서에서 이야기하는 ObjectId](https://www.mongodb.com/ko-kr/docs/manual/reference/bson-types/#std-label-objectid)에 따르면, ObjectId의 생성은 다음과 같이 이루어진다.

![Untitled](/assets/img/240826-2.png){: width="60%"}

**< Unix timestamp 4bytes >** 초 단위 저장

**< Random value 5bytes >** 프로세스 당 한 번씩 생성되는 임의의 값, 머신과 프로세스마다 고유함

**< count 3bytes >** 임의의 값으로 초기화되는 3바이트 증분 카운터

1초 안에 같은 프로세스 내에서 카운트를 증가시키는 작업을 한다고 해도, 4바이트는 고정되지만 뒤의 8바이트는 다르기 때문에 중복될 확률이 매우 낮다.

**✅ 즉, UUID처럼 고유성을 보장하면서, MongoDB의 내부적인 id 필드인 ObjectId를 사용하면! 별도의 converter 없어도 Spring data MongoDB가 ObjectId로 변환해서 MongoDB의 id 필드로 매핑시킬 것이다!!**

<br>

## **Auto Increment vs UUID vs ObjectId의 승자는?**

**✅ 스키마 설계 단계에서 UUID가 승리했고, Spring - MongoDB 구현 단계에서 ObjectId가 승리했다. ObjectId로 매핑되게 하는 타입 중에서는 String을 선택했다.**

선택 과정에서 >>왜?<<를 고민하고 근거를 찾으며 선택했기 때문에 매우 뿌듯하다. 한 줄의 코드 리뷰가 여기까지 알아보도록 나를 이끌었다!! 코드 리뷰의 중요성, 재미를 더 깨닫는 요즘이다.

![Untitled](/assets/img/240826-3.png){: width="100%"}

![Untitled](/assets/img/240826-4.png){: width="100%"}

<br>
<br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
