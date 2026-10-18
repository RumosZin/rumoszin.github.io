---
title: "[Kubernetes] 쿠버네티스로 컨테이너 시작하기"
author:
date: 2023-10-31 12:30:00 +0900
categories: [Infra & DevOps, Infra]
tags: [Kubernetes]
---

**[쿠버네티스 입문:90가지 예제로 배우는 컨테이너 관리 자동화 표준]** 책의 3장 쿠버네티스로 컨테이너 시작하기 단원을 공부하고 작성한 글입니다. 실습을 진행한 환경은 Window10 운영체제입니다.

# **3.1 kubectl**

쿠버네티스 클러스터를 kubectl로 관리하는 방법을 알아보자

## **3.1.2 기본 사용법**

### **kubectl의 기본 사용법**

간단한 에코 서버를 동작시키는 kubectl 명령 예를 통해 kubectl의 기본 사용법을 알아본다.

![Untitled](/assets/img/231031-1.png){: width="100%"}

- 일반적으로 **애플리케이션의 개별 기능은 각각 파드로 따로 구현하는 것**을 추천한다. 그래야 그 기능에 버그가 생겨도, 다른 파드에 영향 없이 해당 파드만 수정, 제거가 가능하다.

- 각 파드(기능) 간의 통신은 파드의 `private IP address`를 통해 이루어진다.

- 외부 사용자가 파드에 접근하기 위해서, `Service`라는 쿠버네티스 리소스를 통해 접근해야 한다.

<br>

`echoserver`라는 이름의 파드를 생성한다.

```shell
$ kubectl run echoserver --generator=run-pod/v1 --image="k8s.gcr.io/echoserver:1.10" --port=8080
error: unknown flag: --generator
See 'kubectl run --help' for usage.

$ kubectl run echoserver --image=k8s.gcr.io/echoserver:1.10 --port=8080
pod/echoserver created
```

파드에 접근할 때 필요한 서비스를 생성한다.

```shell
$ kubectl expose po echoserver --type=NodePort
service/echoserver exposed
```

`kubectl get pods`로 파드가 정상적으로 생성되었는지 확인한다.

```shell
$ kubectl get pods
NAME        READY      STATUS       RESTARTS        AGE
echoserver  1/1        Running      0               2m44s
```

- NAME - 파드의 이름

- READY

  - 0/1 현재 파드가 생성되었으나 사용할 준비가 되지 않았음
  - 1/1 파드가 생성되었고 사용할 준비가 끝났음

- STATUS - 파드의 현재 상태

  - Running - 파드 실행됨
  - Terminating - (새로운 파드 생성 중) 컨테이너 생성 중
  - ContainerCreating - (새로운 파드 생성 중) 컨테이너 생성 중

- RESTARTS - 해당 파드가 몇 번 재시작했는지 표시

- AGE - 파드를 생성한 후 얼마나 시간이 지났는지 표시

<br>

`kubectl get services`로 서비스가 정상적으로 생성되었는지 확인한다.

```shell
$ kubectl expose po echoserver --type=NodePort
service/echoserver exposed

$ kubectl get services
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
echoserver   NodePort    10.103.207.36   <none>        8080:31205/TCP   3s
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP          26d
```

서버에 접근할 수 있도록 로컬 컴퓨터로 port-forwarding을 해준다. 서버의 8080 포트를 로컬 컴퓨터의 8080으로 forwarding한다.

```shell
$ kubectl port-forward svc/echoserver 8080:8080
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
Handling connection for 8080
Handling connection for 8080
```

`localhost:8080`로 확인할 수 있다.

![Untitled](/assets/img/231031-2.png){: width="100%"}

### **서버를 실행한 후 테스트**

다른 shell을 실행한 후 `curl http://localhost:8080`을 입력한다. Minikube 또는 Kubernetes 클러스터 외부에서 클러스터 내의 서비스에 엑세스 하기 위한 명령어이다.

**로컬 머신에서 localhost 주소와 8080 포트를 통해 클러스터 내의 서비스에 HTTP 요청을 보내는데, 이것은 클러스터 내의 서비스로 요청을 전달하고, 해당 서비스가 에코 서버 Pod에 연결되어 응답을 반환하는 방식이다.**

만약 동작하지 않는다면, `minikube`로 `echoserver`를 서비스한 후, `minikube` 내에서 접근한다.

```shell
$ minikube ssh
docker@minikube:/$ curl http://192.168.@@.@@:31663
```

![Untitled](/assets/img/231031-3.png){: width="100%"}

![Untitled](/assets/img/231031-2.png){: width="100%"}

### **서버 실행 중 로그 수집**

`kubectl logs -f echoserver`으로 시간, HTTP 버전, 웹 브라우저와 운영체제 버전을 확인할 수 있다.

```shell
$ kubectl logs -f echoserver
Generating self-signed cert
Generating a 2048 bit RSA private key
......+++
...........................................................................................................+++
writing new private key to '/certs/privateKey.key'
-----
Starting nginx
127.0.0.1 - - [30/Oct/2023:09:20:29 +0000] "GET / HTTP/1.1" 200 1072 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/118.0.0.0 Safari/537.36 Edg/@@@.@.@@@@.@@"
127.0.0.1 - - [30/Oct/2023:09:20:29 +0000] "GET /favicon.ico HTTP/1.1" 200 1010 "http://localhost:8080/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/118.0.0.0 Safari/537.36 Edg/@@@.@.@@@@.@@"
10.244.0.1 - - [30/Oct/2023:09:21:04 +0000] "GET / HTTP/1.1" 200 1074 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/118.0.0.0 Safari/537.36 Edg/@@@.@.@@@@.@@"
10.244.0.1 - - [30/Oct/2023:09:21:04 +0000] "GET /favicon.ico HTTP/1.1" 200 1013 "http://127.0.0.1:59950/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/118.0.0.0 Safari/537.36 Edg/@@@.@.@@@@.@@"
10.244.0.1 - - [30/Oct/2023:09:40:37 +0000] "GET / HTTP/1.1" 200 424 "-" "curl/7.@@.0"
```

<br>

### **실습한 파드와 서비스 삭제**

shell에서 서버 로그 수집, 서버 실행을 중지한다.

- 만약 앞서 `minikube service echoserver`를 통해 실행했다면, `ctrl + ^C`로 중지한 후 명령어를 통해 파드와 서비스를 삭제한다.

```shell
$ kubectl delete pod echoserver
pod "echoserver" deleted
$ kubectl delete service echoserver
service "echoserver" deleted
```

파드와 서비스 정보를 확인하는 명령으로 파드와 서비스가 정상적으로 삭제되었는지 확인한다.

- Kubernetes 클러스터의 `default namespace`에서 리소스(예: Pod, 서비스 등)를 찾을 수 없다는 의미이므로, 정상적으로 삭제된 것이다!

```shell
$ kubectl get pods
No resources found in default namespace.
$ kubectl get services
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   6h48m
```

<br>

## **3.1.5 kubeconfig 환경 변수**

`kubectl`은 기본적으로 `$HOME/.kube/config` 파일에서 클러스터, 인증, 컨텍스트 정보를 읽어 들인다. 이러한 클러스터 구성 정보를 `kubeconfig`라고 한다. docker desktop으로 쿠버네티스를 사용한다면 자동으로 `kubeconfig`가 설정된다.

클러스터에서 사용할 수 있는 자원들은 `kubectl api-resources` 명령으로 확인할 수 있다.

```shell
$ kubectl api-resources
NAME                              SHORTNAMES   APIVERSION                             NAMESPACED   KIND
bindings                                       v1                                     true         Binding
componentstatuses                 cs           v1                                     false        ComponentStatus
configmaps                        cm           v1                                     true         ConfigMap
endpoints                         ep           v1                                     true         Endpoints
events                            ev           v1                                     true         Event
limitranges                       limits       v1                                     true         LimitRange
namespaces                        ns           v1                                     false        Namespace
nodes                             no           v1                                     false        Node
# 중략
csistoragecapacities                           storage.k8s.io/v1                      true         CSIStorageCapacity
storageclasses                    sc           storage.k8s.io/v1                      false        StorageClass
volumeattachments                              storage.k8s.io/v1                      false        VolumeAttachment
```

`—kubeconfig` 옵션으로 다른 설정 파일을 지정할 수 있다.

```shell
$ kubectl config use-context docker-desktop
Switched to context "docker-desktop".
```

<br>

## **3.1.7 다양한 사용 예**

`kubectl`을 단순히 명령 실행에 사용하는 것뿐 아니라, shell script의 일부분으로 사용하여 클러스터의 많은 동작을 자동화할 수 있다.

## **쿠버네티스에서의 JSONPath 사용**

### **JSONPath란?**

JSONPath에 대한 글이 아니므로, 이 단원에서 필요한 JSONPath에 대한 내용은 [다른 포스트](https://rumoszin.github.io/posts/jsonpath/)에 작성하였다. 쿠버네티스에서는 클러스터 `config` 내용이나 `node`, `resource` 정보를 조회할 때 JSONPath 템플릿을 사용할 수 있다.

장점 1) `kubectl get` 기본 명령으로는 접근할 수 없는 정보를 알아볼 수 있다.

장점 2) 필요에 따라 출력 결과물을 원하는 조건에 따라 정렬시킬 수 있다.

⇒ 수십 개의 노드를 동시에 관리해야 하는 상용 환경에서도 꼭 필요한 정보만 걸러내어 확인할 수 있다!

### **실습**

JSONPath를 이용해서 원하는 정보를 확인하자! echoserver, [자기이름]server 2개의 파드를 생성한다. 앞서 echoserver를 삭제했지만, 연습 삼아서 다시 만들어본다.

```shell
$ kubectl run echoserver --image=k8s.gcr.io/echoserver:1.10 --port=8080
pod/echoserver created

$ kubectl expose po echoserver --type=NodePort
service/echoserver exposed

$ kubectl get pods
NAME         READY   STATUS    RESTARTS   AGE
echoserver   1/1     Running   0          16s

$ kubectl run yunjinserver --image=k8s.gcr.io/echoserver:1.10 --port=8080
pod/yunjinserver created

$ kubectl expose po yunjinserver --type=NodePort
service/yunjinserver exposed

# pod, service 정상 생성 확인
$ kubectl get pods
NAME           READY   STATUS    RESTARTS   AGE
echoserver     1/1     Running   0          63s
yunjinserver   1/1     Running   0          17s
...

$ kubectl get services
NAME           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
echoserver     NodePort    10.106.213.33   <none>        8080:32760/TCP   76s
kubernetes     ClusterIP   10.96.0.1       <none>        443/TCP          26d
yunjinserver   NodePort    10.107.50.145   <none>        8080:32614/TCP   26s
...
```

<br>

# **3.2 디플로이먼트를 이용해 컨테이너 실행하기**

쿠버네티스를 이용해서 컨테이너를 실행하는 방법은 다음과 같다.

방법 1) `kubectl run` 명령으로 직접 컨테이너를 실행한다.

방법 2) 어떻게 실행할 지 세부 내용을 담은 YAML 형식의 템플릿으로 컨테이너를 실행한다. → 버전 관리 시스템과 연동해서 자원 정의 변동 사항을 추적하기 쉽다.

![Untitled](/assets/img/231031-1.png){: width="100%"}

![Untitled](/assets/img/231031-5.png){: width="100%"}

`service`란 `pod`의 논리적인 집합이며, 어떻게 접근할 지에 대한 정책을 내놓은 것이다. `service`는 `Label Selector`를 통해 어떤 `pod`를 포함할 지 정의할 수 있다. 각 `pod`의 외부 IP로는 직접적인 접근이 불가능하지만, `service`는 가능하게 한다.

## **3.2.1 kubectl run으로 컨테이너 실행하기**

쿠버네티스는 파드를 실행하는 여러 가지 컨트롤러를 제공하는데, `kubectl run`으로 파드를 실행시킬 때 기본 컨트롤러는 디플로이먼트이다.

디플로이먼트를 이용해 nginx 컨테이너를 실행하자.

(교재에 나온 명령어) kubectl run 디플로이먼트이름 —image 컨테이너이미지이름 —port=80을 하면 deployment.apps가 나오지 않고 pod가 생성된다.

```shell
$ kubectl run nginx-app --image nginx --port=80
pod/nginx-app created
```

디플로이먼트를 이용해 nginx를 실행하고, 디플로이먼트를 노출시키고, nginx를 실행하는 파드를 시작한다.

```shell
$ docker stop nginx-app
nginx-app

$ docker rm nginx-app
nginx-app

$ docker run -d --restart=always -e DOMAIN=cluster --name nginx-app -p 80:80 nginx
e070188344c507f20942944ab1178ac60fabae6c559ba26bf1e25a47e04d5987
```

```shell
$ docker ps
CONTAINER ID   IMAGE      COMMAND                   CREATED          STATUS          PORTS                  NAMES
677b69a6ec05   nginx     "/docker-entrypoint.…"   4 seconds ago    Up 4 seconds    0.0.0.0:80->80/tcp     nginx-app
```

```shell
$ kubectl create deployment --image=nginx nginx-app
deployment.apps/nginx-app created
```

사용자가 쿠버네티스 클러스터에 컨테이너를 실행하라고 명령하면, 지정된 컨테이너 이미지를 가져와 쿠버네티스 클러스터 안에서 실행한다.

<br>

nginx 컨테이너가 제대로 실행되는지 확인한다.

```shell
$ kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
echoserver                   1/1     Running   0          38m
nginx-app-5c64488cdf-lhtg7   1/1     Running   0          68s <<
yunjinserver                 1/1     Running   0          37m

$ kubectl get deployments
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
nginx-app   1/1     1            1           82s <<
```

디플로이먼트의 상태를 확인하는 항목 각각의 의미는 다음과 같다.

- NAME - 클러스터에 배포한 디플로이먼트 이름

- READY - 사용자가 최종 배포한 파드 개수와 디플로이먼트를 이용해 현재 클러스터에 실제로 동작시킨 파드 개수를 X/X 형태로 표시

- UP-TO-DATE - 디플로이먼트 설정에 정의한 대로 동작 중인 신규 파드 개수

- AVAILABLE - 서비스 가능한 파드 개수

- AGE - 디플로이먼트를 생성한 후 지난 시간

<br>

디플로이먼트를 이용해서 실행 중인 파드 개수를 늘린다. READY의 출력 결과가 2/2 → 사용자가 최종 배포한 파드 개수와 실제로 동작하는 파드 개수가 각각 2개이다.

```shell
$ kubectl scale deploy nginx-app --replicas=2
deployment.apps/nginx-app scaled

$ kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
echoserver                   1/1     Running   0          51m
nginx-app                    1/1     Running   0          31m
nginx-app-5c64488cdf-lh67r   1/1     Running   0          26s <<
nginx-app-5c64488cdf-lhtg7   1/1     Running   0          14m <<
yunjinserver                 1/1     Running   0          50m

$ kubectl get deployments
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
nginx-app   2/2     2            2           14m
```

<br>

원활한 실습을 위해 파드와 디플로이먼트를 삭제한다.

```shell
$ kubectl delete deployment nginx-app
deployment.apps "nginx-app" deleted

$ kubectl get deployments
No resources found in default namespace.

$ kubectl get pods
NAME           READY   STATUS    RESTARTS   AGE
echoserver     1/1     Running   0          56m
yunjinserver   1/1     Running   0          55m
```

<br>

디플로이먼트를 삭제하면, 파드도 함께 삭제된다. 아래 이미지와 같은 구조로 인해, 디플로이먼트를 삭제하면 파드도 함께 삭제된다. 디플로이먼트는 파드를 업데이트하기 위한 선언적 명세이다.

![Untitled](/assets/img/231031-6.png){: width="100%"}

## **3.2.2 템플릿으로 컨테이너 실행하기**

**✅ 디플로이먼트 설정이 담긴 템플릿(YAML 파일)로 컨테이너를 실행해보는 실습을 준비했다. 이 포스트의 내용, 교재의 3장 내용이 충분히 이해됐다면 실습은 어렵지 않을 것이다! (사실 나중에 내가 까먹을까봐, 이 실습을 통해 기억해내려고 만든 것이기도 하다!)**

[Deployment 설정이 담긴 템플릿으로 컨테이너 실행하기 실습](https://github.com/RumosZin/k8s-start-with-container-practice)

<br>

## **3.3 클러스터 외부에서 클러스터 안 앱에 접근하기**

실행 중인 nginx 컨테이너에 접속하는 방법을 살펴보자. 쿠버네티스 내부에서 사용하는 네트워크는, 외부와 격리되어 있다! 쿠버네티스 내부에서 실행한 컨테이너를 외부에서 접근하려면, 쿠버네티스 서비스를 사용해야 한다.

서비스 하나에 모든 노드의 지정된 포트를 할당하는 `NodePort`를 설정한다.

- YAML 파일을 저장한 경로에서 수행한다. nginx-app-deployment-perfect라는 서비스가 생성되어 있다.

- 쿠버네티스 내부의 `80`번 포트가 `30804`라는 외부 포트와 연결되어 있고, Endpoint를 보면 서비스에 컨테이너 2개가 연결되어 있다.

```shell
$ kubectl expose deployment nginx-app-deployment-perfect  --type=NodePort
service/nginx-app-deployment-perfect exposed

$ kubectl get service
NAME                           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
echoserver                     NodePort    10.109.240.33    <none>        8080:31163/TCP   56m
kubernetes                     ClusterIP   10.96.0.1        <none>        443/TCP          26d
nginx-app-deployment-perfect   NodePort    10.96.155.121    <none>        **80:30804/TCP**     17s
yunjinserver                   NodePort    10.102.144.249   <none>        8080:32174/TCP   55m

$ kubectl describe service nginx-app-deployment-perfect
Name:                     nginx-app-deployment-perfect
Namespace:                default
Labels:                   app=nginx-app-deployment-perfect
Annotations:              <none>
Selector:                 app=nginx-app-hello
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.96.155.121
IPs:                      10.96.155.121
LoadBalancer Ingress:     localhost
Port:                     <unset>  80/TCP
TargetPort:               80/TCP
NodePort:                 <unset>  30804/TCP
**Endpoints:                10.1.0.59:80,10.1.0.60:80**
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

<br>

`localhost:30804`로 접속하면 nginx의 기본 페이지를 확인할 수 있고, 이 화면을 만나면 클러스터 안의 앱에 접근한 것이다.

![Untitled](/assets/img/231031-7.png){: width="100%"}

<br>

사용하지 않을 컨테이너를 삭제한다.

```shell
$ kubectl delete deployment nginx-app
deployment.apps "nginx-app" deleted

$ kubectl get pods
NAME           READY   STATUS    RESTARTS   AGE
echoserver     1/1     Running   0          81m
nginx-app      1/1     Running   0          62m
yunjinserver   1/1     Running   0          81m

$ kubectl get deployments
No resources found in default namespace.
```

<br>

### **이미지 출처**

- [https://kubernetes.io/docs/tutorials/kubernetes-basics/explore/explore-intro/](https://kubernetes.io/docs/tutorials/kubernetes-basics/explore/explore-intro/)
- [https://kubernetes.io/docs/tutorials/kubernetes-basics/expose/expose-intro/](https://kubernetes.io/docs/tutorials/kubernetes-basics/expose/expose-intro/)
- [https://yankeexe.medium.com/how-rolling-and-rollback-deployments-work-in-kubernetes-8db4c4dce599](https://yankeexe.medium.com/how-rolling-and-rollback-deployments-work-in-kubernetes-8db4c4dce599)

<br>
<br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
