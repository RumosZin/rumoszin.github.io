---
title: "[메시지 큐] RabbitMQ, ActiveMQ"
author:
date: 2024-07-16 22:30:00 +0900
categories: [Backend, Async & Streaming]
tags: [CS, 운영체제, 메시지 전달, 메시지 큐, 푸앙이사진관]
---

## **메시지 지향 미들웨어(MOM)**

> 응용 소프트웨어 간의 **비동기적** 데이터 통신을 위한 소프트웨어
> 메시지를 전달하는 과정에서 **보관/라우팅/변환** 할 수 있다는 장점을 가진다.

- 보관 : 메시지의 백업 기능을 유지함으로써 지속성 제공 → 송수신 측은 동시에 네트워크 연결을 유지할 필요가 없다.
- 라우팅 : 미들웨어 계층이 직접 메시지 라우팅 수행 → 하나의 메시지를 여러 수신자에게 배포 가능하다.
- 변환 : 송수신 측의 요구에 따라 전달하는 메세지를 변환할 수 있다.

**⇒ 메시지 큐는 메세지 지향 미들웨어에 속한다!**

<br>

## **메시지 큐**

> 메시지 큐(Message Queue)란 큐(Queue)라는 자료구조를 채택해서 메시지를 전달하는 시스템이고, 메시지 지향 미들웨어(MOM)을 구현한 시스템이다.

- Producer : 메시지를 발행하고 전달하는 부분
- Consumer : 메시지를 받아서 소비하는 부분
- 메시지 큐 : Producer, Consumer의 메시지 전달 역할을 하는 매개체

<br>

**⇒ 메시지 큐는 MSA(Microservice Architecture)에서 아키텍처의 핵심 역할을 한다!**

**서버가 죽어버린다면 클라이언트의 요청은 증발한다. 하지만 중간에 메시지 큐를 둔다면 서버가 죽은 경우 요청이 메시지 큐에 존재하기 때문에 서버의 alive 여부에 관계 없이 요청들을 가지고 있을 수 있다.**

**✅ 우리 푸앙이 사진관 프로젝트에서도 메시지 큐 사용은 필수적이다! 스프링에서 AI 프로필 생성 요청을 받아서 이미지 처리를 하는 파이썬 서버로 (이미지 N개를 포함한) 요청을 전달해야 하는데, 프로필을 생성하는 AI 모델이 한 번에 하나의 요청만 처리할 수 있는 상황이기 때문에 요청을 대기시키는 작업은 필수적이다.**

<br>

- 메시지 큐에서 데이터를 운반하는 방식에 따라 메시지 브로커와 이벤트 브로커로 나눌 수 있다.
  - 메시지 큐 : 메시지 or 이벤트가 송신되고 수신되는 통신 통로
  - 브로커 : 메시지 큐에 메시지 or 이벤트를 넣어주고 중개하는 역할을 하는 주체

### **메시지 브로커 (Message Broker)**

- Producer가 생산한 메시지를 메시지 큐에 저장하고, 저장된 메시지를 Consumer가 가져갈 수 있도록 한다.
- 메시지 브로커는 Consumer가 메시지 큐에서 데이터를 가져가면 짧은 시간 내에 메시지 큐에서 삭제된다.
- 예) RabbitMQ, ActiveMQ, AWS SQS, Redis

### **이벤트 브로커 (Event Broker)**

- 이벤트 브로커는 기본적으로 메시지 브로커의 역할을 할 수 있다. 그러나 반대로 메시지 브로커는 이벤트 브로커의 기능을 하지 못한다.
- 이벤트 브로커가 관리하는 데이터를 이벤트라고 하고, 메시지 브로커는 Consumer가 메시지 큐에서 데이터를 가져가면 짧은 시간 내에 메시지 큐에서 삭제되는데, 이벤트 브로커 방식에서는 Consumer가 소비한 데이터가 필요한 경우 다시 소비할 수 있다!
- 메시지 브로커보다 대용량 데이터를 처리할 수 있는 능력이 있다.
- 예) Kafka

### **메시지 큐의 장점**

- 비동기 / 낮은 결합도 / 탄력성 / 과잉 / 신뢰성 / 확장성 이렇게 여섯 가지가 검색하면 나오는데, 메시지 큐의 특징을 살려서 주요 장점 세 가지를 자세히 살펴보자.

**비동기 (Asynchronous)**

- 메시지 큐가 없다면, Producer 애플리케이션은 자신의 메시지를 전달 받는 Consumer 애플리케이션에 직접 메시지를 보내야 한다 → End-to-End 통신을 통해 메시지 전달이 이루어진다! → Consumer가 받기 전에 다른 메시지 전달을 할 수 없다.
- 앞선 방식을 동기(Synchronous)라고 하며, 전송 속도가 빠르고 결과를 바로 알 수 있지만 대용량 트래픽이 발생하는 서버에서 매우 비효율적임!

**낮은 결합도 (Decoupling)**

- 푸앙이 사진관의 경우 파이썬과의 통신이 필요한데, 이때 메시지 큐를 사용하지 않는다면 스프링 백엔드에서 보내는 Request와 파이썬 AI에서 보내는 Response의 결합이 일대일로 강해질 것이다.
- 푸앙이 사진관 서비스를 구성하는 백엔드, AI 사이에 메시지 큐를 두면서 둘 사이의 결합을 확 줄일 수 있다!

**탄력성 (Resilience)**

- 탄력성 : 시스템이 장애에 대응하고 유연하게 대처할 수 있는 능력
- 앞선 낮은 결합도를 통해 실현될 수 있음!
- 예) 은행 송수금
  - 메시지 큐가 없다면, 수금 쪽에 문제가 생겼을 때 문제가 송금 쪽에도 전파되어 복구되는 동안 송/수금 둘 다 원활하지 못한 채 Blocking
  - 송금이 수금 시스템의 장애와 상관없이 자신의 송금을 메시지 큐에 전달만 하면 됨! 메시지 큐는 수금 시스템의 장애가 해결될 때까지 큐 내부에 송금 메시지를 보관함!

**✅ 메시지 큐의 비동기적인 특성 덕분에 Producer 프로세스는 Consumer 프로세스가 다운되어 있어도 메시지를 정상적으로 발행할 수 있고, Consumer는 구독한 메시지를 발행하는 Producer 프로세스가 다운되어 있어도 정상적으로 수신할 수 있다!**

<br>

## **AMQP : Advanced Message Queue Protocol**

> 메시지 지향 미들웨어(MOM)을 위한 개방형 표준 응용 계층 프로토콜

![Untitled](/assets/img/240716-1.png){: width="100%"}

- Image from [Here](https://aws.amazon.com/ko/message-queue/)

- AMQP는 일반적인 메시지 큐와 비슷하지만, Exchange라는 라우터가 존재하고 Binding 개념이 존재함!
  - Producer : 요청을 보내는 주체, 보내고자 하는 메시지를 Exchange에 Publish
  - Consumer : Producer로부터 메시지를 받아 처리하는 주체
  - Exchange : Producer로부터 전달받은 메시지를 어떤 메시지 큐로 전송할 지 경ㄹ정하는 장소 (라우팅 기능)
  - Queue : Consumer가 소비하기 전까지 메시지가 보관되는 장소
  - Binding : Exchange와 Queue와의 관계, 즉 특정 Exchange가 특정 Queue에 메시지를 보내도록 정의

<br>

## **RabbitMQ**

> RabbitMQ는 AMQP 프로토콜을 구현해 놓은 오픈소스 메시지 브로커

- Broker 중심적인 형태
- 관리 UI 존재, 거의 모든 언어와 운영체제 지원
- 데이터 처리보단, 관리적 측면이나 다양한 기능 구현을 위한 서비스를 구축할 때 사용

### **처리 구조**

1. Producer가 Broker로 메시지를 보낸다.
2. Broker 내 Exchange에서 해당하는 key에 맞게 Queue에 분배한다. (Binding)
3. 해당 Queue를 구독하는 Consumer가 메시지를 소비한다.

### **단점**

- 메시지 큐 서버 종료 후 재가동 시 큐 내용을 모두 삭제하므로 데이터 손실 위험이 있다.
- Producer, Consumer의 결합도가 높다.

<br>

## **JMS**

> Java MOM 표준 API로서, 메시지 큐 및 Publish-Subscribe 패턴과 같은 메시지 기반 통신을 추상화하고 표준화한 API

![Untitled](/assets/img/240716-2.png){: width="100%"}

- Image from [Here](https://www.novell.com/documentation/extend5/Docs/help/WSSDK/tutorial/jms.html)

- Java 에서 채택한 대표적인 MOM 시스템, 일반적인 메세지 큐 구조와 비슷하다.

<br>

## **ActiveMQ**

> JMS 스펙을 좀 더 편리하게 구현한 오픈소스 메시지 브로커, RabbitMQ와 더불어 현재 대표적인 메시지 브로커로 사용되고 있다.

- Message Broker : 목적지에 안전하게 메시지를 건네주는 중개자 역할
- Destination : 목적지에 배달될 2가지 메시지 모델인 Queue, Topic
- Queue : 메시지가 전달되는 통로 (경합 있음)
- Topic : Queue와 비슷한 역할, 여러 Consumer에게 메시지를 건네줄 수 있음 (경합 없음)

### **처리 모델**

- Queue 모델 : 메시지를 받는 Consumer가 다수일 때 연결된 순서로 메시지 제공
- Topic 모델 : 메시지를 받는 Consumer가 다수일 때 메시지는 모두에게 제공

![Untitled](/assets/img/240716-3.png){: width="100%"}

- Image from [Here](https://gwonbookcase.tistory.com/49?pidx=3)

- ActiveMQ 는 일반적인 메세지큐의 특징 및 장점을 가지고 있으며 AMQP 기반의 RabbitMQ 을 사용하는 프로세스와의 통신은 불가능하다.

**✅ AMQP는 프로토콜, JMS는 API이다. JMS는 메시지 형식이 아닌 브로커와 통신하는 방법을 정의하고, 자바 애플리케이션에만 국한되어 있다. AMQP는 브로커와 통신하는 방법에 대해서 논하지 않지만, 메시지가 유선을 통해 큐에 어떻게 넣고 꺼내지는지에 대해 정의한다.**

**✅ 서로 다른 두 가지 애플리케이션이 소통할 때, 둘 다 자바면 JMS를 통해서 통신할 수 있지만 하나가 파이썬이라면 JMS는 사용할 수 없다. 푸앙이 사진관에서 ActiveMQ가 후보에서 배제된 이유이다!**

### **참고 자료**

- https://support.smartbear.com/readyapi/docs/testing/amqp.html
- https://aws.amazon.com/ko/message-queue/
- https://velog.io/@choidongkuen/%EC%84%9C%EB%B2%84-%EB%A9%94%EC%84%B8%EC%A7%80-%ED%81%90Message-Queue-%EC%9D%84-%EC%95%8C%EC%95%84%EB%B3%B4%EC%9E%90
- https://gwonbookcase.tistory.com/49
- https://m.blog.naver.com/seek316/222117711303

  <br>
  <br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
