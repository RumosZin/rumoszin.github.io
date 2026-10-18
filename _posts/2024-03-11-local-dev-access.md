---
title: "[Tips] 로컬 컴퓨터에서 원격 서버 접속하기 (feat. key-gen 에러)"
author:
date: 2024-03-11 13:30:00 +0900
categories: [Infra & DevOps, Infra]
tags: [Tips, 인턴]
---

## **로컬에서 .ssh/config 파일 생성**

### **.ssh 경로 확인**

```shell
# .ssh 파일 확인
$ ls -al
$ cd .ssh
$ vi config
```

### **config 파일 작성**

```shell
# dev_server
host dev_server
	user rumos
	hostname 192.168.@@.@@
```

### **.ssh 폴더 권한 설정**

루트 경로에서 `ls -al` 명령어를 이용해 `.ssh` 폴더의 권한을 확인하고, `read`, `execute` 권한을 추가한다.

```shell
# before
$ ls -al
total N
...
drwx------   4 {root_name}  staff    128  3 11 10:58 .ssh
...

# read, execute
$ chmod +rx .ssh
$ ls -al
total N
...
rwxr-xr-x   4 {root_name}  staff    128  3 11 10:58 .ssh
...
```

## **원격 CentOS 접속**

config에 `user`를 `rumos`, `hostname`을 `192.168.@@.@@`로 설정했기 때문에, `ssh rumos@192.168.@@.@@` 명령어와 같다.

```shell
$ ssh dev_server
rumos@192.168.@@.@@'s password:
Last login: Mon Mar 11 11:04:26 2024 from gimyunjcbookpro
[rumos@RUMOS-CentOS ~]$
```

### **key-gen 오류**

`ssh dev_server`로 `192.168.@@.@@`에 접속했는데, 원격 서버를 **centOS**에서 **rocky-linux**로 변경한 후 아래와 같은 오류가 발생하며 한 번에 접속할 수 없었다.

```bash
$ ssh dev_server
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
IT IS POSSIBLE THAT SOMEONE IS DOING SOMETHING NASTY!
Someone could be eavesdropping on you right now (man-in-the-middle attack)!
It is also possible that a host key has just been changed.
The fingerprint for the ////// key sent by the remote host is
SHA256://///.
Please contact your system administrator.
Add correct host key in /Users/gim-yunjin/.ssh/known_hosts to get rid of this message.
Offending ECDSA key in /Users/gim-yunjin/.ssh/known_hosts:2
Host key for 192.168.@@.@@ has changed and you have requested strict checking.
Host key verification failed.
```

### **해결 방법**

- `ssh-keygen -R {host IP}` 로 기존의 key를 삭제한다.
  - ex) `ssh-keygen -R 192.168.@@.@@`
- `ssh dev_centos`로 다시 접속하고 connecting 질문에 `yes` 입력하면 정상적으로 접속할 수 있다.

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
