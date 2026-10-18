---
title: "[Docker] Docker 오류와 해결 방법 모음.zip"
author:
date: 2024-04-09 13:30:00 +0900
categories: [Infra & DevOps, Infra]
tags: [Tips, Docker, 인턴]
---

## **Docker daemon socket Permission denied**

원격 서버 rocky-linux에서 `docker ps` 명령어 실행 시, 아래와 같은 `docker daemon` 오류를 내며 실행할 수 없었다. 혹은 `root`가 아니라 일반 사용자가 docker 명령어를 실행할 때 이런 오류가 발생했다.

```shell
[rumos@rocky-linux deploy]$ docker ps
permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.45/containers/json": dial unix /var/run/docker.sock: connect: permission denied
```

### **해결 방법**

- `sudo usermod -aG docker mint` 으로 docker 명령어를 실행하려는 mint 계정에 권한 부여
- `exit` 후 ssh로 재접속 하면 정상적으로 docker 명령어 실행 가능

<br>

## **Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?**

docker 명령어를 실행했을 때 daemon running 에러가 발생했다. 로컬인 경우 docker desktop으로 간단하게 서비스를 실행할 수 있지만, 원격으로 접속하는 경우 CLI만 확인 가능하기 때문에 서비스 실행 여부를 확인하기 어렵다.

```shell
$ docker ps
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

### **해결 방법**

`sudo systemctl status docker`로 서비스의 상태를 확인한다. `disabled` 이거나 `stop`이면 daemon running 오류가 발생한다!

```shell
$ sudo systemctl status docker
[sudo] ...의 암호:
○ docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; **disabled**; preset: **disabled**)
     Active: inactive (dead)
TriggeredBy: ○ docker.socket
       Docs: https://docs.docker.com
```

`docker start, enable`로 서비스를 시작한다.

```shell
$ sudo systemctl start docker
$ sudo systemctl enable docker
Created symlink /etc/systemd/system/multi-user.target.wants/docker.service → /usr/lib/systemd/system/docker.service.
```

<br>

## **Docker logs : exec format error**

컨테이너 실행은 문제없이 되는 것처럼 보이지만, 계속 재시작하고 `Docker logs`로 확인하면 `exec format error`가 발생

```shell
[root@localhost supabase]# docker ps
CONTAINER ID   IMAGE                              COMMAND                    CREATED          STATUS                                     PORTS                                       NAMES
fe1c605a23f0   supabase/studio:20240326-5e5586d   "docker-entrypoint.s…"    12 minutes ago   Restarting (1) Less than a second ago                                                  studio
3c8edd16c52b   supabase/storage-api:v0.46.4       "docker-entrypoint.s…"    12 minutes ago   Restarting (1) Less than a second ago                                                  storage
c31e3f6c768a   supabase/postgres:15.1.0.147       "docker-entrypoint.s…"    12 minutes ago   Up Less than a second (health: starting)   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   db
```

```shell
[root@localhost supabase]# docker logs fe1c605a23f0
exec /usr/local/bin/docker-entrypoint.sh: exec format error
exec /usr/local/bin/docker-entrypoint.sh: exec format error
exec /usr/local/bin/docker-entrypoint.sh: exec format error
exec /usr/local/bin/docker-entrypoint.sh: exec format error
exec /usr/local/bin/docker-entrypoint.sh: exec format error
exec /usr/local/bin/docker-entrypoint.sh: exec format error
exec /usr/local/bin/docker-entrypoint.sh: exec format error
exec /usr/local/bin/docker-entrypoint.sh: exec format error
exec /usr/local/bin/docker-entrypoint.sh: exec format error
exec /usr/local/bin/docker-entrypoint.sh: exec format error
exec /usr/local/bin/docker-entrypoint.sh: exec format error
```

### **해결 방법**

✅ Docker 이미지를 운영체제/CPU에 맞게 받았는지 확인한다. `docker-compose.yml`에 이미지를 지정했다면, `platform`을 설정해 본인의 개발 환경에 맞는 이미지를 받아야 한다.

`192.168.@@.@@` (개발 서버)의 플랫폼 설정은 `linux/amd64`이고, macOS apple chip인 로컬의 플랫폼 설정은 `linux/arm64`이다!

<br>
<br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
