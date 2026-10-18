---
title: "[Supabase] supabase-db 컨테이너에서 storage-bucket 확인하기"
author:
date: 2024-04-23 21:00:00 +0900
categories: [Backend, Database]
tags: [Database, Supabase, 인턴]
---

인턴십을 진행하면서 데이터베이스는 MariaDB, 스토리지는 Supabase를 사용하고 있다. (회사에서 이전에 진행했던 프로젝트에서 Supabase 스토리지를 cloud를 이용해서 개발자가 모두 공동 데이터베이스에 접근할 수 있도록 설정한 후 개발했다고 한다.
이 방식이 모두의 데이터베이스가 동기화 된다는 점에서 좋았으나 A와 B가 이용하고 있는 테이블 정보가 충돌해서 둘 다 변경된 정보를 반영하려고 할 때 데이터가 날아가버리는 현상이 있었다고 한다. )

이번에 인턴십을 하며 참여하는 프로젝트에서 Supabase는 스토리지 역할만 한다! 또한 각자 로컬 도커에 Supabase 이미지를 띄우고 작업하므로 앞선 충돌 상황이 발생하지 않을 것으로 예상한다. (bucket만 사용하고 데이터베이스는 MariaDB를 이용한다.)

<br>

### **Supabase Self-hosting with Docker**

[Self-Hosting으로 로컬 도커에 Supabase를 세팅하는 방법은 이전에 레포지토리에 정리한 적이 있다!](https://github.com/RumosZin/supabase-self-hosting-docker) 궁금한 사람들은 확인해보면 좋을 것 같다.

레포지토리에 나온 가이드대로 따라한다면, `localhost:8000`으로 접속해서 supabase studio를 이용할 수 있다. **✅ 로컬에 supabase를 세팅하고나서는 supabase studio라는 편리한 도구를 이용해 확인하면 되는데, 만약 내부 서버에 supabase를 self-hosting 한다면 studio로 확인할 수 없다. 이런 경우에 직접 supabase storage 관련 컨테이너에 들어가서 내부망의 storage-bucket을 확인할 수 있다.**

서론이 길었지만 여러 명령어를 기억하기 위해 글을 남긴다.

<br>

### **접속**

```shell
# psql -U postgres
psql (15.5 (Ubuntu 15.5-1.pgdg20.04+1), server 15.1 (Ubuntu 15.1-1.pgdg20.04+1))
Type "help" for help.

postgres=#
```

<br>

### **schema list**

- pgbouncer: pgbouncer 사용자가 소유한 pgbouncer라는 스키마이다. 이는 데이터베이스 연결 풀링을 제공하는 PgBouncer와 관련된 객체들을 포함할 수 있다.
- public: pg_database_owner 사용자가 소유한 public 스키마이다. public은 기본 스키마로, 명시적으로 다른 스키마를 지정하지 않을 경우 이 스키마가 사용된다.
- storage: supabase_admin 사용자가 소유한 storage 스키마이다. 스토리지와 관련된 데이터 객체들이 이 스키마에 포함될 수 있다.

```shell
postgres=# \dn
        List of schemas
    Name    |       Owner
------------+-------------------
 auth       | supabase_admin
 extensions | postgres
 pgbouncer  | pgbouncer
 public     | pg_database_owner
 storage    | supabase_admin
(5 rows)
```

<br>

### **database list**

```shell
postgres=# \list
                                                List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  | ICU Locale | Locale Provider |      Access privileges
-----------+----------+----------+---------+---------+------------+-----------------+-----------------------------
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =Tc/postgres               +
           |          |          |         |         |            |                 | postgres=CTc/postgres      +
           |          |          |         |         |            |                 | dashboard_user=CTc/postgres
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres                +
           |          |          |         |         |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 |            | libc            | =c/postgres                +
           |          |          |         |         |            |                 | postgres=CTc/postgres
(3 rows)
```

<br>

### **relation list**

```shell
postgres=# \dt
                   List of relations
 Schema  |    Name    | Type  |         Owner
---------+------------+-------+------------------------
 storage | buckets    | table | supabase_storage_admin
 storage | migrations | table | supabase_storage_admin
 storage | objects    | table | supabase_storage_admin
(3 rows)
```

<br>

### **relation select**

```shell
postgres=# set search_path to storage;
SET
```

<br>

### **show all bucket's from storage.bucket**

```shell
postgres=# select * from buckets;
  id   | name  | owner |          created_at          |          updated_at          | public | avif_autodetection | file_size_limit | allowed_mime_types | owner_id
-------+-------+-------+------------------------------+------------------------------+--------+--------------------+-----------------+--------------------+----------
 user | user |       | 2024-04-23 01:37:26.73349+00 | 2024-04-23 01:37:26.73349+00 | t      | f                  |                 |                    |
(1 row)
```

<br>

### **insert new bucket**

- `ON CONFLICT (id) DO NOTHING` : id 칼럼이 충돌할 경우 아무 작업도 하지 않고 넘어간다.

```shell
postgres=# INSERT INTO storage.buckets (id, name, public) VALUES ('new_bucket', 'new_bucket', true) ON CONFLICT (id) DO NOTHING;
INSERT 0 1
```

<br>

### **select all data from bucket**

```shell
postgres=# SELECT * FROM storage.objects WHERE bucket_id = (SELECT id FROM storage.buckets WHERE name = 'user');
 id | bucket_id | name | owner | created_at | updated_at | last_accessed_at | metadata | path_tokens | version | owner_id
----+-----------+------+-------+------------+------------+------------------+----------+-------------+---------+----------
(0 rows)
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
