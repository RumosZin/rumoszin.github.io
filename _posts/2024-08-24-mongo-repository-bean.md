---
title: "MongoRepository에 @Repository가 없어도 빈 등록이 되는 이유"
author:
date: 2024-08-24 00:12:00 +0900
categories: [Backend, Database]
image: /assets/img/240824-1.png
tags: [Database, NoSQL, MongoDB, Spring]
---

<br>

인턴십을 하면서 코드 리뷰를 하며 남의 코드를 유심히 들여다 보는 것도 큰 성장임을 알게 되었다. 사이드 프로젝트를 할 때도 남의 코드를 자세히 보고 **이유있는 수정**을 요구하려고 노력 중이다! 그 첫번째 경험을 하게 되어 공유한다.

## **`@Autowired`의 지속적인 등장**

아래의 코드가 `UserChallengeService`, `PublicChallengeService`, `ChecklistService`, `ReactionService`에 계속해서 등장하고 있었는데, @Autowired를 반복할 필요가 없다고 생각해서, 삭제를 요청했다.

```java
    @Autowired
    private ChallengeRepository challengeRepository;
```

이번 사이드 프로젝트에서는 뭐든지 "이유있는" 행동을 하고싶었다. 삭제해달라고 요청했을 때 왜? 라는 질문을 받는다면 과연 나는 잘 대답할 수 있을까? 생각해보았는데 대답은 No 였다. **✅ 이제부터 @Autowired를 repository 선언마다 사용할 필요가 없는 이유에 대해 알아보자!**

<br>

## **MongoRepository의 빈 등록**

{: file='ChallengeRepository.java'}

```java
public interface ChallengeRepository extends MongoRepository<Challenge, UUID>
```

이렇게 `ChallengeRepository`는 `MongoRepository`를 상속받는다.

`MongoRepository`는 `@NoRepositoryBean`이고 `ListCrudRepository<T, ID>`, `ListPagingAndSortingRepository<T, ID>`, `QueryByExampleExecutor<T>`를 받는다.

그리고 이들 각각도, 전부 `@NoRepositoryBean`으로 등록되어 있다.

<br>

## **@NoRepositoryBean**

레포지토리 코드들을 아무리 타고타고 들어가서 확인해봐도, 정확히 `@Bean`으로 등록하는 부분이 없어서 이상하게 생각하고 있었다! `@NoRepositoryBean`이 굉장히 수상해서 찾아보게 되었다.

[Annotation Interface NoRepositoryBean](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/NoRepositoryBean.html)

> In this case you typically derive your concrete repository interfaces from the intermediate one but don't want to create a Spring bean for the intermediate interface.

**✅ 즉, 중간에 위치한 기본 인터페이스는 빈으로 생성되지 않고, 이를 상속하는 하위 리포지토리들만 빈으로 등록될 수 있도록 하는 어노테이션이었다!** 실제로 애플리케이션 실행 시점에 스프링이 생성하는 빈들을 확인해보면, `ChallengeRepository`와 관련해서 아래 두 개를 생성하는 것을 확인할 수 있었다.

- **`challengeRepository` : `@Autowired`에 의해 생성되는 빈, 여러 번 코드에 등장하지만 실제로는 하나의 인스턴스만 생성하고 이후에 등장할 때 빈 목록에서 `challengeRepository`를 찾아 재사용한다**

- **`mongodb.ChallengeRepository.fragments#0` : `@NoRepositoryBean`에 의해서 `MongoRepository`를 상속하는 가장 하위 리포지토리인 `ChallengeRepository`를 빈으로 등록한 것**

**✅ 이러한 이유와 함께 [코드 리뷰](https://github.com/TodoChallengers/TodoChallengers-BE/pull/15#discussion_r1729794846)를 하니, 리뷰 받는 분도 납득하고 코드를 수정할 수 있었고, 나도 `@NoRepositoryBean`의 역할에 대해 정확히 알아갈 수 있었다.**

![Untitled](/assets/img/240824-1.png){: width="100%"}

<br>

## **애플리케이션 생성 시점에 등록된 빈 목록 확인하는 코드**

```java
public static void main(String[] args) {

    ConfigurableApplicationContext context = SpringApplication.run(BeApplication.class, args);
    String[] beanNames = context.getBeanDefinitionNames();
    Arrays.sort(beanNames);
    for (String beanName : beanNames) {
        System.out.println(beanName);
    }
}
```

<br>
<br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
