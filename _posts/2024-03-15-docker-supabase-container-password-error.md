---
title: "[Supabase/Docker] Supabase 컨테이너 password 에러 : password 변경하기"
author:
date: 2024-03-11 13:30:00 +0900
categories: [Backend, Database]
tags: [Database, Supabase, Docker, 인턴]
---

## **상황**

- Supabase를 개발 서버에 Self-Hosting 하여 사용하는 상황
- 개발 서버에서 직접 Supabase 이미지를 pull 받을 수 없는 상황이어서, 로컬에서 개발 서버에 맞는 이미지를 받아서 압축한 후 scp로 보내야 하는 상황
- 로컬에서는 문제 없었던 이미지가 개발 서버에서는 password 에러를 내며 재시작하는 상황

[Supabase 컨테이너 password 에러 : `.env` 확인하기](https://rumoszin.github.io/posts/docker-supabase-container-error-solution/) 방법으로도 해결이 안되고 Supabase 컨테이너가 재시작하는 경우가 있다.

아래와 같이 `docker ps` 명령어로 확인한 결과 status가 `Restarting (1) 17 seconds`인 경우가 있다.

```shell
[yunjin@YUNJIN-CentOS db]$ docker ps
CONTAINER ID   IMAGE                              COMMAND                    CREATED          STATUS                          PORTS                                                                                                      NAMES
e8b8b7dc3947   supabase/storage-api:v0.46.4       "docker-entrypoint.s…"    9 minutes ago    Restarting (1) 17 seconds ago                                                                                                              supabase-storage
b0c5e236b589   supabase/postgres-meta:v0.79.0     "docker-entrypoint.s…"    9 minutes ago    Up 9 minutes (healthy)          8080/tcp                                                                                                   supabase-meta
6f4fcf86c945   supabase/gotrue:v2.143.0           "auth"                     9 minutes ago    Restarting (1) 31 seconds ago                                                                                                              supabase-auth
70100fc3732c   postgrest/postgrest:v12.0.1        "postgrest"                9 minutes ago    Restarting (1) 31 seconds ago                                                                                                              supabase-rest
7b57a638f471   supabase/postgres:15.1.0.147       "docker-entrypoint.s…"    9 minutes ago    Up 9 minutes (healthy)          0.0.0.0:5432->5432/tcp, :::5432->5432/tcp                                                                  supabase-db
4a8b906004c0   supabase/studio:20240301-0942bfe   "docker-entrypoint.s…"    9 minutes ago    Up 9 minutes (unhealthy)        3000/tcp                                                                                                   supabase-studio
7f0f0815bfb1   kong:2.8.1                         "bash -c 'eval \"echo…"   14 minutes ago   Up 14 minutes (healthy)         0.0.0.0:8000->8000/tcp, :::8000->8000/tcp, 8001/tcp, 0.0.0.0:8443->8443/tcp, :::8443->8443/tcp, 8444/tcp   supabase-kong
b205666dc51f   darthsim/imgproxy:v3.8.0           "imgproxy"                 14 minutes ago   Up 14 minutes (healthy)         8080/tcp                                                                                                   supabase-imgproxy
```

재시작하는 컨테이너의 로그를 확인해보면 알겠지만, 컨테이너 실행에 뭔가 문제가 생겨 재시작하는 경우가 대다수이다. 또는, status는 정상적으로 보이지만 컨테이너 로그를 확인하면 오류가 난 경우가 있다.

- healthy 컨테이너의 docker logs (ex : supabase studio)

```shell
  ▲ Next.js 13.5.6
  - Local:        http://4a8b906004c0:3000
  - Network:      http://172.22.0.3:3000

 ✓ Ready in 284ms
```

- error 컨테이너의 docker logs (ex : storage-api)

**password authentication failed for user "supabase_storage_admin"**

```shell
/app/node_modules/pg-protocol/dist/parser.js:287
        const message = name === 'notice' ? new messages_1.NoticeMessage(length, messageValue) : new messages_1.DatabaseError(messageValue, length, name);
                                                                                                 ^

error: password authentication failed for user "supabase_storage_admin"
    at Parser.parseErrorMessage (/app/node_modules/pg-protocol/dist/parser.js:287:98)
    at Parser.handlePacket (/app/node_modules/pg-protocol/dist/parser.js:126:29)
    at Parser.parse (/app/node_modules/pg-protocol/dist/parser.js:39:38)
    at Socket.<anonymous> (/app/node_modules/pg-protocol/dist/index.js:11:42)
    at Socket.emit (node:events:517:28)
    at addChunk (node:internal/streams/readable:368:12)
    at readableAddChunk (node:internal/streams/readable:341:9)
    at Readable.push (node:internal/streams/readable:278:10)
    at TCP.onStreamRead (node:internal/stream_base_commons:190:23) {
  length: 118,
  severity: 'FATAL',
  code: '28P01',
  detail: undefined,
  hint: undefined,
  position: undefined,
  internalPosition: undefined,
  internalQuery: undefined,
  where: undefined,
  schema: undefined,
  table: undefined,
  column: undefined,
  dataType: undefined,
  constraint: undefined,
  file: 'auth.c',
  line: '326',
  routine: 'auth_failed'
}

Node.js v18.19.0
```

예시로는 `storage-api`만 가지고 왔으나, 사실 온갖 password 에러가 날 수 있다. 로컬에서는 문제 없었던 `docker-compose.yml`과 `.env`를 개발 서버(Rocky Linux)에서 실행하면, 특정 컨테이너마다 password 에러가 발생할 수 있다. 전체 로그를 가져오기에는 무리가 있어서, 핵심 로그만 가져왔다.

- **`storage-api` : password authentication failed for user "supabase_storage_admin"**
- **`gotrue` : failed to connect to host=db user=supabase_auth_admin database=postgres**
- **`postgrest/postgrest` : password authentication failed for user "authenticator"**
- **`supabase/postgres` : FATAL: password authentication failed for user "supabase_auth_admin"**

<br>

## **해결 방법 : password 변경하기**

Supabase를 클라우드에 띄워놓고 사용하는 방식이 아니라, 개발 서버를 두고 Self-Hosting 하는 방식이라 해결 방법을 찾기 어려웠다. **이럴 때는 단순히 구글링을 하기 보다는, 공식 문서 레포지토리의 Self-Hosted 라벨을 찾는 것이 좋다.** 실제로 Closed된 Self-Hosted 라벨까지 거의 확인했다! 도움이 된 [Issue의 Comment](https://github.com/supabase/supabase/issues/18836#issuecomment-1804051169)를 첨부한다.

[Step 1] `postgres`의 컨테이너 아이디를 알아낸다.

```shell
[yunjin@localhost supabase]$ docker ps
CONTAINER ID   IMAGE                              COMMAND                    CREATED         STATUS                          PORTS                                                                                                      NAMES
**6a96e18b188b**   supabase/postgres:15.1.0.147       "docker-entrypoint.s…"    4 minutes ago   Up About a minute (healthy)     0.0.0.0:5432->5432/tcp, :::5432->5432/tcp                                                                  db
17d5546bd758   supabase/studio:20240326-5e5586d   "docker-entrypoint.s…"    4 minutes ago   Up About a minute (unhealthy)   3000/tcp                                                                                                   studio
8bd6d8cfd0d6   supabase/storage-api:v0.46.4       "docker-entrypoint.s…"    4 minutes ago   Up About a minute (unhealthy)   5000/tcp                                                                                                   storage
73f003d31775   kong:2.8.1                         "bash -c 'eval \"echo…"   4 minutes ago   Up About a minute (healthy)     0.0.0.0:8000->8000/tcp, :::8000->8000/tcp, 8001/tcp, 0.0.0.0:8443->8443/tcp, :::8443->8443/tcp, 8444/tcp   kong
274de8adaa2c   mariadb:10                         "docker-entrypoint.s…"    4 days ago      Up About an hour                0.0.0.0:3306->3306/tcp, :::3306->3306/tcp                                                                  mariadb
```

[Step 2] `postgres` 컨테이너에 접속한다.

```shell
[yunjin@localhost supabase]$ docker exec -it 6a96e18b188b /bin/bash
root@6a96e18b188b
```

[Step 3] `postgres` 사용자로 접속한다.

**supabase_admin의 password에 오류가 나서, supabase_admin 계정으로 접속해서 비밀번호를 변경해야 하는데, supabase_admin의 password가 initialize되지 않았다거나, 틀렸다는 오류가 나면 postgres로 접속한다.**

```shell
root@6a96e18b188b:/# psql -U supabase_admin
psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  password authentication failed for user "supabase_admin"
root@6a96e18b188b:/# psql -U postgres
psql (15.5 (Ubuntu 15.5-1.pgdg20.04+1), server 15.1 (Ubuntu 15.1-1.pgdg20.04+1))
Type "help" for help.

postgres=#
```

[Step 4] `pgpass`를 원하는 비밀번호로 설정하고, password 오류가 났던 사용자의 비밀번호를 `pgpass`로 변경한다.

```shell
postgres=# \set pgpass ***원하는 비밀번호***
postgres=# ALTER USER supabase_admin WITH PASSWORD :'pgpass';
ALTER ROLE
postgres=# ALTER USER supabase_storage_admin WITH PASSWORD :'pgpass';
ALTER ROLE
postgres=# exit
```

[Step 5 (Optional)] `supabase_admin` password를 설정했으므로 `supabase_admin` 계정으로 접속이 가능한지 확인한다.

```shell
root@6a96e18b188b:/# psql -U supabase_admin
psql (15.5 (Ubuntu 15.5-1.pgdg20.04+1), server 15.1 (Ubuntu 15.1-1.pgdg20.04+1))
Type "help" for help.

postgres=# exit
root@6a96e18b188b:/# exit
exit
```

<br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
