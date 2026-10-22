---
title: "[Tips] Spring Boot 프로젝트 빌드하고 실행"
author:
date: 2024-01-03 11:30:00 +0900
categories: [Backend, Spring]
image: /assets/img/240103-1.png
tags: [Spring, Tips]
published: false
---

### **터미널로 프로젝트 경로 이동**

이동 후 아래 명령어를 순서대로 따라한다.

<br>

```shell
$ ./gradlew build
```

![Untitled](/assets/img/240103-1.png){: width="100%"}

<br>

```shell
$ cd build/libs
$ ls
```

![Untitled](/assets/img/240103-2.png){: width="100%"}

<br>

```shell
$ java -jar {project-name}-0.0.1-SNAPSHOT.jar
```

![Untitled](/assets/img/240103-3.png){: width="100%"}
