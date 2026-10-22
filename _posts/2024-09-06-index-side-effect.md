---
title: "조회 성능을 위해 WHERE / ORDER BY 조건을 다중 칼럼 인덱스로 설정해도 될까?"
author:
date: 2024-09-06 16:30:00 +0900
categories: [Backend, Database]
image: /assets/img/240906-12.png
tags: [푸앙이사진관, 인덱스]
---

<br>

푸앙이 사진관 서비스에서 사용하는 쿼리 메서드를 살펴보던 중, 신경쓰이는 녀석을 발견했다. 바로 PhotoRequestRepository의 `Optional<PhotoRequest> findTopByUserIdOrderByCreateDateDesc(Long userId);`이다. 프로필 이미지 생성 요청을 하기 전에 사용자의 요청이 이미 생성되었고, 이미지가 생성 중이라면 새 요청을 받으면 안된다. 따라서 이 메서드로 새 요청을 받기 전에 사용자의 가장 최근 요청의 처리 상태를 확인해야한다.

```sql
select * from photo_request where user_id=? order by create_date desc limit 1;
```

<br>

**아래 그림은 user와 photo_request의 일대다 관계를 보여주고, 쿼리에 사용되는 필드만 표현한 것이다. 위 쿼리의 결과를 얻기 위해 현재 인덱스의 도움을 어느 정도 받고 있는지, 개선할 수 있는 여지는 없는지, side effect는 없는지 알아보자. (MySQL 8.0 기준으로 실험했다!)**

![Untitled](/assets/img/240906-2.png){: width="80%"}

<br>

## **[1] user_id 단일 칼럼 인덱스**

현재 photo_request 테이블은 user 테이블의 user_id를 외래키로 가지고 있다. `show index from photo_request;`로 photo_request 테이블의 인덱스를 확인할 수 있는데, 주요 정보만 정리하면 다음과 같다.

|             | request_id | user_id     |
| ----------- | ---------- | ----------- |
| Key name    | PRIMARY    | idx_user_id |
| Index type  | B-tree     | B-tree      |
| None Unique | 0          | 1           |
| Collation   | ASC        | ASC         |
| Cardinality | 99617      | 1000        |

- 인덱스는 `B-Tree`로 정렬을 유지하며, 인덱스 칼럼에 대해 정렬(Collation)을 Asc/Desc 중 하나로 정할 수 있다. 아무 조건이 없으면 default로 Asc로 설정된다.

  - 오름차순 인덱스 : 작은 인덱스 키가 B-Tree 왼쪽에 위치, 내림차순은 오른쪽에 위치

- `None Unique`는 고유 인덱스인지 아닌지를 나타내는데, photo_request 테이블의 primary key인 request_id는 unique하고, user : photo_request는 일대다 관계이므로 user_id 외래키에 대해서는 none unique 하다.

- `Cardinality`는 단어가 생소할 수 있는데 인덱스 키 중에 유니크한 값의 수를 의미한다. request_id는 photo_request 테이블의 primary key 이므로 모든 값들이 서로 달라서, cardinality도 행의 개수만큼 나와야 한다.

  - `select count(*) from photo_request;`를 실행했을 때의 행의 개수와 cardinality가 일치하지 않을 수도 있는데, 이는 MySQL이 인덱스의 통계 정보 업데이트를 지연시켜서 그런 것이다. `ANALYZE TABLE photo_request;`로 수동으로 업데이트 시키면 개수가 동일하게 나온다. (실제로는 10만 개의 데이터를 넣었다.)

- **user_id가 1000인 것에 대해서는, 1000명의 user가 100번씩 요청한 것으로 가정하고 테스트 했기 때문이다. 실제로 프로필 이미지 생성을 백 번씩이나 요청할까 싶었지만, 이왕이면 극단적인 상황을 만들고 테스트하고 싶어 이렇게 설정했다.**

<br>

### **1.1 조회 시 실행 시간**

이제 아래 쿼리에 대한 실행 시간, 실행 계획을 확인해본다.

```sql
select * from photo_request where user_id=1000 order by create_date desc limit 1;
```

![Untitled](/assets/img/240906-7.png){: width="100%"}

5번 조회 시도 했을 때 네 번은 0.001초, 한 번은 0.002초가 걸렸다. (평균 조회 시간 0.0012)

### **1.2 조회 시 실행 계획**

![Untitled](/assets/img/240906-8.png){: width="100%"}

실행 계획으로 알 수 있는 사항 중 중요한 것만 정리해보자.

- `select_type : SIMPLE` SELECT 문의 유형이다. SIMPLE은 union, 내부 쿼리가 없는 단순한 SELECT 구문을 말한다.
- `type : ref` primary key가 아닌 인덱스에 대한 비교를 말한다. where 절에 user_id 외래키를 사용했고, 이 인덱스를 이용해 조회한다.
- `key: fk_user_id` possible_keys는 사용할 수 있는 인덱스를 말하고 keys는 실제로 사용된 인덱스를 말한다. user_id의 인덱스 이름이 바로 이것이다.
- `key_len: 9` 인덱스로 사용된 user_id가 Long 타입(8바이트)이라서 8로 나올 줄 알았는데, 9가 나왔다. 찾아보니 Null을 허용하는 경우 1바이트를 추가한다고 한다.
- **`rows : 100` 모든 사용자가 100번 요청하는 상황을 가정 했으므로 where 조건으로 검색한 결과가 100개이다.**
- **`extra : Using filesort` 쿼리의 결과를 정렬할 때, MySQL이 추가적인 작업을 한다는 것을 의미한다. 즉, ORDER BY create_date DESC를 처리하기 위해 정렬이 발생한다.**

### **1.3 Using filesort가 왜 조회 쿼리 성능에 영향을 줄까?**

**✅ 실행 계획에 있던 Using filesort는 단일 테이블에 대해 실행한 쿼리가 정렬 시 인덱스를 사용하지 못하는 경우 발생하는 것이다.**

**✅ 조회 쿼리에서 `WHERE user_id=1000` / `ORDER BY create_date desc` 두 개의 조건이 있는데, 이때 WHERE 절의 user_id만 인덱스로 설정되어 있고, create_date는 인덱스가 아니다. 즉, 인덱스를 이용한 desc 정렬을 할 수 없으므로, MySQL이 100개의 행에 대해 따로 정렬을 해야 한다는 것을 의미한다.**

filesort가 필요한 상황이 생기면, MySQL에서는 내부적으로 테이블을 Sort Buffer에 옮겨 정렬하는 작업을 거친다. 정렬을 하고 결합이 필요한 데이터가 있다면 합치는 작업을 거친 후 데이터를 보내는 작업을 하는데, 단일 테이블이므로 그대로 보낸다.

소량의 데이터를 Sort Buffer에 옮겨서 정렬하는 작업은 크게 문제되지 않는다. 그러나 문제는 데이터가 Sort Buffer의 크기를 넘어설 때 발생한다. 정렬해야 할 데이터가 많아진다면 MySQL은 레코드를 여러 조각으로 나누어서 처리하는데, 이때 임시 저장을 위해 디스크를 사용한다.

Sort Buffer에서 정렬을 수행하고, 결과를 임시로 디스크에 기록하고, 다음 레코드를 가져와서 정렬하고... 를 반복하면서 디스크에 임시 저장을 한다. 그리고 이후에는 정렬된 조각들을 가져와서 병합하면서 정렬을 해야 한다. (Multi-merge)

반복적인 디스크의 쓰기와 읽기는 성능에 매우 안좋은 결과를 가져온다.

**⚠️ 이 글은 조회 성능을 위해 여러 인덱스를 도입하고 최적의 결과를 위해 선택하는 과정을 담고 있으니, "정렬해야 할 데이터가 많아지면 디스크 I/O가 많아져 성능에 좋지 않다"까지만 알아보고, Sort Buffer와 관련된 자세한 내용은 다른 글에서 다루도록 하겠다.**

### **1.4 삽입 시 실행 시간**

인덱스를 설정한다는 것은, B-Tree 알고리즘의 정렬을 이용해 조회 시 성능을 얻으면서 trade off로 삽입/수정/삭제 시 성능을 약간 잃는다는 것과 같다. 대표적으로 삽입 시 실행 시간에 대해서도 체크하면서 넘어가자!

5번 삽입 시도 했을 때 네 번은 0.003초, 한 번은 0.004초가 걸렸다. (평균 삽입 시간 0.0032)

![Untitled](/assets/img/240906-9.png){: width="100%"}

<br>

## **[2] (user_id, create_date) 다중 칼럼 인덱스**

다중 칼럼 인덱스를 도입하게 된 계기는 간단하다. "앞서 WHERE의 절의 user_id만 인덱스가 걸려 있었으니까, ORDER BY의 create_date도 인덱스를 설정하면 되지 않을까?" 라고 생각했고 `ALTER TABLE photo_request
ADD INDEX idx_user_id_create_date (user_id, create_date);`로 다중 칼럼 인덱스를 만들었다.

|              | request_id | user_id                 | create_date             |
| ------------ | ---------- | ----------------------- | ----------------------- |
| Key name     | PRIMARY    | idx_user_id_create_date | idx_user_id_create_date |
| Index type   | B-tree     | B-tree                  | B-tree                  |
| None Unique  | 0          | 1                       | 1                       |
| Seq_in_index | 1          | 1                       | 2                       |
| Collation    | ASC        | ASC                     | ASC                     |
| Cardinality  | 99617      | 1000                    | 97947                   |

<br>

### **2.1 조회 시 실행 시간**

같은 조회 쿼리에 대한 실행 시간, 실행 계획을 확인해본다.

![Untitled](/assets/img/240906-7.png){: width="100%"}

**조회 시간이 매우 짧아질 것으로 예상했는데, 생각보다 그렇지 않았다. user_id가 1000인 것을 가져오면 데이터가 100개인데, 이정도 데이터 개수는 MySQL에서 정렬로 처리하는 것과 인덱스를 이용하는 것에 큰 차이가 없는 것 같다.**

실행 계획을 보고 조회 쿼리에서 인덱스를 잘 활용하고 있는지 확인해보자.

### **2.2 조회 시 실행 계획**

![Untitled](/assets/img/240906-11.png){: width="100%"}

`Extra: Backward index scan`이 매우 눈에 띈다... 왜 이것이 발생했는지 알아보자.

앞서 (user_id, create_date) 다중 칼럼 인덱스를 생성할 때, 정렬 방향을 설정하지 않았으므로 두 칼럼 다 기본 ASC 오름차순으로 정렬된다. 아래 사진에서 왼쪽 케이스처럼 작은 인덱스 키가 B-Tree 왼쪽에 위치한다.

![Untitled](/assets/img/240906-12.png){: width="100%"}

**✅ user_id, create_date 둘 다 오름차순 정렬되어 있는데, 조회 쿼리에서 `ORDER BY create_date DESC`로 데이터를 내림차순으로 정렬한 결과를 원한다. create_date 칼럼에 대해 인덱스의 정렬 방향(ASC)과 쿼리에서 원하는 정렬 방향(DESC)이 달라 MySQL이 인덱스를 역방향으로 스캔한 것이다.**

### **2.3 Backward index scan이어도 인덱스를 잘 활용하고 있는 것일까?**

이 대목에서 많은 고민을 하게 되었다.

1. **✅ create_date는 NOW()를 이용해서 생성하는데, 분초까지 같기 쉽지 않다. create_date는 겹칠 확률이 매우 낮고, 자연히 높은 Cardinality를 보여준다. 실제로 primary key인 request_id의 Cardinality인 10만에 매우 근접한 9만 7천이라는 값을 보여준다. Cardinality가 높고 업데이트가 빈번하지 않은 칼럼에 인덱스를 주면 좋다고 알고 있었으므로, 인덱스를 잘 활용하고 있다고 생각했다.**
2. **✅ `ORDER BY create_date ASC`는 오름차순으로 정렬을 하고 하나를 고르는 것인데, create_date의 가장 최신 값은 B-Tree의 우측 끝 리프 노드에 올 것이므로 Backward index scan을 해서 하나만 가져오면 된다고 생각했다.**

- `ORDER BY create_date ASC` : 오름차순 정렬, Backward index scan으로 하나만 값을 가져온다
- `ORDER BY create_date DESC` : 내림차순 정렬, Forward index scan으로 하나만 값을 가져온다

라고 생각하고 인덱스를 잘 활용하는 방식이라고 (개인적으로) 생각했다. 이후 테크 기업에서 이런 경우에 어떻게 결정하는지 확인하고자 검색을 여럿 했는데, 굉장한 글을 발견했다.

[MySQL Ascending index vs Descending index](https://tech.kakao.com/posts/351)

> 실제 InnoDB에서 Backward index scan이 Forward index scan에 비해서 느릴 수밖에 없는 2가지 이유를 가지고 있다.

> 페이지 잠금이 Forward index scan에 적합한 구조 / 페이지 내에서 인덱스 레코드는 단방향으로만 연결된 구조 (Forwarded single linked link)

<br>

## **[3] (user_id ASC, create_date DESC) 다중 칼럼 인덱스**

우리 서비스는 규모가 작아서, create_date로 정렬할 때 오직 DESC로 정렬하는 것만 사용한다. 위 테크 블로그에 나온 내용을 근거 삼아, create_date의 인덱스는 DESC로 생성한다. Collation이 변경되었다.

|              | request_id | user_id                 | create_date             |
| ------------ | ---------- | ----------------------- | ----------------------- |
| Key name     | PRIMARY    | idx_user_id_create_date | idx_user_id_create_date |
| Index type   | B-tree     | B-tree                  | B-tree                  |
| None Unique  | 0          | 1                       | 1                       |
| Seq_in_index | 1          | 1                       | 2                       |
| Collation    | ASC        | ASC                     | DESC                    |
| Cardinality  | 99617      | 1000                    | 97947                   |

### **3.1 조회 시 실행 시간**

2.1의 조회 쿼리 실행 시간에서도 언급했지만, 데이터 100개에 대해서는 Using filesort로 처리하는 것과 다중 칼럼 인덱스로 정렬 효과를 얻는 방식에 성능 차이가 미미하다 (똑같이 측정된다).

(처음에는 당황했지만 위 카카오 테크 블로그의 테스트를 보니 `SELECT * FROM t1 ORDER BY tid ASC LIMIT 12619775,1;` 정렬 해야할 데이터의 크기가 이정도는 되어야 성능의 차이를 확인할 수 있는 정도가 되는 것 같다.)

### **3.2 조회 시 실행 계획**

![Untitled](/assets/img/240906-13.png){: width="100%"}

2.2에는 Backward index scan이 발생했지만 여기서는 NULL이다. 즉, 효율적으로 인덱스를 이용해서 바로 데이터를 가져온다는 것이다! create_date 칼럼이 이미 DESC로 정렬되어 있으므로 바로 가져올 수 있다.

### **3.3 정말 이렇게 인덱스를 설정해도 될까?**

조회 성능에 이점이 있다고 해서, 인덱스를 바로 설정해도 될까? 이 대답은 무조건 No 이다. Real MySQL 8.0의 8.2 인덱스란? 챕터에 이런 설명이 있다.

> 결론적으로 DBMS에서 인덱스는 데이터의 저장(INSERT, UPDATE, DELETE) 성능을 희생하고 그 대신 데이터의 읽기 속도를 높이는 기능이다.

대표적으로 user_id가 1000인 사람이 하나의 요청을 더 만드는 INSERT 상황이라고 가정해보자. 새로운 키 값이 B-Tree에 저장될 때 즉시 인덱스에 저장될 수도 있고 아닐 수도 있다. (즉시 or 지연)

**B-Tree에 저장될 때는 키 값을 이용해 B-Tree 상의 적절한 위치를 검색해야 한다. 저장될 위치가 결정되면 레코드의 키 값과 대상 레코드의 주소 정보를 B-Tree의 리프 노드에 저장한다.**

이때 리프노드가 꽉차면 리프노드가 분리되어야 하는데, 이것은 상위 브랜치 노드(루트도 아니고 리프도 아닌)까지 작업의 범위가 넓어진다. **이 때문에 B-Tree는 상대적으로 새로운 키를 추가하는 작업에 비용이 많이 드는 것으로 알려졌다.**

인덱스 추가로 인해서 INSERT 쿼리가 정확히 어떤 영향을 받을지는, 테이블의 칼럼 수 / 칼럼의 크기 / 인덱스 칼럼의 특성 등을 확인해야 한다고 한다. Real MySQL 8.0에서 이야기 하는 대략적인 계산 방법으로 인덱스 추가로 인한 비용을 계산해보자. 테이블에 레코드 추가 비용 1, 테이블의 인덱스에 키 추가 비용 1.5일 때 (대부분이 메모리와 CPU 처리가 아니라 디스크로부터 인덱스 페이지를 읽고 써야 해서 걸리는 시간)

- **[create_date가 인덱스가 아닌 경우] request_id, user_id, result_id만 존재하므로 1.5 \* 3 + 1 = 5.5**
- **[create_date가 인덱스인 경우] 인덱스가 4개이므로 1.5 \* 4 + 1 = 7**

### **3.4 조회 vs 나머지**

SELECT의 성능을 얻는 대신에 INSERT, UPDATE, DELETE를 어느 정도 포기할 수 있을까? 이 글의 처음부터 여기까지는 혼자 실험하고 확인해본 결과이지만, 여기서부터는 팀원들과 상의가 필요했다.

**결론부터 말하자면 우리 팀은 create_date에 인덱스를 설정하지 않기로 했다.**

**✅ 조회가 자주 발생할까, 나머지가 자주 발생할까? 이것부터 논의를 시작했는데, 이건 의심의 여지 없이 "나머지"였다. 한 사람이 요청을 여러 번 하는 것과 여러 사람이 요청을 한 번씩 하는 것 중에 많이 발생하는 쿼리를 고르자면, 무조건 후자였다. 푸앙이 사진관 서비스는 한 명이 여러 번 AI 프로필을 생성하는 것보다는, "한번쯤" 생성하는 서비스이다. 게다가 아직은 AI 프로필 생성 시 여러 컨셉을 적용할 수 없으므로, 많으면 세 번 정도 요청을 할 것으로 예상된다.**

<br>

## **결론!**

조회 성능을 위해 WHERE / ORDER BY 조건을 다중 칼럼 인덱스로 설정해도 될까? 이 글의 제목이다. 조회 성능을 위해 인덱스를 설정하고, 인덱스를 잘 활용하는지 확인하고 개선해보았다. 그러나 인덱스를 거는 것은 반드시 trade-off가 있다. 여러 방면에서 검토하고 적용해야 한다.

비록 이 쿼리에 대해서는 적용하지 않는 것으로 결론이 났지만, 생각없이 쿼리를 짜고 남겨두는 것과 이유를 알고 남겨두는 것은 큰 차이가 있다. 정확한 이유에 대해서 알게 되어 뿌듯하다.

기능이 확장되어 더 복잡한 쿼리가 발생한다면, 기쁜 마음으로 이 글을 참고해서 개선해보고 싶다.

<br>
<br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
