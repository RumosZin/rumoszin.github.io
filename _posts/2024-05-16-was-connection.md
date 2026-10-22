---
title: "WAS 분리 과정에서 만난 Connection Error"
author:
date: 2024-05-08 23:30:00 +0900
categories: [Infra & DevOps, Web]
image: /assets/img/240516-1.png
tags: [Server, WAS, 인턴]
---

<br>

인턴으로 참여하는 프로젝트에서, WEB - DB처럼 웹이 바로 데이터베이스에 접근하는 구조인데, WEB - WAS - DB 와 같은 구조로 변경해야 한다. 현재는 WEB 내의 `api/routes`에 구현된 API들을 사용하는데, WAS가 제공하는 API를 사용하도록 해야 한다.

웹 애플리케이션으로 SvelteKit / Svelte를 사용하고 있고, ORM은 Sequelize, 데이터베이스는 MariaDB를 사용하고 있다. 로그인은 lucia를 이용해서 구현했기 때문에 예외적으로 Drizzle ORM을 사용하고 있다. 따로 WAS 서버를 만들지 않았으므로, 회사 내부서버에 웹 이미지를 배포하고 이 서버가 WAS가 되도록 설정해서 테스트를 진행했다.

SvelteKit 예제가 적었으나, 스택오버플로우에서 [Proxy on Svelte-Kit](https://stackoverflow.com/questions/72753092/how-to-proxy-on-svelte-kit-in-dev-mode)를 발견해서 참고했다.

{: file='vite.config.ts'}

```typescript
import { sveltekit } from "@sveltejs/kit/vite";
import { defineConfig } from "vite";

export default defineConfig({
  plugins: [sveltekit()],
  server: {
    proxy: {
      "/api": "http://{내부서버 IP}/5273"
    }
  }
});
```

<br>

이렇게 설정 후 로컬에서 WEB을 간단히 도커로 배포한 후 확인한 결과, 500 에러가 발생했다. WAS 서버의 로그를 확인하니, 아래와 같았다.

```shell
Error: Can't add new command when connection is in closed state
    at PromiseConnection.query (/app/node_modules/mysql2/promise.js:94:22)
    at MySql2PreparedQuery.execute (file:///app/node_modules/drizzle-orm/mysql2/session.js:50:33)
    at MySqlSelectBase.execute (file:///app/node_modules/drizzle-orm/mysql-core/query-builders/select.js:671:27)
    at MySqlSelectBase.then (file:///app/node_modules/drizzle-orm/query-promise.js:21:17) {
  code: undefined,
  errno: undefined,
  sql: undefined,
  sqlState: undefined,
  sqlMessage: undefined
}
```

<br>

Drizzle ORM의 PromiseConnection 에러가 발생해서 drizzle.js의 connection을 찾아보았다. 현재는 CreateConnection으로 연결을 설정하고 있었다!

```typescript
import "dotenv/config";
import { drizzle } from "drizzle-orm/mysql2";
import mysql from "mysql2/promise";
import * as schema from "./schema";

const connection = await mysql.createConnection({
  host: process.env.MARIADB_HOST,
  user: process.env.MARIADB_USERNAME,
  port: Number(process.env.DB_PORT),
  password: process.env.MARIADB_PASSWORD,
  database: process.env.MARIADB_DB_NAME,
  pool: {
    keepAliveInitialDelay: 10000,
    enableKeepAlive: true
  }
});

export const db = drizzle(connection, {
  schema,
  mode: "default",
  logger: false
});
```

이전에 Drizzle ORM과 lucia.ts를 연결할 때 MySQL2 관련 문서에서 connection pool 부분을 본 기억이 있어서, 변경해서 테스트를 시도했다. createConnection과 createPool의 차이를 알기 쉽게 정리한 그림을 첨부한다.

**createConnection**
![Untitled](/assets/img/240516-1.png){: width="80%"}

**createPool**
![Untitled](/assets/img/240516-2.png){: width="80%"}

- 출처 : https://velog.io/@gwon713/Nodejs-MySQL-DB-connection-pool

(회사의 코드를 첨부할 수 없어서, 코드 없이 설명만 작성한 점은 양해 부탁드립니다!)

✅ **로그에서 connection이 closed state일 때는 command를 추가할 수 없다고 했으므로, createConnection의 단일 연결이 끊기고 나서 WAS와 연결이 이루어져서 끊긴 것이라고 생각했다. 반면 createPool 방식은 여러 쿼리를 병렬적으로 실행할 수 있으므로, connection이 하나 closed state여도 다른 connection이 있다면 문제가 발생하지 않을 것이라고 생각했다!**

하지만 변경하고 실행해도 오류는 변화 없이 그대로였다. 여러 connection 중에 유효한 하나만 있으면 되는 상황이 아니라, WAS와의 connection 자체에 문제가 있는 것 같다. 여기까지 테스트를 하고 위와 같은 생각을 하고 있었는데, 사수께서 로그인에서 예외적으로 허용한 Drizzle ORM에서의 커넥션이 문제가 있는 것 같다고 하셨다.

`hook.server.js`에서 lucia를 이용해서 세션을 이용해서 로그인을 구현하고 있다. 이때 `@sveltejs/kit`의 `redirect`를 이용해서 로그인할 수 없는 경우를 처리하는데, 이때 lucia와 연결된 Drizzle ORM이 WAS와 연결되지 않으면서 `redirect`를 정의할 수 없으니 문제가 생긴 게 아닌가 하는 것이다!

다른 `routes/api`의 API들은 WAS에서 제공하도록 설정할 수 있다고 해도, 로그인 `/auth/login` API는 처리 방식이 달라 생긴 문제인 것이다. 일단 createConnection 방식에서 createPool 방식으로 변경한 뒤, 추후 WAS를 분리해야 하는 단계에서 Drizzle ORM connection을 다시 확인하기로 하였다.

**Drizzle ORM connection 관련해서 찾아보다가 알게 된 createPool vs createConnection을 정리해두고, 추후 관련 문제를 만났을 때 낯설지 않도록 해야겠다.**

<br>
<br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
