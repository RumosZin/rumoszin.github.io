---
title: "scp 명령어 실행 시 Permission denied 오류와 해결 방법"
author:
date: 2024-04-10 18:30:00 +0900
categories: [Infra & DevOps, Infra]
tags: [Tips, Server, 인턴]
---

<br>

## **scp로 원격 서버 접속 시 Permission denied**

로컬에서 원격으로 이미지 압축 파일이나 폴더를 보내려는데 scp 명령어에서 Permission denied 오류가 발생했다.

```shell
$ scp -r ./supabase/deploy user@hostip:~/project/supabase/
user@hostip's password:
scp: dest open "/home/project/deploy/deploy_dev.sh": Permission denied
```

<br>

## **해결 방법**

### **[방법 1] 폴더, 파일의 읽기, 쓰기 권한을 확인한다**

- r : read / w : write / x : exec
- `chmod -R 755 {directory name}` : 모든 하위 폴더와 파일의 권한을 읽기 및 실행으로 변경
- `chmod -R 775 {directory name}` : 모든 하위 폴더와 파일의 권한을 읽기, 쓰기 및 실행으로 변경
- 아래와 같이 권한이 있다면 문제 없다.

```shell
$ ls -al
합계 0
drwxr-xr-x. 4 user user 73  4월  9 15:03 mariadb
drwxr-xr-x. 4 root root 73  4월  9 15:31 supabase
```

### **[방법 2] 파일 및 디렉토리의 소유자, 소유자 그룹 변경**

위의 `ls -al` 명령어 결과 mariadb 폴더는 소유자와 소유자 그룹이 user이고, supabase는 root이다. supabase 폴더에 접근하려고 하면 발생했던 Permissino denied를 해결하기 위해, supabase 폴더의 소유자, 소유자 그룹을 user로 변경한다.

```shell
$ chown mint supabase # 소유자 변경
$ chown :mint supabase # 소유자 그룹 변경
```

<br>

## **root에서 docker 명령어를 실행하면..**

docker 이미지를 띄울 때 root에서 명령어를 실행하면, docker 명령어로 생성되는 폴더들의 소유자와 소유자 그룹도 root로 설정된다. docker 명령어를 root에서 실행하지 않도록 주의한다.

만약 root에서만 docker 명령어를 실행해야 하는 오류가 난다면 사용자를 추가해서 오류를 해결할 수 있고, 관련해서 [이 글](https://rumoszin.github.io/posts/docker-daemon-socket-error/)을 참고하기 바란다.

<br>
<br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
