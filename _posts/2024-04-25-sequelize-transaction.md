---
title: "[Sequelize] Unmanaged/Managed Transaction과 주의사항"
author:
date: 2024-04-25 20:00:00 +0900
categories: [Backend, Database]
tags: [ORM, Sequelize, 인턴]
---

<br>

인턴으로 참여하는 프로젝트에서 SvelteKit / Svelte로 웹 애플리케이션 개발을 하고 있다. 데이터베이스는 MariaDB, 스토리지는 Supabase storage, ORM은 Sequelize를 사용해서 풀스택 개발을 한다. 하나의 화면에도 여러 레포지토리에 데이터를 저장하는데, 이 작업들이 시작부터 끝까지 완결성 있게 이루어지지 않으면 큰일이 난다.

**✅ 이를 위해 A부터 Z까지 원하는 작업이 전부 수행된다면 commit, 중간에 B 작업에서 문제가 생긴다면 A 작업까지 없던 것으로 만드는 트랜잭션을 도입하였다. Sequelize로 어떻게 트랜잭션을 이용할 수 있는지 정리하자!**

Sequelize는 지원하는 프레임워크가 많아서 그런지 특정 프레임워크에 대한 설명은 명확하게 자세히 나와있지 않다고 생각했는데, 베이직한 내용에 대해서는 예시 코드와 설명이 잘 되어있다. 블로그를 많이 찾아봐도 [공식 문서](https://sequelize.org/docs/v6/other-topics/transactions/)만한 글을 발견하지 못했으므로 한 번 확인하길 추천한다.

<br>

## **Unmanaged Transaction**

- 사용자가 직접 한 번에 이루어져야 하는 작업 단위들에 transaction을 설정하고, commit과 rollback 위치를 지정한다.
- 트랜잭션으로 처리할 작업의 단위를 try-catch로 묶고, 단위에 transaction을 설정한다.

```javascript
const t = await sequelize.transaction();

try {
  const user = await User.create(
    {
      firstName: "Bart",
      lastName: "Simpson"
    },
    { transaction: t }
  );

  await user.addSibling(
    {
      firstName: "Lisa",
      lastName: "Simpson"
    },
    { transaction: t }
  );

  await t.commit();
} catch (error) {
  await t.rollback();
}
```

<br>

## **Managed Transaction**

- 사용자가 한 번에 이루어져야 하는 작업 단위들에 transaction을 설정하는 것은 맞지만, commit과 rollback 위치를 지정하지 않아도 sequelize에 의해 자동으로 처리된다.

```javascript
try {
  const result = await sequelize.transaction(async (t) => {
    const user = await User.create(
      {
        firstName: "Abraham",
        lastName: "Lincoln"
      },
      { transaction: t }
    );

    await user.setShooter(
      {
        firstName: "John",
        lastName: "Boothe"
      },
      { transaction: t }
    );

    return user;
  });
} catch (error) {
  // If the execution reaches this line, an error occurred.
  // The transaction has already been rolled back automatically by Sequelize!
}
```

<br>

## **주의 사항**

📌 Sequelize의 create 문에서 error가 throw 되어야 try-catch에서 catch로 가기 때문에 정상적으로 동작할 수 있다. Sequelize의 메서드(create, find …)를 사용하는데 repository에서 `throw new Error()`를 하지 않으면 catch에 걸리지 않는다.

📌 **✅ Repository에서 반드시 throw error가 필요하다. BaseRepository를 지정해서 다른 Repository들이 BaseRepository를 extends 한다면 BaseRepository에 custom throw error를 만들어 명시적으로 공통된 에러를 반환하는 것도 좋은 방법이다!** (나는 이 방식을 선택하고, 구현했다. 코드를 첨부하고 싶지만 회사 프로젝트의 코드여서 위에 첨부한 것처럼 공식 문서의 코드만 첨부하게 된 점은 양해 부탁드립니다.)

📌 Sequelize의 메서드를 사용하지 않는 경우 정상적으로 롤백되지 않는다. 예를 들어 Sequelize의 find, create를 사용한다면 정상적으로 동작하지만, 직접 구현한 메서드를 사용하는 경우 롤백되지 않을 수 있다.

<br>

**✅ 인턴십을 하며 Sequelize + MariaDB에서 사용하는 트랜잭션 방법에 대해 알아보라고 하셔서 공식 문서에 나오는 내용을 어떻게 코드에 적용할 수 있을지만 알아봤었다. 트랜잭션이 프로젝트에 필요한 이유, Why에 대해서는 어렴풋이 알게 되었으니, How를 공부해야할 것 같다.**

<br>
<br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
