---
title: "[Kubernetes] 데이터 저장"
author:
date: 2023-11-07 11:30:00 +0900
categories: [Infra & DevOps, Infra]
image: /assets/img/231127-1.png
tags: [Kubernetes]
---

**[쿠버네티스 입문:90가지 예제로 배우는 컨테이너 관리 자동화 표준]** 책의 14장 데이터 저장 단원을 공부하고 실습한 내용을 담은 글입니다. Window10 운영체제에서 실습을 진행했습니다.

## **데이터 저장 단원 공부의 필요성**

Docker 컨테이너를 띄워서 작업을 하는데, 컨테이너 안에 저장한 데이터는 컨테이너가 삭제되면 모두 삭제된다. **컨테이너가 삭제되어도 데이터를 보존하도록, 컨테이너 외부에 데이터를 저장하는 방법을 알아본다.** 이를 위해 볼륨, 퍼시스턴트 볼륨을 사용하는 방법을 알아보고, 관련한 다양한 설정을 알아보자!

# **14.1 볼륨**

일반적으로 얘기하는 컨테이너는 기본적으로 **상태가 없는** 앱 컨테이너이다.

상태가 없다는 것의 의미는, 아래와 같은 문제 상황에서 컨테이너를 새로 실행했을 때 다른 노드로 자유롭게 옮길 수 있다는 뜻이다!

- 컨테이너 자체에 문제 발생
- 노드에 장애 발생

그러나 컨테이너가 실행되지 않거나 삭제된다면??? 현재까지 저장한 데이터가 사라진다.

컨테이너에 문제가 생겨도 데이터를 보존해야 하는 경우, 데이터를 파일로 저장하는 **Jenkins CI CD 파이프라인**을 이용해 데이터를 보존할 수 있다. 데이터베이스도 컨테이너를 종료하거나 재시작해도 데이터가 사라지면 안된다. (만약 데이터베이스의 데이터가 사라진다면 대참사가 발생할 것이다...)

**❗ 볼륨을 사용하면 컨테이너를 재시작해도 데이터를 유지한다 ❗**

**❗ 퍼시스턴트 볼륨을 사용하면 데이터를 저장했던 노드가 아닌 다른 노드에서 컨테이너를 재시작하더라도 데이터를 저장한 볼륨을 그대로 사용할 수 있다 ❗**

볼륨, 퍼시스턴트 볼륨을 사용하면 단순히 서버 하나에서 데이터를 저장해 사용하는 것보다 안정적으로 서비스를 운영할 수 있다.

## **쿠버네티스의 볼륨 플러그인**

1. aws, azure, gce
   - 클라우드 서비스에서 제공하는 볼륨 서비스
2. glusterfs, cephfs
   - 오픈소스로 공개된 스토리지 서비스
3. 컨피그맵, 시크릿
   - 쿠버네티스 내부 오브젝트
4. emptyDir, hostPath, local
   - 컨테이너가 실행된 노드의 디스크를 볼륨으로 사용하는 옵션
5. nfs 볼륨 플러그인
   - 하나의 컨테이너에 볼륨을 붙여서 NFS 서버로 설정하고, 다른 컨테이너에서 NFS 서버 컨테이너를 가져다가 사용하도록 설정 가능
   - 볼륨에서 직접적으로 멀티 읽기/쓰기를 지원하지 않더라도 그와 비슷한 효과를 내는 것도 가능

## **볼륨 관련 필드**

`.spec.container.volumeMounts.mountPropagation`

- 파드 하나 안에 있는 컨테이너들끼리 or 같은 노드에 실행된 파드들끼리 볼륨을 공유해서 사용할지 설정

`None`

- 이 필드 값으로 볼륨을 마운트 했으면, 호스트에서 볼륨에 해당하는 디렉터리 하위에 마운트한 다른 마운트들은 볼 수 없음
- 컨테이너가 만들어 놓은 마운트를 호스트에서 볼 수도 없음, 기본 필드 값

`HostToContainer`

- 이 필드 값으로 볼륨을 마운트했으면 호스트에서 해당 볼륨 하위에 마운트된 다른 디렉터리들도 해당 볼륨에서 볼 수 있도록 함

`Bidirectional`

- 이 필드 값으로 볼륨을 마운트 했으면 HostToContainer처럼 하위에 마운트된 디렉터리도 볼 수 있고, 호스트 안 다른 모든 컨테이너나 파드에서 같은 볼륨을 사용할 수 있음

교재의 볼륨 예제는 로컬 서버에서 사용할 수 있는 볼륨 중에서 내부 호스트의 디스크를 사용하는 emptyDir, hostPath / nfs 볼륨을 이용해서 볼륨 하나를 여러 개 컨테이너에서 공유해서 사용하는 방법을 알려주고 있다. 하나씩 실습을 해보자.

## **14.1.1 emptyDir**

- 파드가 실행되는 호스트의 디스크를 임시로 컨테이너에 볼륨으로 할당해서 사용하는 방법이다.
- 파드가 사라지면 emptyDir에 할당해서 사용했던 볼륨의 데이터도 함께 사라진다.
- 주로 메모리와 디스크를 함께 이용하는 대용량 데이터 계산에 사용한다.
- 연산 중 컨테이너에 문제가 발생해서 재시작되더라도 파드는 살아 있으므로 emptyDir에 저장해둔 데이터를 계속 이용할 수 있다.

volume/volume-emptydir.yml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubernetes-emptydir-pod
spec:
  containers:
    - name: kubernetes-simple-pod
      image: arisu1000/simple-container-app:latest
      volumeMounts:
        - mountPath: /emptydir
          name: emptydir-vol
  volumes:
    - name: emptydir-vol
      emptyDir: {}
```

spec 부분을 확인해보자. 쿠버네티스에서 볼륨을 설정할 때의 기본 형식이다.

1. .spec.volumes[]

   - 하위 필드에 사용하려는 볼륨들을 먼저 선언
   - .name 필드 값을 emptydir-vol이라는 이름으로 설정
   - emptyDir을 사용하려고 .emptyDir 필드 값으로 {} 빈 값 설정함

2. .spec.containers[].volumeMounts[]
   - 선언한 볼륨 사용
   - volumeMounts 하위 필드에서 선언한 볼륨을 불러와서 사용할 수 있음
   - name 필드 값을 emptydir-vol로 설정해 볼륨을 사용할 수 있도록 함
   - mountPath 필드 값으로 컨테이너의 /emptydir 디렉터리를 설정해 볼륨을 마운트 함

## **14.1.2 hostPath**

| emptyDir                                                                                         | hostPath                                                                                                       |
| ------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------- |
| 임시 디렉터리를 마운트                                                                           | 호스트에 있는 실제 파일이나 디렉터리를 마운트                                                                  |
| 컨테이너를 재시작했을 때 데이터를 보존하는 역할                                                  | 파드를 재시작했을 때도 호스트에 데이터가 남음                                                                  |
| 모니터링 툴을 섬세하게 설정했는데도 재시작 했을 때, <br> 설정만큼은 남기고 싶을 때 임시로 마운트 | 스프링부트에서 로그 파일 뜯어 볼 때, <br> hostPath를 설정하지 않으면 쿠버네티스가 재시작 했을 때 로그가 없어짐 |
| 파드가 재시작 할 때 같은 상태를 유지하고 싶을 때                                                 | 실제 데이터 저장                                                                                               |

- 호스트의 중요 디렉터리를 컨테이너에 마운트해서 사용할 수 있다.
- `/var/lib/docker` 같은 도커 시스템용 디렉터리를 컨테이너에서 사용할 때 혹은 시스템용 디렉터리를 마운트해서 **시스템을 모니터링하는 용도로도** 사용할 수 있다.

/volume/volume-hostpath.yml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubernetes-hostpath-pod
spec:
  containers:
  - name: kubernetes-hostpath-pod
    image: arisu1000/simple-container-app:latest
    volumeMounts:
    - mountPath: /test-volume
      name: hostpath-vol
    ports:
	    - containerPort:8080
volumes:
- name: hostpath-vol
  hostPath:
    path: /tmp
    type: Directory
```

1. .spec.containers[].volumeMounts
   - .mountPath 필드는 볼륨을 컨테이너의 /test-volume이라는 디렉터리에 마운트하도록 값을 설정
   - .name 필드로는 hostpath-vol로 볼륨의 이름 설정
2. .spec.volumes[]
   - .name 필드는 볼륨의 이름인 hostpath-vol으로 설정
   - 경로를 뜻하는 .hostPath.path 필드 값으로는 호스트의 /tmp 디렉터리 설정
   - 설정한 경로의 타입이 디렉터리임을 알리기 위해 .type 필드 값으로 Directory 설정
   - .spec.volumes[].hostpath.type의 필드 값

## **실습 1**

현재 클러스터에 등록된 노드들의 목록을 출력한다. 만약 클러스터가 구성되어 있으면, 하나 이상의 노드가 나타난다.

```shell
$ kubectl get nodes
NAME             STATUS   ROLES           AGE   VERSION
docker-desktop   Ready    control-plane   54d   v1.27.2

$ kubectl get pods
NAME                                    READY   STATUS                 RESTARTS      AGE
...
yunjinserver                            1/1     Running                9 (56m ago)   26d
```

사용자 노드에서 `index.html`를 연다.

- 실습자의 환경에서 사용자 클러스터의 단일 노드에 연결되는 shell을 연다.
- 윈도우의 경우, 아래와 같은 방법으로 열어서 `index.html`을 생성할 수 있다.

```shell
$ kubectl exec -it yunjinserver -- /bin/sh

$ mkdir /mnt/data
$ sh -c "echo Hello from k8s storage" > /mnt/data/index.html
```

- `index.html` 파일이 존재하는지 테스트한다.

```shell
$ cat /mnt/data/index.html

Hello from k8s storage
```

## **실습 2 (교재)**

/volume/volume-hostpath.yml을 작성하고, 클러스터에 적용한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubernetes-hostpath-pod
spec:
  containers:
    - name: kubernetes-hostpath-pod
      image: arisu1000/simple-container-app:latest
      volumeMounts:
        - mountPath: /test-volume
          name: hostpath-vol
      ports:
        - containerPort: 8080
  volumes:
    - name: hostpath-vol
      hostPath:
        path: /tmp
        type: Directory
```

```shell
$ kubectl apply -f volume-hostpath.yml
pod/kubernetes-hostpath-pod created
```

- `/test-volume` 디렉터리에 `test.txt` 파일을 생성하고 `exit` 한다.

```shell
$ kubectl exec kubernetes-hostpath-pod -it sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.

~ # cd /test-volume
/test-volume # touch test.txt
/test-volume # ls
test.txt
/test-volume # exit
```

컨테이너를 종료하고 호스트의 `/tmp` 디렉터리를 확인한다.

- 컨테이너를 종료하고 호스트의 `/tmp` 디렉터리를 확인하면 `test.txt` 파일이 남아있는 것을 확인할 수 있다.
- `volume-hostpath.yml`에서 설정한 내용대로 컨테이너 안 `/test-volume` 디렉터리는 컨테이너 외부 호스트의 `/tmp` 디렉터리를 마운트 했고 데이터를 보존한다❗

```shell
$ kubectl delete pod kubernetes-hostpath-pod
pod "kubernetes-hostpath-pod" deleted

$ ls /tmp/
test.txt
```

## **14.1.3 nfs**

nfs - network file system

nfs 볼륨은 기존에 사용하는 NFS 서버를 이용해서 파드에 마운트 하는 것으로, NFS 클라이언트 역할을 한다. 여러 개 파드에서 볼륨 하나를 공유해 읽기/쓰기를 동시에 할 때도 사용한다.

파드 하나에 안정성이 높은 외부 스토리지를 볼륨으로 설정한 후, 해당 파드에 NFS 서버를 설정한다. 다른 파드는 해당 파드의 NFS 서버를 nfs 볼륨으로 마운트한다.

- 우리는 노트북으로 실습하지만, 회사에서 일반 서버 컴퓨터 하나에 모든 파일을 두는 것은 매우 위험하다. 아예 안정적인 nfs 컴을 하나 두고, 파드가 이 nfs 컴퓨터랑 연결을 하고 그 파드에 다른 파드들을 연결한다.
- nfs 에 연결된 파드는 리눅스 형태의 파일 시스템에 제공을 하고 다른 파드들은 nfs에 연결될 필요 없이 쓴다.
- 서버를 분산 시킬 때도 사용한다.

## **실습**

노드 하나에 NFS 서버를 설정한 후 공유해서 사용한다.

- 외부 볼륨이 아닌 hostPath 볼륨으로 NFS 서버를 만들고, 다른 파드에서 해당 파드의 볼륨을 마운트한다.

/volume/volume-nfsserver.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-server
  labels:
    app: nfs-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-server
  template:
    metadata:
      labels:
        app: nfs-server
    spec:
      containers:
        - name: nfs-server
          image: arisu1000/nfs-server:latest
          ports:
            - name: nfs
              containerPort: 2049
            - name: mountd
              containerPort: 20048
            - name: rpcbind
              containerPort: 111
          securityContext:
            privileged: true
          volumeMounts:
            - mountPath: /exports
              name: hostpath-vol
      volumes:
        - name: hostpath-vol
          hostPath:
            path: /tmp
            type: Directory
```

`mountd`

- NFS 서버에서 사용하는 프로세스이다.
- 요청이 왔을 때 지정한 디렉터리로 볼륨을 마운트 하는 `mountd` 데몬이 사용하는 포트를 지정한다.

`rpcbind`

- rpcbind도 NFS 서버에서 사용하는 프로세스이다.
- 시스템에서 Remote Procedure Call 서비스를 관리할 rpcbind 데몬이 사용하는 포트를 지정한다.

`securityContext`

- 컨테이너의 보안을 설정한다.
- 현재 상태는 모든 호스트 장치에 접근이 가능하다.

`volumeMounts`

- 볼륨을 마운트할 디렉터리 경로로 `/exports`를 지정한다.

`deployment`를 생성하고, 실행한 컨테이너의 IP를 확인한다.

```shell
$ kubectl apply -f volume-nfsserver.yaml
deployment.apps/nfs-server created

$ kubectl get pods -o wide -l app=nfs-server
NAME                          READY   STATUS    RESTARTS   AGE   IP           NODE             NOMINATED NODE   READINESS GATES
nfs-server-84fd4c5858-bvrtc   1/1     Running   0          19s   10.1.0.172   docker-desktop   <none>           <none>
```

NFS 서버에 접속할 클라이언트 앱 컨테이너를 설정한다.

/volume/volume-nfsapp.yml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubernetes-nfsapp-pod
  labels:
    app: nfs-client
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nfs-client
  template:
    metadata:
      labels:
        app: nfs-client
    spec:
      containers:
        - name: kubernetes-nfsapp-pod
          image: arisu1000/simple-container-app:latest
          volumeMounts:
            - mountPath: /test-nfs # nfs 볼륨을 마운트할 디렉터리 설정
              name: nfs-vol
          ports:
            - containerPort: 8080
      volumes:
        - name: nfs-vol
          nfs:
            server: 10.1.0.172 # nfs-server 파드의 IP
            path: "/exports"
```

파드 2개가 nfs 볼륨을 사용할 수 있는 상태로 실행한다.

```shell
$ kubectl apply -f volume-nfsapp.yaml
deployment.apps/kubernetes-nfsapp-pod created

$ kubectl get pods
NAME                                     READY   STATUS                 RESTARTS        AGE
kubernetes-nfsapp-pod-85689b5fbd-fb8fh   0/1     ContainerCreating      0               62s
kubernetes-nfsapp-pod-85689b5fbd-ndtx8   0/1     ContainerCreating      0               62s
```

이 파드들 중 하나에 접속해서 파일을 변경해보자.

- `index.html`은 `nfs-server`가 자동으로 생성한다.

```shell
$ kubectl exec it kubernetes-nfsapp-pod-6df5b4f6cf-m2xvp sh
~ # cd /test-nfs/
/test-nfs # ls
index.html
```

`index.html`을 vi 편집기로 열어서, 간단하게 수정하고, `exit` 명령어로 컨테이너에서 빠져나온다. `/tmp/index.html`을 확인해보면, 내용이 수정된 것을 알 수 있다!

```shell
/test-nfs # vi index.html
/test-nfs # exit

$ ca /tmp/index.html
Hello from NFS!
modify
```

### **참고**

- 볼륨을 구성해서 마운트할 때, 디플로이먼트에서 바로 볼륨을 생성하지 않고 퍼시스턴트 볼륨으로 마운트해 안정성을 높인다.
- 파드 접근도 NFS 서버용 파드가 아니라 서비스를 생성해서 연결한다.

# **14.2 퍼시스턴트 볼륨과 퍼시스턴트 볼륨 클레임**

쿠버네티스에서 볼륨을 사용하는 구조는 다음과 같다.

PV (persistent volume)

- 볼륨 자체, 클러스터 안에서 자원으로 다룬다.
- 파드와는 별개로 관리되고, 별도의 생명 주기가 있다.

PVC (persistent volume claim)

- 사용자가 PV에 하는 요청을 의미한다.
- 사용하고 싶은 용량, 읽기/쓰기 모드 설정 등을 정해서 요청한다.

쿠버네티스는 볼륨을 파드에 직접 할당하는 방식이 아니라 중간에 PVC를 두어 파드와 파드가 사용할 스토리지를 분리한다 → 파드 각각의 상황에 맞게 다양한 스토리지를 사용할 수 있게 한다!

- 아래 그림과 같이, pod가 pv를 직접 사용하는 것이 아니라 pvc를 거쳐서 사용하므로 파드는 어떤 스토리지를 사용하는지 신경 쓰지 않아도 된다.

![Untitled](/assets/img/231127-1.png){: width="100%"}

[https://medium.com/devops-mojo/kubernetes-storage-options-overview-persistent-volumes-pv-claims-pvc-and-storageclass-sc-k8s-storage-df71ca0fccc3](https://medium.com/devops-mojo/kubernetes-storage-options-overview-persistent-volumes-pv-claims-pvc-and-storageclass-sc-k8s-storage-df71ca0fccc3)

### 14.2.2 바인딩

프로비저닝으로 만든 PV를 PVC와 연결하는 단계

PVC에서 원하는 스토리지의 용량과 접근 방법을 명시해서 요청하면 거기에 맞는 PV가 할당됨

PVC에서 원하는 PV가 없다면 요청 실패 → 바로 요청을 끝내지 않고, PVC에서 원하는 PV가 생길 때까지 대기하다가 PVC가 바인딩 됨

PV와 PVC의 매핑은 1대1 관계, PVC 하나가 여러 개 PV에 바인딩될 수 없음

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/7ee22bbf-f04e-4fa4-9f6a-3c3277e80c15/1c9a5c78-6e37-4e5c-937f-6dc1509eb824/Untitled.png)

### 14.2.3 사용

PVC - 파드에 설정

파드 - PVC를 볼륨으로 인식해서 사용

할당된 PVC는 파드를 유지하는 동안 계속 사용하고, 시스템에서 임의로 삭제할 수 없음 → 사용 중인 스토리지 오브젝트 보호 (사용 중인 데이터 스토리지를 임의로 삭제하면 ㅈ됨)

- 파드가 사용 중인 PVC를 삭제하려고 하면 상태가 terminating, 해당 PVC를 사용 중인 파드가 남아 있을 때는 PVC도 삭제되지 않고 남아 있음

### 14.2.4 반환

사용이 끝난 PVC는 삭제, PVC를 사용하던 PV를 초기화 하는 과정을 거치는 것을 반환이라고 함, 초기화 정책의 3가지

1. Retain
   - PV를 그대로 보존
   - PVC가 삭제되면 사용 중이던 PV는 해제 상태라서, 아직 다른 PVC가 재사용할 수 없음
   - 단순히 사용 해제 상태! PV 안의 데이터는 그대로 남아 있음
   - 이 PV를 재사용 하려면 관리자가 다음 순서대로 직접 초기화 해야 함
   1. PV 삭제 - PV가 외부 스토리지와 연결 되었다면, PV는 삭제되더라도 외부 스토리지의 볼륨은 그대로 남아 있음
   2. 스토리지에 남은 데이터를 직접 정리함
   3. 남은 스토리지의 볼륨을 삭제하거나 재사용하려면 해당 볼륨을 이용하는 PV를 다시 만듦
2. Delete
   - PV를 삭제하고 연결된 외부 스토리지 쪽의 볼륨도 삭제
   - 프로비저닝에서 동적 볼륨 할당으로 생성된 PV들은 기본 반환 정책이 delete
3. Recycle
   - PV의 데이터들을 삭제하고 다시 새로운 PVC에서 PV를 사용할 수 있도록 함
   - 특별한 파드를 만들어 두고 데이터를 초기화하는 기능도 있음 → will be deprecated.. why?

## 14.3 퍼시스턴트 볼륨 템플릿 - PV

volume/pv-hostpath.yaml

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-hostpath
spec:
  capacity:
    **storage: 2Gi # 스토리지 용량 2GB**
  volumeMode: Filesystem # 볼륨을 파일 시스템 형식으로 설정해서 사용
  accessModes: # 읽기/쓰기 옵션
  - ReadWriteOnce
  storageClassName: manual
  persistentVolumeReclaimPolicy: Delete # 앞선 retain, recycle, delete 중 하나 선택
  hostPath:
    path: /tmp/k8s-pv # 스토리지를 연결할 Path
```

빨간색

볼륨의 읽기/쓰기 옵션 설정, 볼륨은 한번에 accessModes 필드를 하나만 설정할 수 있고 필드 값은 세 가지가 있음

- ReadWriteOnce - 노드 하나에만 볼륨을 읽기/쓰기 하도록 마운트할 수 있음
- ReadOnlyMany - 여러 개 노드에서 읽기 전용으로 마운트할 수 있음
- ReadWriteMany - 여러 개 노드에서 읽기/쓰기 가능하도록 마운트할 수 있음

노란색

특정 스토리지 클래스가 있는 PV는 해당 스토리지 클래스에 맞는 PVC와만 연결됨, PV에 필드 설정이 없으면 없는 PVC와만 연결됨

파란색

해당 PV의 볼륨 플러그인 명시, 하위의 .path 필드에는 마운트시킬 로컬 서버의 경로를 설정

1. 위의 코드를 적용해보자
   - status가 available이면 잘 적용된 것
   - 특정 PVC에 연결된 bound, PVC는 삭제되었고 PV는 아직 초기화 되지 않은 released, 자동 초기화를 실패한 failed

```powershell
[root@k8s-master jh]# kubectl apply -f pv-hostname.yaml 
persistentvolume/pv-hostpath created
 
[root@k8s-master jh]# kubectl get pv
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv-hostpath   2Gi        RWO            Delete           **Available**           manual                  16m
```

## 14.4 퍼시스턴트 볼륨 클레임 템플릿 - PVC

/volume/pvc-hostpath.yaml

```yaml
kind: PersistentVolumeClain
apiVersion: v1
metadata:
  name: pvc-hostpath
spec:
  accessModes:
  - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      **storage: ???**
  storageClassName: manual
```

storage

- 자원을 얼마나 사용할 것인지 요청 (request)

### 퀴즈

앞서 PV의 용량 설정을 2Gi로 하였고, 내가 현재 사용 가능한 스토리지 용량은 3Gi이다. 이때 PVC의 용량을 얼마로 해야 할까?

- 정답
  1GB
  만약 PV의 용량 이상을 설정하면, 사용할 수 있는 PV가 없으므로 PVC를 생성할 수 없는 pending 상태가 됨!
  ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/7ee22bbf-f04e-4fa4-9f6a-3c3277e80c15/acb2b677-fc4e-41c6-9e4d-5a6267a5d637/Untitled.png)

2G - 여진, 수민, 명근 / **1G - 동희** / **2G 이하 - 완돌아버님**

1. 앞선 pvc-hostpath.yaml을 클러스터에 적용함

```powershell
$ kubectl apply -f pvc-hostpath.yaml
persistentvolumeclaim "pvc-hostpath" created

$ kubectl get pvc
NAME           STATUS    VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-hostpath   **Bound     pv-hostpath**   2Gi        RWO            manual         3s

$ kubectl get pv
NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                  STORAGECLASS   REASON    AGE
**pv-hostpath**   2Gi        RWO            Delete           Bound     default/pvc-hostpath   manual                   11m
```

🔥

## 14.5 레이블로 PVC와 PV 연결하기

다시 정리하자면,

PV - 쿠버네티스 안에서 사용되는 자원

PVC - 해당 자원을 사용하겠다고 요청하는 것, 파드와 서비스를 연결하는 것처럼 **레이블**을 사용할 수 있음!

### PV에 레이블 추가

/volume/pv-hostpath-label.yaml

```powershell
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-hostpath
	labels:
		location: local
spec:
  capacity:
    **storage: 2Gi # 스토리지 용량 2GB**
  volumeMode: Filesystem
  accessModes: # 읽기/쓰기 옵션
  - ReadWriteOnce
  storageClassName: manual
  persistentVolumeReclaimPolicy: Delete
  hostPath:
    **path: /tmp/k8s-pv** 
```

### PVC에 레이블 추가

/volume/pvc-hostpath-label.yaml

```powershell
kind: PersistentVolumeClain
apiVersion: v1
metadata:
  name: pvc-hostpath
spec:
  accessModes:
  - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
     **** storage: 1Gi
  storageClassName: manual
	selector:
		matchLabels:
			location: local
```

- location: local 이 부분을 {key: stage, operator: In, values: [deployment]}로 원하는 레이블 조건을 설정할 수 있음

### 레이블을 붙인 각 코드를 클러스터에 적용

```powershell
$ kubectl apply -f pv-hostpath-label.yaml

$ kubectl apply -f pvc-hostpath-label.yaml
```

## 14.6 파드에서 PVC를 볼륨으로 사용하기

만든 PVC를 실제 파드에 붙여보자

/volume/deployment-pvc.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubernetes-simple-app
  labels:
    app: kubernetes-simple-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kubernetes-simple-app
  template:
    metadata:
      labels:
        app: kubernetes-simple-app
    spec:
      containers:
      - name: kubernetes-simple-app
        image: arisu1000/simple-container-app:latest
        ports:
        - containerPort: 8080
        imagePullPolicy: Always
        volumeMounts:
        - mountPath: "/tmp"
          name: myvolume
      volumes:
      - name: myvolume
        persistentVolumeClaim:
          claimName: pvc-hostpath
```

파란색

- 사용할 PVC를 volumes.persistentVolumeClaim.claimName에 넣어서 설정
- 디플로이먼트 파드에서 사용할 myvolume이라는 볼륨을 준비한 것임!

빨간색

- 준비한 볼륨을 실제 컨테이너에 연결하는 필드
- 마운트 경로를 컨테이너의 /tmp 디렉터리로 설정
- kubernetes-pvc-app 디플로이먼트는 /tmp 디렉터리에 app.log라고 접속 로그를 남김

```yaml
$ kubectl apply -f deployment-pvc
// 클러스터에 적용

$ kubectl get pods
// 파드 이름 확인

$ kubectl port-forward pds/파드 이름 8080:8080
```

1. 마지막 명령어로 파드에 접근할 수 있는 포트 번호를 설정함
2. 웹 브라우저에서 localhost:8080에 몇 번 접속해봄
3. 그리고 명령 실행을 종료함

### 퀴즈

PV가 생성된 곳?

여진, 수민, 동희, 명근 - tmp / **완돌아버지 - tmp/k8s-pv**

- 정답
  /tmp/k8s-pv
  로컬 서버의 {} 디렉터리가 컨테이너의 /tmp 디렉터리 하위에 마운트 되었을 것임!
  ```yaml
  $ cat /tmp/k8s-pv/app.log
  [GIN] 2023/11/27 - 07:32:17 | 200 |       777.2µs |       127.0.0.1 | GET      /
  [GIN] 2023/11/27 - 07:32:19 | 200 |       241.2µs |       127.0.0.1 | GET      /
  [GIN] 2023/11/27 - 07:32:19 | 200 |       138.9µs |       127.0.0.1 | GET      /
  ```
  kubernetes-simple-app 디플로이먼트의 접속 로그인 app.log가 컨테이너의 /tmp/app.log에 남아서, 해당 로그 내용은 로컬 서버의 /tmp/k8s-pv/app.log에서 확인할 수 있음

## 14.7 PVC 크기 늘리기

한 번 할당한 PVC의 크기를 늘리는 것도 가능함!

단, gcePersistentDisk, awsElasticBlockStore, Cinder, glusterfs, rbd, Azure File, Azure Disk, Portworx 등의 볼륨 타입을 사용해야 하고, 스토리지클래스에 allowVolumeExpansion 옵션이 true로 설정되어 있어야 함

1. 기존 .spec.resources.requests.storage 필드 값에 더 높은 용량을 설정한 후 클러스터에 적용함
2. 볼륨에서 사용 중인 파일 시스템이 XFS, EXT3, **EXT4**라면 파일 시스템이 있더라도 볼륨 크기를 늘릴 수 있음
3. 파일 시스템이 있는 볼륨 크기를 늘리는 작업은 해당 PVC를 사용하는 새로운 파드를 실행할 때만 진행됨
4. 기존에 특정 파드가 사용 중인 볼륨의 크기를 늘리려면, 파드를 재시작 해야 함 → 불편쓰… 그래서 1.11 버전에서는 사용 중인 볼륨 크기를 조절하는 기능이 도입됨!

## 14.8 노드별 볼륨 개수 제한

쿠버네티스에서는 노드 하나에 설정할 수 있는 볼륨 개수에 제한을 둠

스케줄러 KUBE_MAX_PD_VOLS 환경 변수를 이용해서 설정할 수 있음

아래와 같이 클라우드 서비스마다 제한 사항이 있음
