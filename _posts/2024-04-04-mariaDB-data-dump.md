---
title: "[MariaDB] 개발 서버 MariaDB 데이터 백업하기 (feat. mysqldump 명령어)"
author:
date: 2024-04-04 21:30:00 +0900
categories: [Backend, Database]
tags: [Database, MariaDB, 인턴]
---

## **상황**

현재 개발 서버에 MariaDB 이미지가 띄워진 상태이다. **개발 서버에서 테스트를 하던 도중 오류를 발견하면, MariaDB의 데이터를 내 로컬 환경으로 가져와서 확인하는 작업이 필요하다.**

## **[1] mysqldump 명령어로 데이터 백업 파일 생성**

아래의 명령어들은 모두 내 로컬 터미널에서 실행해야 한다! (처음에 개발 서버에서 sql 파일을 만들어서 저장하고, scp를 이용해 로컬로 보내야 하는 줄 알았다. 이 글을 읽는 분들은 실수하지 않도록 주의한다!)

데이터베이스의 이름은 `hello`이고, 테이블의 이름은 `world`라고 가정하자.

### **로컬 경로에 dump 파일 생성**

- 여러 번 dump 하는 경우가 생길 수 있어서, `date -I`를 이용해 파일 이름에 날짜가 들어가도록 했다.
- `mysqldump command not found` 에러가 생길 수 있는데, 로컬 mysql 설치가 선행되어야 한다.

```shell
mysqldump -u root -h -p --databases {database_name} | gzip > ~/dump-$(date -I).sql.tar.gz
```

- dump 파일을 이용해서 없어진 테이블을 되돌려보기 위해, 테이블 `world`를 삭제한다.

```shell
mysql > drop table world;
```

### **.gz 파일 압축 해제**

```shell
gunzip dump-$(date -I).sql.tar.gz
```

<br>

## **[2] 백업 파일의 데이터 확인**

- `mysql -u root -p < dump-$(date -I).sql.tar`
- `mysql -u root -p` 로 접속해서 데이터를 확인할 수 있다.

```shell
mysql > show databases;
mysql > use hello;
mysql[hello] > select * from world;
```

<br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
