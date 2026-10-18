---
title: "[Supabase/Docker] Supabase 컨테이너 password 에러 : `.env` 확인하기"
author:
date: 2024-03-11 13:30:00 +0900
categories: [Backend, Database]
tags: [Database, Supabase, Docker, 인턴]
---

## **상황**

`docker-compose.yml` 파일은 [공식 문서의 파일](https://github.com/supabase/supabase/blob/master/docker/docker-compose.yml)을 그대로 참고했다.

`docker compose up -d --build` 명령어로 컨테이너를 실행했을 때, **supabase-analytics** 컨테이너에서 waiting이 길어지고 결국 에러가 발생했다.

```shell
01:36:45.559 [error] Postgrex.Protocol (#PID<0.163.0>) failed to connect: ** (Postgrex.Error) FATAL 28P01 (invalid_password) password authentication failed for user "supabase_admin"
...
2024-03-13 10:54:52 supabase-analytics  | ** (DBConnection.ConnectionError) connection not available and request was dropped from queue after 10963ms. This means requests are coming in and your connection pool cannot serve them fast enough. You can address this by:
2024-03-13 10:54:52 supabase-analytics  |
2024-03-13 10:54:52 supabase-analytics  |   1. Ensuring your database is available and that you can connect to it
2024-03-13 10:54:52 supabase-analytics  |   2. Tracking down slow queries and making sure they are running fast enough
2024-03-13 10:54:52 supabase-analytics  |   3. Increasing the pool_size (although this increases resource consumption)
2024-03-13 10:54:52 supabase-analytics  |   4. Allowing requests to wait longer by increasing :queue_target and :queue_interval
```

위 에러에서 다른 것보다 이 부분이 눈에 들어왔다. **`supabase_admin`의 비밀번호가 틀렸다고 한다.**

```shell
FATAL 28P01 (invalid_password) password authentication failed for user "supabase_admin"
```

<br>

## **해결 방법 : `.env` 확인하기**

`docker-compose.yml`은 `.env`의 환경 변수를 이용한다. `.env`에 `POSTGRES_PASSWORD`가 올바르게 설정되어 있는지 확인한다. 처음 공식 문서의 코드를 받으면 `.env.example`을 `.env`에 copy 해서 사용하는데, copy 후 첫 실행 전에 비밀번호를 바꾸는 것을 추천한다.

추후 supabase_storage_admin, authenticator, postgres .. 등 많은 user에서 password 에러가 날 수 있는데, 이때는 비밀번호를 변경하는 방식으로 해결할 수 있다.

<br>

## **`.env` 파일을 읽지 못하는 경우**

### **`.env`의 위치 확인하기**

[docker 공식 문서의 local .env vs .env file](https://docs.docker.com/compose/environment-variables/variable-interpolation/?highlight=envlocal#local-env-file-versus-project-directory-env-file)에 따르면, `COMPOSE_FILE`이라는 미리 정의된 변수를 정의하여 프로젝트 디렉토리가 다른 폴더로 설정되는 경우, compose는 두 번째 `.env` 파일을 로드하는데, 이 파일이 존재할 경우 우선순위가 낮아진다.

명령어로 `.env`의 위치를 명시할 것이 아니라면, `.env`를 `docker-compose.yml`과 같은 위치에 존재하도록 했는지 확인한다.

### **우선순위 이용하기**

[docker 공식 문서의 Environment variables precedence in Docker Compose](https://docs.docker.com/compose/environment-variables/envvars-precedence/)에 따르면, CLI에서 `docker compose run -e`를 사용해서 설정하는 것이 가장 우선순위가 높고 컨테이너 이미지의 `Dockerfile`에서 `env`를 지정할 때가 가장 낮다.

<br>

## **나의 선택**

여러 `.env`가 `docker-compoe.yml`과 같은 경로에 있다면, 사용할 env file을 명시하기

**[첫번째 선택]**

공식 문서의 우선순위를 통해 명시적으로 `.env`를 가장 높은 CLI에서 지정했다. **local, dev, prod** 등 항상 CLI에서 지정하는 것은 오타날 가능성도 높았고, 무엇보다 항상 파일을 지정하고 이 점을 상기해야 한다는 점에서 번거로웠다.

**☑️ [두번째 선택]**

`build_local.sh`, `build_dev.sh`, `build_prod.sh`를 작성하고, 그 안에 서로 다른 명령어들을 넣어주었다. 어떤 `.env`를 적용해야할 지 고민하지 않아도 될 뿐더러, docker 명령어를 찾을 필요가 없다. **무엇보다, docker에 대해 잘 모르는 다른 사람이 빌드해야 할 때 유용했다.**

```shell
docker compose --env-file .env.local up -d
```

{: file='build_local.sh'}

<br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
