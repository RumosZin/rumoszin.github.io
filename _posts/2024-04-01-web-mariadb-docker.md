---
title: "[MariaDB/Drizzle] Web - DrizzleORM - MariaDB를 연결시키자"
author:
date: 2024-04-01 20:30:00 +0900
categories: [Backend, Database]
tags: [Database, MariaDB, 인턴]
---

<br>

## **Docker MariaDB를 띄우자**

### **docker-compose.yml**

```yaml
version: "3.8"
networks:
  mariadb:
services:
  mariadb:
    container_name: mariadb
    image: mariadb:10
    ports:
      - 3306:3306
    volumes:
      - ./volumes/conf.d:/etc/mysql/conf.d
      - ./volumes/db/data:/var/lib/mysql
    env_file: .env
    environment:
      TZ: Asia/Seoul
    restart: always
```

`.env` : host, port, database name 등과 관련된 변수들이 정의되어 있다.

`docker compose up -d —build`

### **MariaDB 컨테이너 접속**

- `mysql -u root -p` 명령어 입력 후 `.env`에 존재하는 `MARIADB_ROOT_PASSWORD`로 MariaDB 컨테이너 내부로 들어갈 수 있다. DataGrip, DBeaver 등을 이용해도 좋다.

<br>

## **Web에서 MariaDB를 이용하도록 설정하자**

**✅ Web - DrizzleORM - MariaDB에서 MariaDB는 준비된 상황이다. Web - DrizzleORM 설정으로, Web - DrizzleORM - MariaDB를 매끄럽게 연결시키자!**

현재 Web 프레임워크로 Svelte/SvelteKit를 사용하고 있다. 관련해서 아래의 문서들을 많이 참고했는데, 이 글을 읽고 있는 사람들도 이 문서들을 통해 도움을 받았으면 한다!

- [https://orm.drizzle.team/docs/get-started-mysql](https://orm.drizzle.team/docs/get-started-mysql)
- [https://www.beekeeperstudio.io/blog/how-to-use-mariadb-with-docker](https://www.beekeeperstudio.io/blog/how-to-use-mariadb-with-docker)
- [https://mariadb.com/kb/en/getting-started-with-the-nodejs-connector/](https://mariadb.com/kb/en/getting-started-with-the-nodejs-connector/)
- [https://bug41.tistory.com/entry/Docker-Docker-compose-에서-env-파일-사용](https://bug41.tistory.com/entry/Docker-Docker-compose-%EC%97%90%EC%84%9C-env-%ED%8C%8C%EC%9D%BC-%EC%82%AC%EC%9A%A9)
- [https://www.reddit.com/r/sveltejs/comments/v1d1ms/connecting_my_database_with_sveltekit/](https://www.reddit.com/r/sveltejs/comments/v1d1ms/connecting_my_database_with_sveltekit/)

### **(error) Access denied for user 'mariadb'@'192.168.@@.@@' (using password: YES)**

```shell
Error: Access denied for user 'mariadb'@'192.168.@@.@@' (using password: YES)
    at Object.createConnection (/Users/gim-yunjin/project/node_modules/mysql2/promise.js:253:31)
    at /Users/gim-yunjin/project/src/lib/server/drizzle.js:13:32
    at async instantiateModule (file:///Users/gim-yunjin/%E1%84%92%E1%85%A7%E1%86%AB%E1%84%83%E1%85%A2ITC%E1%84%8B%E1%85%A1%E1%86%AB%E1%84%8C%E1%85%A5%E1%86%AB%E1%84%87%E1%85%A9%E1%84%80%E1%85%A5%E1%86%AB/project/node_modules/vite/dist/node/chunks/dep-G-px366b.js:54758:9) {
  code: 'ER_ACCESS_DENIED_ERROR',
  errno: 1045,
  sqlState: '28000'
}
Error: Access denied for user 'mariadb'@'192.168.@@.@@' (using password: YES)
    at Object.createConnection (/Users/gim-yunjin/project/node_modules/mysql2/promise.js:253:31)
    at /Users/gim-yunjin/project/src/lib/server/drizzle.js:13:32
    at async instantiateModule (file:///Users/gim-yunjin/%E1%84%92%E1%85%A7%E1%86%AB%E1%84%83%E1%85%A2ITC%E1%84%8B%E1%85%A1%E1%86%AB%E1%84%8C%E1%85%A5%E1%86%AB%E1%84%87%E1%85%A9%E1%84%80%E1%85%A5%E1%86%AB/safety-health-system-web/node_modules/vite/dist/node/chunks/dep-G-px366b.js:54758:9) {
  code: 'ER_ACCESS_DENIED_ERROR',
  errno: 1045,
  sqlState: '28000'
}
```

- 사용자 권한을 주지 않으면 발생하는 오류이다.
- 이전에 postgres 패스워드 오류 관련해서 컨테이너 로그를 봤을 때, 패스워드 입력 시도 IP가 192.168.@@.@@이었다. 이것도 그것과 비슷한데, 내 로컬 컴퓨터가 데이터베이스 컴퓨터(MariaDB 컨테이너)에 접근할 수 없으므로 내 로컬 컴퓨터의 IP 주소를 거부한다.
- 권한 설정을 필요한 경우 해주자.

```shell
GRANT ALL PRIVILEGES ON *.* TO 'root'@'localhost' IDENTIFIED BY 'mariadb_password' WITH GRANT OPTION;

FLUSH PRIVILEGES;
```

<br>

### **drizzle.js : MariaDB 연결 설정**

```jsx
import "dotenv/config";
import { drizzle } from "drizzle-orm/mysql2";
import mysql from "mysql2/promise";
import * as schema from "./schema";

const connection = await mysql.createConnection({
  host: process.env.MARIADB_HOST,
  user: process.env.MARIADB_USERNAME,
  port: process.env.DB_PORT,
  password: process.env.MARIADB_PASSWORD,
  database: process.env.MARIADB_DB_NAME
});

export const db = drizzle(connection, { schema, mode: "default" });
```

<br>

## **export const db = drizzle(connection, { schema, mode: "default" });**

[Drizzle에서 MySQL 계열을 데이터베이스로 사용할 때 사용할 수 있는 스키마 모드](https://orm.drizzle.team/docs/rqb#modes)로 세 가지가 있다.

### **PlanetScale**

- MySQL과 호환되는 서버리스 데이터베이스 플랫폼이다. (서버리스 데이터베이스 플랫폼 중에서는 가장 발전됐다고 한다)
- 서버리스 환경에서 drizzle-orm/planetscale-serverless 드라이버를 통해 PlanetScale에 접근할 수 있다. (개발하는 컴퓨터에 MySQL을 설치하고 싶지 않은 경우 사용할 수 있다)

### **MySQL2**

- mysql2의 성능에 집중한 Node.js용 MySQL 클라이언트
- drizzle-orm/mysql2를 통해 이용할 수 있음

> Drizzle relational queries use lateral joins of subqueries under the hood and for now PlanetScale does not support them.

이로 인해 MySQL2를 선택하게 되었다.

<br>

## **(참고) lucia.ts : drizzle adapter 설정**

현재 웹에서 [lucia 인증 라이브러리](https://lucia-auth.com/)를 이용해서 로그인을 처리하고 있다.

> Lucia is an auth library for your server that abstracts away the complexity of handling sessions. It works alongside your database to provide an API that's easy to use, understand, and extend.

데이터베이스를 postgres를 사용하는 supabase에서, MySQL 계열의 MariaDB로 변경하게 되었는데, 이에 따라 수정이 필요했다.

이전 postgres를 데이터베이스로 사용 : `lucia` → `drizzle postgres adapter` → `postgres`

현재 mariaDB를 사용하므로 mysql 데이터베이스를 사용 : `lucia` → `drizzle mysql adapter` → `mysql`

<br>

- [https://lucia-auth.com/database/drizzle](https://lucia-auth.com/database/drizzle)
- [https://lucia-auth.com/basics/users#define-user-id-type](https://lucia-auth.com/basics/users#define-user-id-type)

`const adapter = new DrizzleMySQLAdapter(db, schema.user_sessions, schema.users);`

- `users` 테이블의 Id는 `Number`가 가능하지만, `user_sessions`는 `string` type이어야 함에 주의한다!!

<br>
<br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
