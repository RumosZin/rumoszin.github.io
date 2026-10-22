---
title: "두 번의 갱신 분실 문제의 원인을 찾아보자! (feat. 공유 잠금, 배타적 잠금)"
author:
date: 2024-09-03 12:30:00 +0900
categories: [Backend, Concurrency & Lock]
image: /assets/img/240903-12.png
tags: [푸앙이사진관, 락]
---

<br>

푸앙이 사진관은 유저의 사진을 받아서 AI 프로필을 생성하는 서비스이다. 사용자는 AI 프로필 생성을 요청할 때마다, 완료된 프로필 이미지를 받을 이메일을 입력받는다. **하지만 이메일을 잘못 입력하면 프로필 이미지가 엉뚱한 이메일로 보내질 수 있기 때문에, 이메일 체크와 사용자의 이메일 수정은 필수적인 기능이다.**

관련해서 테스트 코드를 짜다가, 이런 생각이 들었다. **✅ uploadPhoto와, updateEmail이 동시에 호출되면, 변경된 이메일로 이미지가 전송되고, 이미지 처리 완료 상태도 올바르게 업데이트 될까? 테스트를 해보자!**

## **동시 호출 테스트 코드를 작성하자**

쓰레드 두 개를 만들어서 두 서비스 코드를 동시에 수행하는 경우를 테스트 한다. uploadPhoto에서 완료 이메일을 보낼 때 변경된 이메일로 보내기를 원한다. 아래와 같이 테스트 코드를 작성했다.

```java
@Test
    void 수정_조회가_동시에_발생하는_경우_테스트() throws InterruptedException {

        Long userId = 1000L; // 테스트할 사용자 ID
        String email = "newUser998@example.com"; // 변경할 이메일

        CountDownLatch latch = new CountDownLatch(2);
        ExecutorService executor = Executors.newFixedThreadPool(2);

        AtomicInteger successCount = new AtomicInteger();
        AtomicInteger failCount = new AtomicInteger();

        // uploadPhoto 호출
        executor.submit(() -> {
            try {
                photoService.uploadPhoto(998L, "http://example.com/image.jpg");
                successCount.incrementAndGet();
            } catch (Exception e) {
                System.out.println(e.getMessage());
                failCount.incrementAndGet();
            } finally {
                latch.countDown();
            }
        });

        // updateEmail 호출
        executor.submit(() -> {
            try {
                photoRequestService.updateEmail(userId, email);
                successCount.incrementAndGet();
            } catch (Exception e) {
                System.out.println(e.getMessage());
                failCount.incrementAndGet();
            } finally {
                latch.countDown();
            }
        });

        latch.await(); // 두 개의 쓰레드가 완료될 때까지 대기
        executor.shutdown();

        System.out.println("success count: " + successCount.get());
        System.out.println("fail count: " + failCount.get());
    }
```

### **동시 호출 실행 결과**

결과를 확인하니, 원하는 것과 정반대의 결과가 나왔다.

**문제 1. 이메일 수정이 완료되었다면서, 이전 이메일로 전송되고 있었다.**

**문제 2. 오직 uploadPhoto의 이미지 처리 완료 상태만 반영되고 updateEmail의 이메일 수정은 반영되지 않았다.**

![Untitled](/assets/img/240903-2.png){: width="100%"}
![Untitled](/assets/img/240903-5.png){: width="100%"}

<br>

## **현재의 코드를 살펴보자**

uploadPhoto는 AI 모델이 프로필 이미지 생성을 완료한 후 S3에 업로드를 마쳤을 때 호출하는 API이다. uploadPhoto, updateEmail 내부가 현재는 어떤 로직으로 되어 있는지 확인해보고, 문제 해결의 실마리를 얻어보자.

### **uploadPhoto**

세 단계의 작업이 `@Transactional`로 묶여있다.

1. 예외처리
2. 결과 이미지 업데이트
3. 이메일 발송

셋 중에 꼭 트랜잭션으로 묶을 필요가 없는 것을 뽑으라면, 단연 이메일 발송 부분일 것이다. **2. 결과 이미지 업데이트는 uploadPhoto 메서드에서 가장 중요한 부분이고, 무조건 트랜잭션에 포함시켜야 한다. 1. 예외처리는 꼭 uploadPhoto 내에서 처리할 필요가 없어보일 수 있지만, photoRequestRepository에 접근해서 2번 작업의 실행 여부를 결정하므로 필요하다.**

**✅ 하지만 3. 이메일 발송은, 프로그램이 실행되는 동안 메일 서버와 통신할 수 없는 상황이 생긴다면 웹 서버뿐만 아니라 DBMS까지 영향을 줄 수 있으므로 분리해야 한다.** (관련해서 [RealMySQL 8.0] 5.1 트랜잭션, 5.1.2 트랜잭션 주의사항을 참고했다.)

**⚠️ 수정 필요 포인트. 이메일 발송을 uploadPhoto 트랜잭션에서 제외시킨다. 현재 글에서는 두 번의 갱실 분실 문제를 다루므로, 트랜잭션에서 제외시키는 내용은 다른 글에서 다루도록 하겠다.**

![Untitled](/assets/img/240903-1.png){: width="100%"}

### **updateEmail**

updateEmail에는 요구사항이 있다. **만약 photo_request의 status가 1이라면, 이미 요청이 처리가 완료되었으므로 (이미지가 생성되고 이메일도 전송되었으므로) 이메일을 수정할 수 없다. 하지만 처리가 완료되지 않은 상태라면 이메일을 수정할 수 있다.**

![Untitled](/assets/img/240903-3.png){: width="100%"}

<br>

## **동시 호출 시 트랜잭션 Thread의 동작을 확인하자**

동시 호출 실행 결과 이미지를 다시 보자. 왼쪽에 적힌 thread-1, thread-2을 보면, uploadPhoto가 thread-1, updateEmail이 thread-2에서 실행되고 있음을 알 수 있다.
![Untitled](/assets/img/240903-2.png){: width="80%"}

### **Hibernate JPA 쿼리 실행을 확인하자**

application.yml에서 spring.jpa.show-sql=true로 설정하면 쿼리가 실행될 때마다 로그가 출력된다. 어떤 쿼리가, 어떤 코드에 의해, 어떤 순서로 처리되는지 확인한 결과는 다음과 같다. (클릭하면 크게 보입니다.)

번호가 실행 순서를 의미하는 것은 아니며, 단순히 설명의 편의를 위해 붙인 것이다!

![Untitled](/assets/img/240903-6.png){: width="100%"}

### **[1] 같은 row에 대해 다른 트랜잭션이 동시에 공유 잠금을 가질 수 있다**

[MySQL의 공유 잠금에 대해서 공식 문서](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)는 다음과 같이 정의하고 있다. **✅ InnoDB가 제공하는 공유 잠금은 잠금을 유지하는 트랜잭션이 행을 읽을 수 있도록 허용한다. 한 트랜잭션에 공유 행 잠금이 있는 경우, 다른 트랜잭션의 행 잠금 요청도 승인된다. 즉, 두 트랜잭션 모두 같은 행에 대해 공유 잠금을 유지한다.**

> InnoDB implements standard row-level locking where there are two types of locks, shared (S) locks and exclusive (X) locks. A shared (S) lock permits the transaction that holds the lock to read a row.

> If transaction T1 holds a shared (S) lock on row r, then requests from some distinct transaction T2 for a lock on row r are handled as follows: A request by T2 for an S lock can be granted immediately. As a result, both T1 and T2 hold an S lock on r.

따라서, 그림에서 1번과 같이 uploadPhoto와 updateEmail 모두 같은 행에 대해 공유 잠금을 유지한다!

### **[2] Repository.findByXXXId의 결과도 공유 잠금을 가진다**

[1]과 마찬가지로, findByXXXId는 select 쿼리를 발생시키기 때문에 쿼리 결과로 나온 행에 대해 공유 잠금을 가진다.

### **[3] User-PhotoRequest의 Lazy Loading으로 인해 select 쿼리가 실행된다**

User와 PhotoRequest가 일대다 매핑되어있는데, 이때 `@ManyToOne(fetch = FetchType.LAZY)`로 인해 실제로 getUser()로 User를 가져올 때 조회 쿼리가 발생한다. 하나씩 쿼리 발생을 확인하다보니 자세히 찾아보게 된 점이 많은데, 그 중 하나가 JPA의 즉시 로딩과 지연 로딩이다.

**⚠️ 정리 필요 포인트. JPA의 즉시 로딩과 지연 로딩에 대해서 알아본다. 이메일 발송 로직 분리처럼 다른 글에서 자세히 다루도록 하겠다.**

### **[4] photo_request의 행을 수정하는 경우 배타적 잠금을 가진다**

[MySQL의 배타적 잠금에 대해서 공식 문서](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)는 다음과 같이 정의하고 있다. **✅ 한 트랜잭션이 행에 배타적 잠금을 가지는 경우, 다른 트랜잭션은 행에 대해 공유 잠금이나 배타적 잠금을 걸 수 없다. 배타적 잠금이 해제될 때까지 기다려야 한다!**

> An exclusive (X) lock permits the transaction that holds the lock to update or delete a row.

> If a transaction T1 holds an exclusive (X) lock on row r, a request from some distinct transaction T2 for a lock of either type on r cannot be granted immediately. Instead, transaction T2 has to wait for transaction T1 to release its lock on row r.

[7] photo_result의 행을 수정하는 경우도 같은 내용이다.

### **[5] 배타적 잠금이 걸린 행에 접근하는 경우 잠금이 풀릴 때까지 대기한다**

Hibernate로는 쿼리만 확인할 수 있어서, 5번 요청이 4번의 처리가 끝나고 트랜잭션이 commit 될 때까지 대기하는지 스프링의 로그로는 확인하기 어렵다. 서로 다른 두 MySQL 세션을 열어 update 쿼리를 발생시키고, commit 하지 않은 채로 대기한 후 락 정보를 확인해보자!

먼저 한 세션을 열어서 updateEmail에서 실행되는 이메일 수정 쿼리를 입력한다. 이메일 수정 쿼리의 트랜잭션이 종료되지 않은 상태에서, 다른 세션을 열어서 uploadPhoto에서 실행되는 status 수정 쿼리를 입력한다.

이메일 수정 트랜잭션이 커밋되지 않은 상태이므로, status 수정 쿼리는 대기하고 있음을 확인할 수 있다. 시간이 흐르면, 다음과 같이 `Lock wait timeout exceeded;`를 확인할 수 있다.

![Untitled](/assets/img/240903-5.png){: width="100%"}

- 이메일 수정 쿼리

![Untitled](/assets/img/240903-8.png){: width="100%"}

- 상태 업데이트 쿼리

![Untitled](/assets/img/240903-9.png){: width="100%"}

상태 업데이트 쿼리가 행에 대한 배타적 잠금을 얻기 위해 대기하는지 확인해본다. performance_schema 데이터베이스의 data_locks를 출력하면 행의 잠금 정보에 대해 확인할 수 있다. `select * from data_locks\G;` (쓰레드 정보를 알고 싶다면 `SELECT * FROM performance_schema.threads;`를 통해 확인할 수 있다.)

**✅ 2.row, 4.row를 보면, request_id 998 데이터에 대해서 2.row는 WAITING 상태이고, 4.row는 GRANTED 상태이고 X(배타적) 잠금을 가지고 있음을 확인할 수 있다.**

**⚠️ 정리 필요 포인트. 데이터베이스 상에서 잠금을 확인하는 과정에서 공유 잠금, 배타적 잠금 외에 1.row처럼 테이블에 대해 잠금을 설정하는 경우, IX 잠금, REC_NOT_GAP 잠금에 대해서 알게 되었다. 여러 잠금에 대해서는 다른 글에서 자세히 알아본다!**

![Untitled](/assets/img/240903-11.png){: width="100%"}

### **[6] 현재 코드에서는 데이터 수정 시 버전을 확인하지 않는다**

[Thread 2] : updateEmail이 Transaction commit되고 request_id 998 행에 가지고 있던 배타적 잠금을 해제한다. 이와 동시에, WAITING 상태였던 status 수정 요청이 배타적 잠금을 가지고, 행을 수정한다.

**✅ 분명 [4]의 이메일 수정을 수행했는데, status를 수정하고 나서 이메일을 수정한 내용이 온데간데 없어지는 두 번의 갱신 분실 문제가 발생한다! 두 트랜잭션 모두 [1]에서 photo_request의 request_id가 998인 데이터에 대해 조회를 한다. 하지만, 두 트랜잭션 모두 이 데이터를 수정할 때 다른 트랜잭션에 대해 변경된 이력이 있는지 확인하지 않는다.**

![Untitled](/assets/img/240903-12.png){: width="100%"}

<br>

## **두 번의 갱신 분실 문제가 발생하는 이유를 찾았다**

지금까지 uploadPhoto와 updateEmail 코드가 동시에 실행될 때 두 번의 갱신 분실 문제가 발생하는 이유를, 메서드 내부의 JPA repository 호출 순서와 그에 따른 MySQL InnoDB의 행 잠금 확인을 통해 알게 되었다.

두 트랜잭션 모두 이 데이터를 수정할 때 다른 트랜잭션에 대해 변경된 이력이 있는지 확인하지 않기 때문에 발생한 문제였다! 원인에 대해 명확히 알고 나니 해결할 일만 남았다. 해결 방식들에 대해서는 다음 글에서 이어서 작성하도록 하겠다!

## **참고 자료**

- [MySQL InnoDB Locking](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)
- Real MySQL 8.0 5.1.2 트랜잭션 주의사항, 5.3 InnoDB 스토리지 엔진 잠금

  <br>
  <br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
