---
title: "[Gitlab CI/CD 2] 환경 변수 분리하기"
author:
date: 2024-08-05 20:00:00 +0900
categories: [Infra & DevOps, DevOps]
tags: [인턴, gitlab]
---

<br>

지난 주간회의에서 받은 피드백을 반영하여 배포 자동화 프로세스를 개선하자!

> 빌드 부분을 자세히 설명하길래 gitlab-ci.yml 파일을 봤는데, **실행에 필요한 환경 변수까지 빌드 단계에서 주입하는 것은 지양해야 한다.** 현재는 Dockerfile 자체가 .env의 모든 내용을 이용하도록 설계되어 있는데, 변수를 분리해라.

## **.env**

- NODE_ENV
  - production, development
- ORIGIN
  - 운영서버 환경변수에서 tss domain을 적으면 ORIGIN domain에서 온 post 요청만 허용한다.
- MariaDB 관련 환경변수
  - lucia.ts, config.js : `process.env.MARIADB_HOST` → 실행 시에 필요한 환경 변수!
- Supabase 관련 환경변수
  - supabase.js : `import { PUBLIC_SUPABASE_URL, PUBLIC_SUPABASE_SERVICE_ROLE_KEY } from "$env/static/public";` → 빌드 시에 필요한 환경 변수!
- `BODY_SIZE_LIMIT=Infinity`
  - 실행 환경 변수에 포함하지 않으니 아래와 같은 오류 Payload Too Large 오류가 발생했다.

```bash
2024-08-07 00:51:22:SS +00:00: **SvelteKitError: Content-length of 4706173 exceeds limit of 524288 bytes.**
2024-08-07 00:51:22:SS +00:00:     at Object.start (file:///app/build/handler.js:984:19)
2024-08-07 00:51:22:SS +00:00:     at setupReadableStreamDefaultController (node:internal/webstreams/readablestream:2435:23)
2024-08-07 00:51:22:SS +00:00:     at setupReadableStreamDefaultControllerFromSource (node:internal/webstreams/readablestream:2467:3)
2024-08-07 00:51:22:SS +00:00:     at new ReadableStream (node:internal/webstreams/readablestream:278:7)
2024-08-07 00:51:22:SS +00:00:     at get_raw_body (file:///app/build/handler.js:973:9)
2024-08-07 00:51:22:SS +00:00:     at getRequest (file:///app/build/handler.js:1054:7)
2024-08-07 00:51:22:SS +00:00:     at Array.ssr (file:///app/build/handler.js:1248:19)
2024-08-07 00:51:22:SS +00:00:     at handle (file:///app/build/handler.js:1318:23)
2024-08-07 00:51:22:SS +00:00:     at file:///app/build/handler.js:1318:40
2024-08-07 00:51:22:SS +00:00:     at Array.<anonymous> (file:///app/build/handler.js:1237:4)
2024-08-07 00:51:22:SS +00:00:     at handle (file:///app/build/handler.js:1318:23)
2024-08-07 00:51:22:SS +00:00:     at file:///app/build/handler.js:1318:40
2024-08-07 00:51:22:SS +00:00:     at Array.<anonymous> (file:///app/build/handler.js:682:28)
2024-08-07 00:51:22:SS +00:00:     at handle (file:///app/build/handler.js:1318:23)
2024-08-07 00:51:22:SS +00:00:     at Array.<anonymous> (file:///app/build/handler.js:1324:10)
2024-08-07 00:51:22:SS +00:00:     at loop (file:///app/build/index.js:224:63) {
2024-08-07 00:51:22:SS +00:00:   status: 413,
2024-08-07 00:51:22:SS +00:00:   text: 'Payload Too Large'
2024-08-07 00:51:22:SS +00:00: }
```

위와 같이 빌드/실행 각각에 필요한 환경 변수를 확인했으므로 아래와 같이 구분해서 변수를 넣어준다.

- **Gitlab CI/CD Variables에는 빌드 시에 필요한 환경 변수**
- **deploy\_{STAGE}.sh의 docker run에는 실행 시에 필요한 환경 변수**

**✅ 이제 이를 잘 반영하도록 Dockerfile, build 스크립트, deploy 스크립트를 수정하자! 사내 온프레미스 서버에 배포할 때는 Gitlab CI/CD를 이용할 수 있으나, 협력 업체의 온프레미스 서버에 배포할 때는 VPN이 필요하기 때문에 Gitlab CI의 도움만 받는다.**

<br>

## **Dockerfile**

bun to npm

### **step 1. AS build**

- **✅ migrate 단계가 포함되어 있었는데, migrate는 빌드 단계에서 하지 않아도 되므로 삭제한다.**
- `npm run build:${STAGE}`로 설정하면 `package.json`의 scripts에서 `build:staging`을 참고해서 `vite build --mode staging` 명령어를 실행한다.
  - mode가 staging이면 빌드할 때 .env.staging 환경변수를 이용한다.

~~사수께 코드 업로드 허락을 받았으나 아무래도 회사에서 이용되는 코드를 공개적인 곳에 올리기는 무리가 있다고 생각해서 코드는 생략한다.~~

```
# package.json / scripts
    "build:qa": "vite build --mode qa",
    "build:staging": "vite build --mode staging",
    "build:prod": "vite build --mode prod",
```

### **step 2. stage**

- .env.{STAGE}를 copy 하는 코드를 제외시켰다.

<br>

## **build_staging.sh**

> **로컬에서 Gitlab에 접근해서 이미지를 가져오고, 이를 압축해서 협력 업체의 개발서버, 배포서버로 전송하는 역할을 한다.**

- 이미지 이름을 @@@@@@-web:staging-current로 고정시킬 것이기 때문에 기존에 존재하던 이미지는 삭제한다.
- Gitlab에 로그인해서 @@@@@@@@-staging-new 이름의 이미지를 다운받고 이미지 태그를 변경한다.
  - 이미지 이름을 @@@@@@-web:staging-current로 변경하지 않으면 회사의 깃랩 주소가 이미지 태그에 노출된다. 반드시 변경한다!
- **✅ `bun dotenv -e .env.staging -- sequelize-cli db:migrate` migration을 빌드 단계에서 제외했으므로 이 단계에서 실행시킨다.**
- 이후로는 기존과 같이, 빌드된 이미지를 tgz로 압축한 후 협력 업체의 개발서버, 배포서버로 전달한다.

내가 직접 짠 스크립트이지만, 아무래도 회사에 있는 코드이기 때문에 주석만 남기려고 한다.

```bash
# config

# gitlab access

# pull new image

# sequelize-cli db migration

# zip image and send by using scp
```

<br>

## **deploy_staging.sh**

> **협력업체의 개발서버에 접속해서 기존의 이미지는 삭제하고, 최근 받은 이미지를 실행시키는 스크립트이다.**

- 이미지 : old rmi → current to old → current load.
- 컨테이너 : current rm → docker run
  - docker run에 실행 시 필요한 환경 변수를 지정해야한다!!
- **prod도 staging과 유사하지만 network를 host로 지정하는 것에 유의!**

내가 직접 짠 스크립트이지만, 아무래도 회사에 있는 코드이기 때문에 주석만 남기려고 한다.

```bash
# config

# remove old image

# rename current image to old image

# load new current image

# docker run!
```

- docker images

```bash
$ docker images
REPOSITORY               TAG                IMAGE ID       CREATED         SIZE
@@@@@@-web               staging-current    a0740f41@@@@   18 hours ago    698MB
@@@@@@-web               staging-old        94486940@@@@   19 hours ago    698MB
```

- docker ps

```bash
$ docker ps
CONTAINER ID   IMAGE                              COMMAND                   CREATED             STATUS                   PORTS                                       NAMES
f3eefb3b5130   @@@@@@-web:staging-current         "docker-entrypoint.s…"    About an hour ago   Up About an hour         0.0.0.0:80->3000/tcp, :::80->3000/tcp       safety-web
```

<br>
<br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
