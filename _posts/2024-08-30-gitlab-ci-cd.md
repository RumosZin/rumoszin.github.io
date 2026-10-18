---
title: "[Gitlab CI/CD 3] CD까지 자동화하기"
author:
date: 2024-08-30 20:00:00 +0900
categories: [Infra & DevOps, DevOps]
tags: [인턴, gitlab]
---

<br>

지난 주간회의에서 받은 피드백을 반영하여 배포 자동화 프로세스를 마무리하자!

> Gitlab은 CI/CD 도구인데, 현재는 CI까지만 Gitlab의 도움을 받는다. 서버에서 크론탭으로 스크립트를 실행 시킬 필요없이, **내부 서버에 접속해서 스크립트를 실행하는 것까지 Gitlab의 도움을 받아서 해결할 수 있다.** 여기까지 시도해보는 것은 어떤지?

<br>

[[Gitlab CI/CD 1] CI 적용하고 CD는 비동기 처리하기](https://rumoszin.github.io/posts/gitlab-ci/)

[[Gitlab CI/CD 2] 환경 변수 분리하기]([Gitlab CI/CD 2] 환경 변수 분리하기)

## **gitlab-ci.yml**

스크립트의 deploy 부분을 작성했다. (전문은 길지만 deploy 부분만 가져왔다.)

```yaml
deploy-qa:
  image: ubuntu:latest
  stage: deploy
  before_script:
    - "command -v ssh-agent >/dev/null || ( apt-get update -y && apt-get install openssh-client -y )"
    - eval $(ssh-agent -s)
    - cat $SSH_PRIVATE_KEY > key.pem
    - chmod 600 key.pem
    - mkdir -p ~/.ssh
    - touch ~/.ssh/config
    - touch ~/.ssh/known_hosts
    - chmod -R 400 ~/.ssh
    - ssh-keyscan -p 6022 @@@@.synology.me >> ~/.ssh/known_hosts
  script:
    - ssh -tt -i key.pem -o StrictHostKeyChecking=no -p 6022 ******@******.synology.me /home/******/deploy.sh
```

<br>

## **SSH 공개키 인증**

**✅ Gitlab 서버가 온프레미스 서버에 접속하기 위해서 SSH 관련 작업을 해야한다. 일반 사용자가 접속할 때는 비밀번호를 입력 받을 수 있지만, Gitlab 서버가 클라이언트이기 때문에 설정이 필요하다.**

이 부분에서 SSH 공개키 인증에 대한 내용이 부족하다고 느껴져서 공부를 했다. 가장 깔끔히 정리하고 있는 블로그를 첨부한다! [SSH 공개키 인증을 사용하여 접속하기](https://velog.io/@lehdqlsl/SSH-%EA%B3%B5%EA%B0%9C%ED%82%A4-%EC%95%94%ED%98%B8%ED%99%94-%EB%B0%A9%EC%8B%9D-%EC%A0%91%EC%86%8D-%EC%9B%90%EB%A6%AC-i7rrv4de)

**ssh-keygen으로 공개키와 개인키를 생성해야 한다. Gitlab CI/CD 이슈 중에 id_rsa 방식을 이용하면 안된다는 글이 많았다. ed25519 알고리즘을 이용해서 생성하는 것을 추천한다. 생성한 키 중 개인 키를 Gitlab Variables로 등록한다.**

**배포할 서버에 접속해서 `.ssh/authorized_keys`에 아래 내용을 추가하면 좋다. 특정 키(앞서 생성한 공개키)를 가진 사용자가 접속하면 다른 모든 명령을 무시하고 `deploy.sh`만 실행한다.**

```shell
command="/home/******/deploy.sh" ssh-ed25519 AAAAC3NzaC@@@@@@@@@@@@@@@@ deploy_bot
```

<br>

### **before_script**

- 먼저 ssh-agent 명령이 설치되어 있는지 확인하고, 없는 경우 openssh-client를 설치한다.
- ssh-agent를 실행해서 현재 세션에서 사용할 수 있도록 한다.
- 미리 저장해둔 개인키를 key.pem에 기록하고 파일의 권한을 소유자만 읽고 쓸 수 있도록 설정한다.
- ssh 설정을 저장하기 위해 .ssh/config를 생성한다.
- known_hosts는 ssh 연결 시 접속한 호스트의 키를 저장한다. 연결 시 서버의 신뢰성을 보장한다!
- ssh-keyscan으로 ssh 서버의 호스트 키를 Known_hosts에 추가한다.

<br>

### **script**

```yaml
ssh -tt -i key.pem -o StrictHostKeyChecking=no -p 6022 ******@******.synology.me /home/******/deploy.sh
```

원격 서버에 로그인할 사용자의 계정 정보를 지정하고, SSH를 이용해서 원격 서버에 접속한다! Synology NAS 서버의 사용자 계정에 접근한다. 그리고 실행할 스크립트의 경로를 명시하면, 일반 사용자가 접속해서 스크립트를 실행하는 것처럼 Gitlab 서버가 스크립트를 실행한다!

<div style="text-align: center;">
    <img src="/assets/img/240830-1.png" alt="Untitled" style="width: 50%;">
</div>

<br>

## **팁 공유**

AWS, GCP 등 클라우드에 배포하는 내용은 많은데 온프레미스 서버에 배포하는 내용은 없어서 무수히 많은 블로그와 공식 문서를 참고했다...

- [Using SSH keys with GitLab CI/CD](https://docs.gitlab.com/ee/ci/jobs/ssh_keys.html)
- [Error loading key "/builds/path/SSH_PRIVATE_KEY": error in libcrypto message](https://docs.gitlab.com/ee/ci/jobs/ssh_keys.html#troubleshooting)
- [SSH 공개키 인증을 사용하여 접속하기](https://velog.io/@lehdqlsl/SSH-%EA%B3%B5%EA%B0%9C%ED%82%A4-%EC%95%94%ED%98%B8%ED%99%94-%EB%B0%A9%EC%8B%9D-%EC%A0%91%EC%86%8D-%EC%9B%90%EB%A6%AC-i7rrv4de)
- [Authentication failure / Permission denied (publickey, password), Server refused our key](https://yonlog.tistory.com/103#toc4)
- Permission denied key : 생성한 키를 ed-25519로 생성했는데도 이 문제가 발생한다면, 실행 스크립트의 권한이 없어서 그런 것일 수도 있다. deploy 스크립트, ssh 키값의 RWX를 점검하자. (개인키는 사용자만이 읽고 쓸 수 있고 600, 공개키는 다른 사용자도 읽을 수 있는 644 권한을 가지고 있다.)

<br>
<br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
