---
title: "두 번의 갱신 분실 문제를 해결하자! (feat. 비관적 락, 낙관적 락)"
author:
date: 2024-09-03 20:00:00 +0900
categories: [Backend, Concurrency & Lock]
tags: [푸앙이사진관, 락]
---

<br>

[앞선 글](https://rumoszin.github.io/posts/second-lost-update-data-mysql/)에서 푸앙이 사진관의 uploadPhoto, updateEmail 트랜잭션이 동시에 일어날 때 두 번의 갱신 분실 문제가 발생함을 확인했다. 그리고 메서드 내부의 repository 호출 코드를 살펴보면서, 쿼리 실행과 MySQL 락을 확인하며 문제의 원인에 대해 파악했다.

이번 글에서는, 두 번의 갱신 분실 문제를 어떻게 해결하려고 시도했는지, 결론적으로 어떤 방법을 채택했는지 소개하고자 한다.

![Untitled](/assets/img/240903-6.png){: width="100%"}

<br>

## **방법 1. 마지막 커밋을 인정하고 최초 커밋 재시도하기**

설명의 편의를 위해 문제가 발생했던 상황을 정리한 그림을 위에 첨부했다. 마지막 커밋을 인정하는 방법은 Thread 1의 커밋을 인정하는 것으로, 이메일을 업데이트한 내용은 없어지고 status 업데이트만 행에 반영된다.

status가 업데이트 될 때 이메일 업데이트가 사라진다면, 사용자는 이메일을 수정한 줄 알고 기다리고 있는데 우리 서비스는 옛날 이메일로 이미지를 보내는 불상사가 벌어진다. 이는 애플리케이션의 신뢰성 측면에서 좋지 않다.

**✅ 현재 코드에서는 마지막 커밋을 인정하는 것까지는 된다. 하지만 최초 커밋(Thread 2)을 재시도할 방법은 없다. 데이터베이스 단에서 전혀 문제가 없다고 판단하기 때문에, 최초 커밋이 없어지는 상황을 애플리케이션이 감지할 방법이 없다.**

테스트 코드에서 AtomicInteger로 트랜잭션 성공/실패 시 카운트를 관리하고 있는데, 두 번의 갱신 분실 문제가 발생해도 실패 카운트는 전혀 발생하지 않는다. status는 0에서 1로 바뀌었지만, email은 new로 바뀌지 않았는데도 말이다!

![Untitled](/assets/img/240904-1.png){: width="100%"}

![Untitled](/assets/img/240904-2.png){: width="100%"}

<br>

## **방법 2. 무조건 순차적으로 실행하기 : 비관적 락**

photo_request의 request_id가 998인 데이터에 대해 두 트랜잭션이 모두 수정하기 때문에 발생하는 문제이다. 무조건 두 트랜잭션에 충돌이 발생할 것으로 가정하고, 비관적 락을 이용해보자. 비관적 락은 데이터베이스의 락을 이용해서 동시성을 제어하는 것이다. 조회 시 일반적인 조회가 아닌 `SELECT FROM ... FOR UPDATE`를 사용한다.

조회 메서드에 `@Lock(LockModeType.PESSIMISTIC_WRITE)`를 걸어서 비관적 락을 걸 수 있다.

![Untitled](/assets/img/240904-12.png){: width="100%"}

쿼리의 진행 순서를 확인해보면, 조회 시 일반적인 select가 아니라 `select ... from update`를 사용한다. 이로 인해 조회 시 공유 잠금이 아닌 배타적 잠금을 가지므로, 다른 트랜잭션은 대기한다. 대기하다가 배타적 잠금이 해제되면 그제서야 조회부터 작업을 시작한다.

![Untitled](/assets/img/240904-13.png){: width="100%"}

![Untitled](/assets/img/240904-14.png){: width="100%"}

<br>

## **방법 3. 최초 커밋을 인정하고 이후 트랜잭션 재시도하기 : 낙관적 락**

uploadPhoto, updateEmail 트랜잭션이 같은 행을 수정하는 상황이 발생하므로, 먼저 수정하는 트랜잭션만 커밋하고 나중에 수정하는 트랜잭션은 롤백 시킨 후 재시도 해보자. 여러 트랜잭션 간 충돌이 일어나지 않을 것이라 가정하는 낙관적 락을 통해 문제를 해결해보자.

나중에 수정하는 트랜잭션을 롤백시키기 위해서 어떻게 해야할까? [Spring의 Optimistic Locking 문서](https://docs.spring.io/spring-data/relational/reference/jdbc/entity-persistence.html#jdbc.entity-persistence.optimistic-locking)에서는 `@Version`을 통해 낙관적 락을 애플리케이션 단에서 적용하는 방법에 대해 소개하고 있다.

> Spring Data supports optimistic locking by means of a numeric attribute that is annotated with @Version on the aggregate root. Whenever Spring Data saves an aggregate with such a version attribute two things happen:

> The update statement for the aggregate root will contain a where clause checking that the version stored in the database is actually unchanged.

**✅ 스프링이 제공하는 낙관적 락을 사용하면 업데이트 문에는 데이터베이스에 저장된 버전이 실제로 변경되지 않았는지 확인하는 위치 절이 포함된다! 앞선 글에서 문제 원인을 [두 트랜잭션 모두 이 데이터를 수정할 때 다른 트랜잭션에 대해 변경된 이력이 있는지 확인하지 않는 것]으로 정의했다. `@Version`을 이용하면 변경 이력을 확인하므로 문제를 해결할 수 있을 것이다! 차근차근 구현해보자.**

<br>

### **3-1. 먼저 첫번째 수정만 인정하도록 하자**

동시에 수정할 것으로 예상되는 엔티티에 Version을 추가한다. 버전 관리용 필드를 만드는 것이다.

```java
@Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Getter
public class PhotoRequest {
    ...
    @Version
    private Long version;
    ...
}
```

LockModeType에는 여러 가지가 있는데, [jakarta.persistence.LockModeType](https://jakarta.ee/specifications/persistence/2.2/apidocs/javax/persistence/lockmodetype#OPTIMISTIC)에서 각 타입에 대한 설명을 볼 수 있다. 낙관적 락을 사용할 것이므로, `OPTIMISTIC`으로 타입을 지정했다.

```java
@Lock(LockModeType.OPTIMISTIC)
```

<br>

테스트 코드를 실행하고 Hibernate로 쿼리를 조회하면, 다음과 같이 version 정보를 포함해서 조회한다. **✅ 업데이트 시 where 절에 version을 포함시켜서 중간에 다른 트랜잭션에 의해 버전이 바뀌었는지 확인하고, 바뀌지 않았다면 (조회 시의 버전과 일치한다면) 버전을 업데이트 하면서 데이터도 업데이트 한다.**

![Untitled](/assets/img/240904-6.png){: width="100%"}

결과를 확인하면, `Row was updated or deleted by another transaction` 오류를 내며 나중에 수정하는 트랜잭션은 실패한다! 실제로 데이터를 확인해보면, 먼저 수정하는 트랜잭션의 수정인 이메일 수정만 반영되고, 두번째 수정인 status는 반영되지 않았다.
![Untitled](/assets/img/240904-7.png){: width="100%"}
![Untitled](/assets/img/240904-8.png){: width="100%"}

(서로 다른 두 메서드가 같은 행을 수정하는 경우이므로, 트랜잭션 시작은 동시에 일어나지만 내부적으로는 순서를 가진다. 따라서 시퀀스로 표시하였다.)

![Untitled](/assets/img/240904-9.png){: width="100%"}

<br>

### **3-2. 두번째 수정은 재시도 하자 : @Retryable**

첫번째 수정만 인정하고 커밋, 두번째 수정은 롤백까지 성공했다. 하지만 우리가 원하는 것은, 첫번째 수정 / 두번째 수정 모두 성공하는 것이다. 두번째 수정에 대해서 재시도 하는 로직을 짜보자.

**✅ 재시도 하기 위해서 충돌이 날 것으로 예상되는 메서드에 일일히 재시도 로직을 포함시키는 것은 적절하지 않다. version 충돌로 처리에 실패했을 때 재시도 하는 것은, 핵심 비즈니스 로직이 아니라 낙관적 락을 도입하는 메서드들에 공통적으로 적용되어야 하는 로직이다.**

[Spring에서 제공하는 retry](https://github.com/spring-projects/spring-retry)를 이용해서 재시도 로직을 추가하자. 먼저 의존성을 추가하고 SpringBoot entryPoint에 `@EnableRetry`를 작성한다.

{: file='build.gradle'}

```shell
implementation 'org.springframework.retry:spring-retry' // spring retry
implementation 'org.springframework:spring-aspects' // 선언적 방식의 retry
```

```java
@EnableRetry
public class PuangbeApplication {}
```

이제 재시도 해야하는 낙관적 락 메서드에 `@Retryable`을 붙여 재시도 조건을 정해보자. [org.springframework.retry.annotation](https://docs.spring.io/spring-retry/docs/api/current/org/springframework/retry/annotation/Retryable.html#include--) 문서에서 Retryable에서 사용할 수 있는 여러 조건들을 확인하고, 상황에 맞게 선택해서 사용하면 된다.

**maxAttempts, backoff**

최대 재시도 횟수의 기본값이 3으로 되어있는데, 1000으로 지정하고 재시도 간격도 0.1s로 지정한다.

**retryFor**

Retryable의 [include](https://docs.spring.io/spring-retry/docs/api/current/org/springframework/retry/annotation/Retryable.html#include--)는 기본적으로 `{}`이기 때문에, 재시도를 해야할 예외가 무엇인지 알려줘야 한다. 낙관적 락의 버전 충돌 시 다음과 같은 예외가 발생한다.

JPA [javax.persistence.OptimisticLockException](https://docs.oracle.com/javaee%2F7%2Fapi%2F%2F/javax/persistence/OptimisticLockException.html)

> Thrown by the persistence provider when an optimistic locking conflict occurs. This exception may be thrown as part of an API call, a flush or at commit time. The current transaction, if one is active, will be marked for rollback.

Spring [org.springframework.orm.ObjectOptimisticLockingFailureException](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/orm/ObjectOptimisticLockingFailureException.html)

> Exception thrown on an optimistic locking violation for a mapped object.

낙관적 락의 버전 충돌 시 JPA, Hibernate에서 오류가 발생한다. Spring은 이를 ObjectOptimisticLockingFailureException으로 감싸서 예외를 발생시키므로, ObjectOptimisticLockingFailureException가 발생했을 때 재시도하도록 지정한다.

**✅ 최종적인 Retryable 조건은 다음과 같다! 이것을 uploadPhoto, updateEmail 메서드에 어노테이션으로 달아주면 재시도 로직 적용이 완료된 것이다.**

```java
    @Retryable(
            retryFor = {ObjectOptimisticLockingFailureException.class},
            maxAttempts = 1000,
            backoff = @Backoff(100)
    )
```

<br>

### **3-3. 결과를 확인하자**

테스트 코드를 실행시키고 결과를 확인하자. 모든 수정사항(status 0->1, email old->new)들이 정확히 반영되었다. Hibernate SQL 쿼리로 확인한 결과 재시도 로직도 잘 적용됨을 알 수 있다! 두 번 업데이트를 하기 때문에 버전도 0에서 2로 변경되었다.
![Untitled](/assets/img/240904-11.png){: width="100%"}

![Untitled](/assets/img/240904-10.png){: width="100%"}

<br>

### **비즈니스 로직 상의 오류는 없을까?**

현재 테스트 결과 이메일 업데이트가 먼저 커밋되고 uploadPhoto는 재시도 한다. uploadPhoto는 재시도 할 때 변경된 이메일을 이메일 전송 로직에 전달할 것이므로, 문제가 없다!!

<br>

**✅ 그런데, uploadPhoto가 먼저 커밋되면 무슨 일이 일어날까? uploadPhoto의 status 업데이트로 인해 "status가 0인 가장 최근 요청에 대해서만 이메일을 변경할 수 있다"는 updateEmail의 요구사항을 만족하지 못해서 "이미 이미지 전송이 완료된 요청입니다."라는 예외와 함께 재시도를 종료할 것이다. 즉, 어떤 트랜잭션이 먼저 커밋되느냐에 상관 없이, 비즈니스 로직이 흐트러지는 일은 없다!**

## **어떤 방법을 선택할까? 비관적 락 vs 낙관적 락**

비관적 락을 사용하는 방식은 확실하게 충돌을 방지하고, 데이터베이스 단에서 처리하기 때문에 데이터 일관성을 보장한다. 하지만, 충돌 발생 확률이 얼마나 될 지 모르겠으나 충돌을 미연에 방지하기 위해 조회 시에도 배타적 잠금을 거는 것은 성능을 크게 저하시킨다.

선착순 티켓팅처럼 1000명의 예매가 동시에 몰려드는 상황이라면 비관적 잠금을 사용할 것이다. 그러나, 지금은 여러 메서드에서 "만에 하나" 같은 레코드를 수정하고자 하는 경우에 대처하는 방식을 골라야 하므로, 낙관적 락을 사용하는 방식을 택하겠다. 충돌 확률이 굉장히 적기 때문에, 충돌 시 재시도 로직을 구현해야 하더라도 성능을 포기하고 싶지는 않다.

<br>
<br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
