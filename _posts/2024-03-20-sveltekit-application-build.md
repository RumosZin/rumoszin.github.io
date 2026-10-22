---
title: "[SvelteKit/Docker] SvelteKit application을 docker로 빌드하기"
author:
date: 2024-03-20 20:30:00 +0900
categories: [Infra & DevOps, Infra]
image: /assets/img/240320-1.png
tags: [Database, Supabase, Docker, 인턴]
---

SvelteKit application은 Node.js app 이기도 해서, Node.js를 dockerizing 하는 것과 SvelteKit app을 빌드하는 과정이 비슷하다. 따라서 `node.js docker build`로 검색해야 자료가 더 풍부하다. (SvelteKit 빌드 과정만을 찾아다니던 사람들에게 팁...)

- [https://www.okupter.com/blog/build-a-sveltekit-application-with-docker](https://www.okupter.com/blog/build-a-sveltekit-application-with-docker)

### **Dockerfile**

At a high level, Dockerizing an application is just writing some instructions about how we want to build and run it.

These instructions are written in a declarative language called Dockerfile. The Dockerfile is then used to build a Docker image, a snapshot of the application at a given time. The image can then be used to run a container, which is the actual running instance of the application.

### **.dockerignore**

`.dockerignore`에 빌드하지 않을 파일을 포함하지 않으면, SvelteKit app 이미지의 용량이 커질 수 있다. 빌드에 상관없는 파일들을 꼭 포함시키도록 하자. 아래 예시는 간단한 `.dockerignore` 예시이다.

**✅ 나의 경우에, Self-Hosting으로 이미지를 띄우기 때문에 로컬에서 개발 서버로 이미지를 보내야 했는데, 이때 이미지를 빌드할 때 사용한 빌드 스크립트나 배포 스크립트가 포함되지 않게 주의했다.**

```shell
node_modules
Dockerfile*
docker-compose*
.dockerignore
.git
.gitignore
README.md
LICENSE
.vscode
Makefile
helm-charts
.editorconfig
.idea
coverage*
```

### **[1] Node/Auto adapter**

- `svelte.config.js`에서 `import adapter from "@sveltejs/adapter-node";`을 확인한다.

```shell
FROM node:alpine

WORKDIR /app
COPY package.json ./
RUN npm install

COPY . .
RUN npm run build

CMD ["node", "build"]

EXPOSE 3000
```

{: file='Dockerfile'}

### **[2] Bun adapter**

- `svelte.config.js` 에서 `import adapter from "svelte-adapter-bun";`을 확인한다.

```shell
FROM oven/bun

WORKDIR /app
COPY package.json package.json
RUN bun install

COPY . .
RUN bun run build

EXPOSE 3000
ENTRYPOINT ["bun", "./build"]
```

- [https://bun.sh/guides/ecosystem/sveltekit](https://bun.sh/guides/ecosystem/sveltekit)
- [한 Bun 써보는 거 어때?](https://techblog.gccompany.co.kr/%ED%95%9Cbun-%EC%8D%A8%EB%B3%B4%EB%8A%94-%EA%B1%B0-%EC%96%B4%EB%95%8C-fa3cb32ac76f)
- Bun은 Node와 같은 Javascript 런타임 및 패키지 관리자인데, 속도 측면에서 npm 보다 낫다.
- **✅ 지금은 간단한 Sveltekit 애플리케이션을 빌드하는 것이니까 npm의 속도를 감당할 수 있지만, 개발을 하며 규모가 커지면 npm의 속도가 느려 빌드하는 시간이 길어지고, 배포 주기도 늘어질 것이다. 이를 대비해서 빠르게 빌드할 수 있는 방법을 찾아보다 bun을 알게 되었다.**

![Untitled](/assets/img/240320-1.png){: width="100%"}

### **Build Image**

[1] `.env` 설정

도커로 빌드하려는 SvelteKit 애플리케이션은 Supabase를 데이터베이스로 사용하고 있다. 애플리케이션을 빌드하고 실행시켰을 때 Supabase 컨테이너를 이용하도록 설정해야 한다.

**🔥 `Project-Supabase` 컨테이너의 `postgres`에 접근하기 위해서, SvelteKit 애플리케이션의 `POSTGRES_HOST`를 데이터베이스 컨테이너의 이름인 `supabase-db`로 지정해야 한다. 🔥**

`docker-compose.yml`에서 `name`을 설정해서 컨테이너들을 하나의 프로젝트로 묶을 수 있다. 이때, 프로젝트 안에 속한 컨테이너에 대해서, docker는 내부 IP를 컨테이너에 순차적으로 할당하는데, 이 IP는 컨테이너를 재시작할 때마다 변경될 수 있다.

**따라서 데이터베이스 컨테이너를 재시작해도 애플리케이션에 영향을 주지 않기 위해서, 호스트를 컨테이너의 IP가 아니라 컨테이너의 이름으로 설정해야 한다.**

(Optional) 컨테이너 재시작 시 IPv4Address 재할당 확인하기

- 존재하는 docker 네트워크 전체를 확인한다. `docker network ls` → supabase 컨테이너가 연결된 `NETWORK ID` , `NAME`을 확인한다.
- supabase 컨테이너의 네트워크를 확인한다. `docker network inspect {NETWORK ID}` → `supabase-db` 컨테이너의 `IPv4Address`를 확인한다.
- `.env` 파일의 `POSTGRES_HOST`, `CONNECTION_STRING` host를 앞서 확인한 `supabase-db`의 `IPv4Address`로 변경한다.

[2] Build image

`docker build -t {애플리케이션 이미지의 이름 지정} . `

(error) Dockerfile의 `RUN npm run build`에서 진행이 안될 때, `svelte.config.js`에서 adapter 설정을 확인한다.

- `import adapter from "@sveltejs/adapter-node";`
- `import adapter from "svelte-adapter-bun";`

### **Run the docker image**

**🔥 애플리케이션이 supabase-db 컨테이너를 이용하도록 `.env`를 설정하는 것뿐만 아니라, 두 컨테이너가 같은 네트워크에 연결되도록 해야 한다. 🔥**

`docker run -p 5273:3000 --network supabase_default --env-file .env --name safety-health-system-web-bun safety-health-system-web:local`

- `-p 5173:3000` : docker port에 연결할 호스트 포트 지정
  - `-p {host port} : {image port}`
  - Listening 0.0.0.0:{image port}, `localhost:5173`으로 접근
- `—network supabase_default`
  - `supabase-db`와 같은 네트워크에서 실행되도록 설정함
- `—env-file .env`
  - `.env` 파일을 사용하도록 지정한다. 로컬 / 개발 / 테스트 / 운영 이런 식으로 다양하게 서버를 운영할 때, 여러 `.env`가 존재할 수 있는데, 가장 우선순위가 높은 CLI 지정으로 사용할 환경 변수 파일을 명시할 수 있다.

### **삽질**

`supabase-db` 컨테이너를 통한 `postgres` 연결이 안될 때

- `.env` 설정을 맞게 했는데도 에러 메시지가 변하지 않는다면, **`.dockerignore`에 `.env`를 추가하지는 않았는지 확인한다.** (`.gitignore`와 헷갈려서 추가했을 수도 있다.)
- `.dockerignore`에 `.env`가 있으면 작업 디렉토리로 COPY할 때 `.env`를 제외한다. → `postgres_host`를 `supabase-db`으로 지정한 것을 반영하지 못한다. → 엉뚱한 127.0.0.1:5432에 접근한다.

localhost:5173에서 500:Internal Error가 발생할 때

- 이미지를 띄우는 `docker run` 명령어의 `—network`에 올바른 네트워크를 입력했는지 확인한다.

### **참고문서**

- [Containerize SvelteKit NodeJS with Docker](https://www.youtube.com/watch?app=desktop&v=kVMG2nWjWk4)
- [https://github.com/sdekna/sveltekit-dockerfiles](https://github.com/sdekna/sveltekit-dockerfiles)
- [https://stackoverflow.com/questions/66038165/error-connect-econnrefused-127-0-0-15432-when-connecting-with-nodejs-program](https://stackoverflow.com/questions/66038165/error-connect-econnrefused-127-0-0-15432-when-connecting-with-nodejs-program)

<br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
