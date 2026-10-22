---
title: "[Spring] Entity, DTO, DAO (feat. 관심사의 분리)"
author:
date: 2024-04-28 21:28:00 +0900
categories: [Backend, Spring]
image: /assets/img/240428-2.png
tags: [Spring]
---

✅ [🏝️ Fairy-Tale Island 🏝️](https://github.com/GDSC-CAU/FTIsland-BE) 리팩토링!

기획을 2023년 12월 말부터 2024년 1월 중순까지 하고, 2024년 2월에 부랴부랴 개발하느라 디테일한 부분들을 모르고 넘어갔다... 개발하면서도 부족함을 느꼈는데, 인턴십을 하면서 **관심사의 분리**의 중요성을 크게 느껴 내 코드를 두고 볼 수 없었다. 성공적인 리팩토링을 위해서, Entity, DAO, DTO, VO에 대해 알아보자.

## **Entity란?**

- 외부에서 최대한 `entity`의 `getter`를 사용하지 않도록 내부에 로직을 구현해야 한다.
- `domain` 로직만 구현하고, `presentation` 로직은 구현하지 않는다. 일반적으로 Presentation 계층 (Controller), Service 계층 (Service), Data access 계층 (Repository)과, 모든 계층에서 사용되는 도메인 모델 클래스로 구성되어 있다.

![Untitled](/assets/img/240428-1.png){: width="70%"}

### **setter 사용 지양**

- `getter` 사용을 최대한 피해야 하지만, 기본적으로 `getter`가 `entity`에 있어야 한다.
  **Entity 클래스에서 `setter` 만드는 것을 피해야 한다!**
- 쉽게 개발하기 위해 습관적으로 getter, setter 어노테이션을 붙이는데, Entity 클래스에 `setter`가 있다면 어느 계층에서든 사용 가능해서 entity의 인스턴스 값들이 언제 어디서 변하는지 명확히 알 수 없다.

**생성자 이용해서 필드에 값 넣기**

- 생성자에 현재 넣는 값이 어떤 필드인지 명확히 알 수 없고, 매개 변수끼리의 순서가 바뀌더라도 코드가 모두 실행되기 전까지는 문제를 알 수 없어서, 인스턴스의 생성 시점에 생성자를 이용하는 것도 좋은 방법이 아니다.

```java
new User("01zxcv", "1234");
```

**Builder 패턴을 사용하기**

- 필드가 많아져도 어떤 값을 어떤 필드에 넣는지 코드로 확인 가능하고, 필요한 필드에만 값을 넣을 수 있다.
- ✅ [🏝️ Fairy-Tale Island 🏝️](https://github.com/GDSC-CAU/FTIsland-BE)에서 어떤 패턴을 쓰고 있는지 확인해보니, **builder 패턴을 쓰지 않고 Entity의 생성자로 인스턴스를 만들고, settter를 사용한다.**
- ✅ **추후 리팩토링 시 builder 패턴을 사용하도록 고쳐야 한다!!** 대부분 아래와 같이 setter를 사용하는데, controller/service/dto에서 무분별하게 사용하는 점이 큰 문제다.

{: file='/VocaService.java'}

```java
public UserVocaEntity save(Integer userId, Integer vocaId){
    Optional <UserVocaEntity> userVoca = userVocaRepository.findByUserIdAndVocaId(userId, vocaId);

    if(userVoca.isPresent()){
        return userVoca.get();

    } else{
        UserVocaEntity userVocaEntity = new UserVocaEntity(); // Entity의 생성자
        userVocaEntity.setUserId(userId); // Entity의 setter 사용
        userVocaEntity.setVocaId(vocaId);
        userVocaRepository.save(userVocaEntity);
        return userVocaEntity;
    }

}
```

<br>

## **DTO란?**

- data transfer object, **계층 간 데이터 교환을 위한 객체**이다.
- DB에서 꺼낸 데이터를 담고있는 Entity를 가지고 만드는 Wrapper라고 볼 수 있는데, Controller 같은 클라이언트 단과 직접 마주하는 계층에 직접 Entity를 전달하는 대신 DTO를 이용해서 데이터를 교환한다.
- 계층간 데이터 교환이 목적인 객체이므로, 특별한 로직이 없는 순수한 데이터 객체여야 한다.

![Untitled](/assets/img/240428-2.png){: width="100%"}

✅ [🏝️ Fairy-Tale Island 🏝️](https://github.com/GDSC-CAU/FTIsland-BE)를 확인해보니, 아래와 같은 두 가지 케이스가 있었다. 첫째는 위의 `/voca`처럼 `VocaDTO`를 이용해 Request를 전달하고, 아무것도 반환하지 않는 컨트롤러이다. 둘째는 아래의 `/voca/list`처럼 Request, Response 모두 `VocaDTO`를 이용하는 것이다.

✅ **나는 두번째 방식이 좋지 않다고 생각한다.** 요청과 반환 중 어느 것 하나라도 API 명세가 변하면, 요청이 들어올 때와 데이터를 반환할 때 모두 확인 작업을 해야 하기 때문이다. **요청, 반환 각자에게 필요하지 않은 필드가 포함되는 것도, 관심사의 분리 목적에서 벗어났다.** **따라서 리팩토링 시 DTO의 경우 Request, Response를 분리해야 한다.**

{: file='/VocaController.java'}

```java
@DeleteMapping("/voca")
public void deleteUserVoca(@RequestBody VocaDTO vocaDTO){
    vocaService.deleteUserVoca(vocaDTO.getUserId(), vocaDTO.getVocaId());
}

@PostMapping("/voca/list")
public List<VocaDTO> getVocaList(@RequestBody VocaDTO vocaDTO){
    return vocaService.getVocaList(vocaDTO.getUserId()); // 해당 id의 동화 내용 list
}
```

### **toEntity 코드의 위치**

첨부한 그림과 같이 Presentation 계층에서 Service 계층으로 데이터를 전달할 때는 DTO를 사용한다. 그런데 Data Access 계층에서 실제로 데이터를 저장할 때는 DTO가 아니라 Entity를 사용하므로, DTO의 값들을 Entity로 바꾸는 변환 작업이 필요하다.

✅ 나는 이때 고민이 생겼는데, **toEntity 코드를 Entity에 작성해야 할까, DTO에 작성해야 할까? 데이터베이스에 접근할 때 DTO에서 Entity로 변환이 일어나야 하는 것이므로, 작업의 주체는 DTO라고 생각했다. 리팩토링 시 toEntity의 경우 DTO에 작성하자!**

<br>

## **DAO란?**

- data access object, **DB의 데이터에 접근하는 객체**이다.
- JPA에서 DB에 데이터를 CRUD 하는 repository 객체들이 DAO이다.
  - JpaRepository<>를 상속받는 Repository 객체들이 DAO라고 볼 수 있고, 프로젝트의 서비스 모델과 실제 데이터베이스를 연결하는 역할을 한다.
- Repository는 entity 객체를 보관하고 관리하는 저장소고, DAO는 데이터에 접근하도록 DB 접근 관련 로직을 모아둔 객체이다. [(둘 다 개념의 차이일 뿐 실제로 개발할 때는 비슷하게 사용한다고 한다.)](https://www.inflearn.com/questions/111159/domain%EA%B3%BC-repository-%EC%A7%88%EB%AC%B8)

{: file='/UserVocaRepository.java'}

```java
public interface UserVocaRepository extends JpaRepository<UserVocaEntity, Integer> {
    List<UserVocaEntity> findByUserId(Integer userId);
    Optional<UserVocaEntity> findByUserIdAndVocaId(Integer userId, Integer vocaId);

    void deleteByUserIdAndVocaId(Integer userId, Integer vocaId);
}
```

<br>

## **프로젝트 패키지 구조**

✅ 이번 주제를 다루면서, 우리 코드를 다시 한 번 살펴봤는데, 관심사의 분리(Seperation of Concern), 의존성(Dependency)에 대해 고민하면서 코드를 고치면 좋을 것 같다는 생각을 하게 되었다. 패키지 구조도 관심사의 분리 측면에서 개선하면 좋을 것 같다.

✅ 이 과정에서 프로젝트 구조부터 고쳐야 하는지?를 고민했는데, [인프런에서 프로젝트 패키지 구조에 대해 영한님이 답변하신 것](https://www.inflearn.com/questions/16046/%ED%94%84%EB%A1%9C%EC%A0%9D%ED%8A%B8-%ED%8F%B4%EB%8D%94-%EA%B5%AC%EC%A1%B0%EC%99%80-%EA%B0%95%EC%9D%98-%EC%9D%BC%EC%A0%95%EC%97%90-%EA%B4%80%ED%95%98%EC%97%AC-%EC%A7%88%EB%AC%B8%EC%9D%B4-%EC%9E%88%EC%8A%B5%EB%8B%88%EB%8B%A4)을 보니, 우리의 현재 상황을 정확히 인식하고 구조를 설정하는 것이 좋을 듯 하다!

✅ 이번 주제에 대해서 찾아보다가 DDD (Domain Driven Development) 단어가 등장해서 찾아보다가, [프로젝트 패키지 구조에 계층형 패키지 구조와 도메인 패키지 구조](https://velog.io/@ogu1208/Spring-%ED%8C%A8%ED%82%A4%EC%A7%80-%EA%B5%AC%EC%A1%B0)가 있다고 해서 추후 Domain Driven에 대해 알아보고 적용해봐야겠다!

<br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
