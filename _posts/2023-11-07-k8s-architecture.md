---
title: "[Kubernetes] 쿠버네티스 아키텍처"
author:
date: 2023-11-07 11:30:00 +0900
categories: [Infra & DevOps, Infra]
tags: [Kubernetes]
---

**[쿠버네티스 입문:90가지 예제로 배우는 컨테이너 관리 자동화 표준]** 책의 4장 쿠버네티스 아키텍처 단원을 정리한 글입니다. Window10 운영체제에서 실습을 진행했습니다. 이론적인 내용이 중심인 글입니다.

✅ **4.2장에서는 주요 컴포넌트에 대해 다루고 있는데, 실습을 해보니 실제로 컴포넌트들을 만들고 본인의 프로젝트 상황에 맞게 운영하면서 컴포넌트들의 역할에 대해 체득하는 게 나을 것 같다. (물론 이론이 아예 안중요한 것 아니다.)**

<br>

# **4.1 쿠버네티스 클러스터의 전체 구조**

쿠버네티스 클러스터는 크게 두 종류의 서버로 구성된다.

- 마스터 - 클러스터를 관리
- 노드 - 실제 컨테이너를 실행

![Untitled](/assets/img/231107-1.png){: width="100%"}
[https://kubernetes.io/docs/concepts/architecture/](https://kubernetes.io/docs/concepts/architecture/)

마스터 안을 자세히 들여다 보자! 마스터 안에는 다양한 컴포넌트가 실행된다. (etcd, kube-apiserver, kube-scheduler, kube-controller-manager, kubelet, kube-proxy, docker 등의 컴포넌트들)

- 컴포넌트 각각이 다른 마스터나 노드 서버에서 실행되어도 실제 쿠버네티스 클러스터를 운영하는데 문제가 없다.
- 서버가 1대라면 프로세스 한 묶음을 해당 서버에서 같이 실행하는 것이 일반적인 구성이다.

**실제 사용하는 컨테이너의 대부분은 노드에서 실행된다.**

각 컴포넌트의 중심에 kube-apiserver가 있고, 쿠버네티스의 모든 통신은 kube-apiserver가 중심이다. 이것을 거쳐 다른 컴포넌트가 서로 필요한 정보를 주고 받는다. 특히 etcd에는 kube-apiserver만 접근 가능하다.

마스터에서, kublet이 마스터에 있는 도커를 관리하고, 도커 안에는 쿠버네티스 관리용 컴포넌트가 있다. etcd는 컨테이너가 아니라 별도의 프로세스로 설정되고, 컨테이너가 아닌 서버 프로세스로 실행하도록 구성할 수 있다.

노드도 kubelet으로 도커를 관리하고, kubelet은 마스터의 kube-apiserver와 통신하면서 파드의 생성, 관리, 삭제를 담당한다. (파드의 스펙을 이용하여 실행 및 헬스 체크를 한다.)

![Untitled](/assets/img/231107-3.png){: width="100%"}
[https://medium.com/cwan-engineering/fully-equipped-kubernetes-cluster-ee8a9f34601d](https://medium.com/cwan-engineering/fully-equipped-kubernetes-cluster-ee8a9f34601d)

<br>

# **4.2 쿠버네티스의 주요 컴포넌트**

쿠버네티스는 근본적으로 클러스터를 관리한다!

- 클러스터 : 단일 컴퓨터가 아니라 여러 대 컴퓨터를 하나의 묶음으로 다루는 것을 뜻하므로, 여러 가지 컴포넌트를 포함한다.

쿠버네티스의 컴포넌트는 다음과 같다.

- 마스터용 컴포넌트 : 클러스터를 관리하는데 필수
- 노드용 컴포넌트
- 애드온용 컴포넌트 : 필수는 아니지만 추가로 사용 가능

<br>

## **4.2.1 마스터용 컴포넌트**

마스터용 컴포넌트들은 실제 클러스터 전체를 관리함

![Untitled](/assets/img/231107-4.png){: width="60%"}

### **etcd**

- 쿠버네티스 클러스터 구성을 유지하는 분산 key-value 저장소이다.
- 쿠버네티스에서는 필요한 모든 데이터를 저장하는 데이터베이스 역할을 한다.
- etcd는 서버 하나 당 프로세스 1개만 사용할 수 있다.

### **kube-apiserver**

- 쿠버네티스 클러스터의 API를 사용할 수 있도록 하는 컴포넌트이다.
- 클러스터로 온 요청이 유효한지 검증한다.
- 수평적으로 확장할 수 있어서, 서버 여러 대에 여러 개의 kube-apiserver를 실행해 사용할 수 있다.

### **kube-scheduler**

- 현재 클러스터 안에서 자원 할당이 가능한 노드 중 알맞은 노드를 선택해서, 새롭게 만든 파드를 실행한다.
- 조건에 맞는 노드를 찾는다.

### **kube-controller-manager**

- 쿠버네티스는 파드들을 관리하는 컨트롤러가 있다.
- 컨트롤러 각각은 논리적으로 개별 프로세스인데, 복잡도를 줄이려고 모든 컨트롤러를 바이너리 파일 하나로 컴파일해 단일 프로세스로 실행된다.
- kube-controller-manager는 컨트롤러 각각을 실행하는 컴포넌트이다.

### **cloud-controller-manager**

- 쿠버네티스의 컨트롤러들을 클라우드 서비스와 연결해 관리하는 컴포넌트이다.
- 관련 컴포넌트의 소스 코드는 각 클라우드 서비스에서 직접 관리한다.
- 네 가지 컨트롤러 컴포넌트를 관리한다. (node, route, service, volume)

<br>

## **4.2.2 노드용 컴포넌트**

노드용 컴포넌트는 쿠버네티스 실행 환경을 관리한다.

![Untitled](/assets/img/231107-6.png){: width="60%"}

### **kubelet**

- 클러스터 안 모든 노드에서 실행되는 에이전트이다.
- 파드 컨테이너들의 실행을 직접 관리한다.
- 파드 스펙이라는 조건이 담긴 설정을 전달 받아서 컨테이너를 실행하고, 컨테이너가 정상적으로 실행되는지 헬스 체크를 한다.
- 노드 안에 있는 컨테이너라도 쿠버네티스가 만들지 않은 컨테이너는 관리하지 않는다.

### **kube-proxy**

- 클러스터 안에 별도의 가상 네트워크의 동작을 관리한다.

### **컨테이너 런타임**

- 실제로 컨테이너를 실행, 가장 유명한 런타임은 docker이다.

<br>

## **4.2.3 애드온**

애드온은 클러스터 안에서 필요한 기능을 실행하는 파드이다. 네임스페이스는 kube-system, 애드온으로 사용하는 파드들은 디플로이먼트, 리플리케이션 컨트롤러 등으로 관리한다.

### **네트워킹 애드온**

- AWS, Azure, GCC 같은 클라우드 서비스에서 제공하는 쿠버네티스를 사용하면 별도의 네트워킹 애드온을 제공한다.
- 하지만 쿠버네티스를 직접 서버에 구성한다면 네트워킹 관련 애드온을 설치해서 사용해야 한다.

### **DNS 애드온**

- 클러스터 안에서 동작하는 DNS 서버, 쿠버네티스 서비스에 DNS 레코드를 제공한다.

### **대시보드 애드온**

- 쿠버네티스는 kubectl이라는 CLI를 많이 사용하는데, 웹 UI로 쿠버네티스를 사용할 필요도 있다.
- 이때 클러스터 현황이나 파드 상태를 한눈에 쉽게 파악하는 기능이 있다.

### **컨테이너 자원 모니터링**

- 클러스터 안에서 실행 중인 컨테이너의 상태를 모니터링하는 애드온이다.

### **클러스터 로깅**

- 클러스터 안 개별 컨테이너의 로그, 쿠버네티스 구성 요소의 로그들을 중앙화한 로그 수집 시스템에 모아서 보는 애드온이다.
- 클러스터 안 각 노드에 로그를 수집하는 파드를 실행해서 로그 중앙 저장 파드로 로그를 수집한다.

<br>

# **4.3 오브젝트와 컨트롤러**

쿠버네티스

- 오브젝트
  - 파드, 서비스, 볼륨, 네임스페이스
- 오브젝트를 관리하는 컨트롤러
  - 레플리카세트, 디플로이먼트, 스테이트풀세트, 데몬세트, 잡

사용자 : 템플릿 등으로 쿠버네티스에 자원의 desired state를 정의한다.

컨트롤러 : desired state와 현재 상태가 일치하도록 오브젝트를 생성하거나 삭제한다.

## **4.3.1 네임스페이스**

쿠버네티스 클러스터 하나를 여러 개 논리적인 단위로 나눠서 사용하는 것이다. 네임스페이스 덕분에 쿠버네티스 클러스터 하나를 여러 개 팀이나 사용자가 함께 공유할 수 있다.

## **실습**

현재 생성되어 있는 네임스페이스들을 확인하자

```shell
$ kubectl get namespaces
NAME              STATUS   AGE
default           Active   26d
kube-node-lease   Active   26d
kube-public       Active   26d
kube-system       Active   26d
```

<br>

default 이외의 네임스페이스를 이용하자

- `kubectl`로 네임스페이스를 지정해서 사용할 때는 네임스페이스를 명시해야 하는데, 일일히 명시하기 번거롭다.
- 기본 네임스페이스를 변경해서 사용한다.
- 아래 예시의 경우, `NAMESPACE`가 비어 있으므로, `default`이다.

```shell
$ kubectl config current-context
docker-desktop

$ kubectl config get-contexts docker-desktop
CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
*         docker-desktop   docker-desktop   docker-desktop
```

<br>

`kube-system`으로 기본 네임스페이스를 변경하자

- `kube-system`의 네임스페이스에는 쿠버네티스 관리용 파드나 설정이 있다.

```shell
$ kubectl config set-context docker-desktop --namespace=kube-system
Context "docker-desktop" modified.

$ kubectl config get-contexts
CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
*         docker-desktop   docker-desktop   docker-desktop   kube-system
          minikube         minikube         minikube         default
```

<br>

변경 방법을 알았으니 기본 네임스페이스를 다시 default로 바꾸자

```shell
$ kubectl config set-context $(kubectl config current-context) --namespace=""
Context "docker-desktop" modified.

$ kubectl config get-contexts $(kubectl config current-context)
CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
*         docker-desktop   docker-desktop   docker-desktop
```

<br>

## **4.3.2 템플릿**

- 쿠버네티스 클러스터의 오브젝트, 컨트롤러가 어떤 상태여야 하는지를 적용할 때는 YAML 형식의 템플릿을 사용한다.
- 템플릿의 내용을 표현하는 YAML은 JSON과 비교했을 때 간결, 주석으로 인해 가독성이 좋다.

- 아래는 템플릿의 기본 형식이고, 각 항목은 필드이다.
- \*\*쿠버네티스의 특징인 선언적 API를 확인할 수 있다!

```yaml
apiVersion: v1 # 사용하려는 쿠버네티스 API 버전을 명시 kubectl api-versions
kind: Pod # 어떤 종류의 오브젝트 혹은 컨트롤러의 작업인지 명시 (Pod, Deployment, Ingress)
metadata: # 메타데이터를 설정, 해당 오브젝트의 이름이나 레이블
spec: # 파드가 어떤 컨테이너를 갖고 실행하며, 실행할 때 어떻게 동작할지 명시, status
```

<br>

### **참고**

파드 템플릿에서 사용하는 하위 필드가 무엇이 있는지 확인할 수 있다. 각 필드의 데이터 타입도 확인 가능하다.

```shell
$ kubectl explain pods
KIND:       Pod
VERSION:    v1

DESCRIPTION:
    Pod is a collection of containers that can run on a host. This resource is
    created by clients and scheduled onto hosts.

FIELDS:
  apiVersion    <string>
    APIVersion defines the versioned schema of this representation of an object.
    Servers should convert recognized schemas to the latest internal value, and
    may reject unrecognized values. More info:
    https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#resources

  kind  <string>
    Kind is a string value representing the REST resource this object
    represents. Servers may infer this from the endpoint the client submits
    requests to. Cannot be updated. In CamelCase. More info:
    https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds

  metadata      <ObjectMeta>
    Standard object's metadata. More info:
    https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata

  spec  <PodSpec>
    Specification of the desired behavior of the pod. More info:
    https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status

  status        <PodStatus>
    Most recently observed status of the pod. This data may not be up to date.
    Populated by the system. Read-only. More info:
    https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
```

<br>

### **이미지 출처**

- [https://medium.com/cwan-engineering/fully-equipped-kubernetes-cluster-ee8a9f34601d](https://medium.com/cwan-engineering/fully-equipped-kubernetes-cluster-ee8a9f34601d)
- [https://kubernetes.io/docs/concepts/architecture/](https://kubernetes.io/docs/concepts/architecture/)

<br>
<br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
