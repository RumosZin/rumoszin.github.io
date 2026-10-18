---
title: "[Tips] IntelliJ 오류와 해결 방법.zip"
author:
date: 2024-01-02 23:03:00 +0900
categories: [Backend, Spring]
tags: [Tips, IntelliJ]
---

### **Main 메서드를 실행할 수 없을 때**

`command + ;`, `command + ,`을 통해 JDK 설정을 확인했음에도 불구하고 main 메서드를 인식하지 못하는 경우가 있다.

![Untitled](/assets/img/240422-1.png){: width="100%"}

만약 Spring Boot 프로젝트를 [start.spring.io](https://start.spring.io/)에서 만들었다면, 만든 프로젝트의 루트 디렉터리에서 열어야 `build.gradle`을 인식할 수 있다.

예를 들어서 만든 Spring boot 프로젝트 `demo`를 다른 디렉터리 `root_example`에 넣었다면, `root_example`로 프로젝트를 열지 않도록 주의한다!!

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
