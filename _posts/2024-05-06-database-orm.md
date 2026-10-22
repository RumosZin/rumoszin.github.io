---
title: "[Database] ORM 정의, 등장 배경, 장단점"
author:
date: 2024-05-06 20:12:00 +0900
categories: [Backend, Database]
image: /assets/img/240106-1.png
tags: [Database]
---

✅ [🏝️ Fairy-Tale Island 🏝️](https://github.com/GDSC-CAU/FTIsland-BE) 리팩토링!

2월에 개발을 할 때, 어떤 것은 Repository 인터페이스에 선언만 하고 사용할 수 있고, 어떤 것은 그것만으로 불충분해서 쿼리를 작성해야 하는 경우가 있었다. JPA의 장점이라고 하는 인터페이스 선언만으로 사용할 수 있는 것에 대해서 흐릿하게 아는 상태에서 JPA를 사용하다보니, 혹시 다른 더 나은 방법이 있는데 몰라서 사용하지 못하는 게 아닌가 하는 생각이 들었다.

이번에 리팩토링을 하는 김에, **제대로 JPA에 대해서 알아보자!**

## **ORM이란?**

ORM은 Object-Relation Mapping의 준말로, 말그대로 객체와 관계형 데이터베이스를 연결하는 개념이다. 프로그래밍 언어의 객체와, 관계형 데이터베이스의 데이터를 연결하는 도구이다.

![Untitled](/assets/img/240106-1.png){: width="100%"}

## **ORM의 등장 배경**

### **ORM 개념이 없던 시대의 개발**

ORM 개념이 없던 시대에는, 프로그래밍 언어의 객체와 관계형 데이터베이스의 데이터 사이에 간극이 존재했다. 예를 들어, 사용자가 회원가입하는 로직을 구현한다고 하자.

우선 프로그래밍 언어로 아이디, 비밀번호를 입력받는 코드를 짠다. 이 정보들은 추후 로그인 시 이용하기 위해 **영속적(Persistence)으로 관리 되어야 한다. 그래서 반드시 데이터베이스에 저장해야 한다.**

기본적으로 프로그래밍 언어는 **사용자 객체**를 기준으로 개발한다. 하지만 관계형 데이터베이스에 저장될 때는 사용자 객체가 아니라, **데이터**가 저장되어야 한다. **즉, 프로그래밍 언어에서 사용자 Entity의 Attribute가 관계형 데이터베이스에서 사용자 Table의 Column과 대응되기 때문에, Attribute의 값이 적절한 Column의 데이터가 되도록 개발자가 직접 매핑을 해야 했다.**

**[Entity의 Attribute : Table의 Column]**을 매핑하는 가장 기본적인 방법은 **SQL 쿼리문**을 작성하는 것이다.

### **SQL 쿼리문 작성의 단점**

아래 코드와 같이 프로그래밍 언어로 개발을 하고 나면, 관계형 데이터베이스에 명시적으로 SQL 쿼리문을 작성해야 한다.

```java
public class User {
    private String userId;
    private String userPassword;

    ...
}
```

```sql
INSERT INRO user(user_id, user_password) VALUES (userId, userPassword)
```

하지만 이 방식은 비즈니스 로직 개발보다 SQL 쿼리문 작성에 큰 노력을 요구한다. 반복되는 CRUD 코드를 작성해야 하고, userId 속성이 userInputId로 변경되거나 새로운 속성이 추가된다면, 관련된 SQL 쿼리문을 하나하나 수정해야 한다.

### **ORM의 등장**

위와 같은 문제를 해결하고자 등장한 것이 ORM이다.

- 프로그래밍 언어로 객체 지향 개발을 한다.

- 관계형 데이터베이스를 설계한다.

- 객체 지향 개발과 관계형 데이터베이스 사이를 Mapping하는 방식으로 ORM이 동작한다.

### **ORM의 장점**

**비즈니스 로직 개발 집중**

- 객체마다 CRUD를 위한 SQL문을 각각 작성해야 했는데, ORM이 Mapping으로 처리하면서 SQL문 작성에 시간을 쏟지 않을 수 있다. 데이터베이스에 관심을 덜고, 비즈니스 로직 개발에 집중할 수 있다.

**쉬운 유지 보수**

- 이전에는 Entity 객체의 attribute가 수정되거나 추가되면 모든 SQL문을 찾아서 수정해야 했는데, ORM이 Mapping attribute를 수정하거나 새로운 Mapping을 만듦으로써 데이터베이스의 세부적인 내용을 신경쓰지 않아도 된다.

**코드의 가독성**

- 무조건 키워드를 알아야 하는 SQL문과 달리, ORM에서는 기본적으로 제공하는 메서드를 사용하므로 가독성이 높아진다.

### **ORM의 단점**

**ORM만으로 모든 처리 불가능**

로직이 복잡하고 저장할 데이터가 여러 개인 경우, ORM에서 제공하는 메서드만으로 적절히 처리할 수 없다. 이런 경우에는 직접 쿼리를 작성해야 할 수 있다.

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>ㄴ
