---
layout: post
title: "쿠버네티스 안전하게 사용하기 - 배경"
author: Jang Kyung Ho
date: 2021-04-02 18:15:00
categories: Kubernetes
img: 2.png
---

<div class="img-in-post">
<img src="/images/2/1.png" />
</div>

위 기사에서도 알 수 있듯, 클라우드에 대한 관심은 점점 증가하고 있으며 올해부터 국내 금융기관 및 공공기관의 서버들을 기존에 기업 내부에서 직접 구축하여 사용하던 On-Premise 서버에서 벗어나 서서히 클라우드 서버로 전환한다는 소식이 전해지고 있습니다.

클라우드가 상용화되어가며 점점 중요시되는 부분 중 하나가 클라우드 보안인데, 이와 관련된 연구를 진행하며 얻은 정보들을 회사 블로그를 통하여 공유하고자 이렇게 글을 쓰게 되었습니다. 이 글은 차후에 다룰 내용을 이해하는 데 필요한 배경지식과 용어 등을 정리하고 설명하는 글입니다.

<br />

## Background

<div class="img-in-post">
<img src="/images/2/2.png" />
</div>

가장 먼저 글의 제목에도 들어가 있는 쿠버네티스에 대하여 설명하겠습니다. 
쿠버네티스 공식 도큐먼트에 적혀있는 내용을 인용하면 아래와 같습니다.

>쿠버네티스는 컨테이너화된 워크로드와 서비스를 관리하기 위한 이식성이 있고, 확장 가능한 오픈소스 플랫폼입니다.

즉 도커와 같은 컨테이너들을 관리하기 쉽게 만들어놓은 플랫폼입니다. 

여기서 컨테이너란 무엇일까요?

<br />

<div class="img-in-post">
<img src="/images/2/3.png" />
</div>

과거 하나의 물리 서버에서 여러 서비스를 실행시켰던 시절에는 서비스마다 필요한 리소스가 상이하기 때문에 하나의 서비스에서 리소스를 독점하면 다른 서비스의 성능이 저하되는 문제를 겪을 수 있었습니다. 이를 방지하기 위해서는 여러 대의 물리 서버가 필요하였으나, 하나의 물리 서버 내에서 해결할 수 있는 방안으로 처음 나온 것이 가상 머신(Virtual Machine)이라고 불리는 전가상화입니다. 

전가상화란 하나의 물리 서버에서 여러 가상 시스템을 구동할 수 있게 하는 것으로, 예를 들어 Windows를 사용하고 있는 PC에서 MacOS, Linux 등을 동시에 사용할 수 있습니다. 장점으로 물리 서버의 리소스를 각 분리된 가상 머신 별로 효율적으로 사용 가능하며 가상 머신끼리의 접근 또한 자유롭지 않기 때문에 보안성 또한 뛰어납니다.

다만, 가상머신별로 OS를 가상화해야 한다는 점에서 많은 리소스를 필요로 한다는 문제점을 가지고 있습니다. 이 때문에 자원을 더 효율적으로 사용하면서 전가상화의 장점은 그대로 살릴 수 있도록 개발된 것이 바로 반가상화 개념인 컨테이너입니다. 전가상화와 달리 반가상화인 컨테이너는 호스트 PC의 커널 영역을 컨테이너끼리 공유하고 있으며 각 컨테이너별로 알맞게 파일 시스템을 Overlay하여 컨테이너 이미지를 만들고 이를 실행합니다. 이러한 반가상화 구조는 OS가 가상화되어 분리되는 VM과 다르게 커널 및 일부 환경을 공유하기 때문에 리소스를 상대적으로 적게 소모하는 장점을 갖추고 있습니다.

컨테이너에 대해서 알아보았고, 쿠버네티스는 이 컨테이너들을 효율적으로 관리하는 오픈소스 플랫폼입니다. 자세한 내용은 아래에서 다루도록 하겠습니다.

<br />

### Cluster

<div class="img-in-post">
<img src="/images/2/4.png" />
</div>

쿠버네티스에서 클러스터(Cluster)란  **"컨테이너화된 어플리케이션을 실행시키는 컴퓨팅 머신의 집합"** 정도로 요약할 수 있습니다. 
클러스터에는 이 컴퓨팅 머신과 컨테이너화된 어플리케이션을 관리하는 컨트롤 플레인(Control Plane)이라는 개념이 존재하는데, 이는 아래와 같습니다. 

#### Control Plane

컨트롤 플레인은 노드 머신들을 관리하며, 스케줄링 및 클러스터에 대한 이벤트를 추적 및 관리하는 역할을 담당하고 있습니다.

- **kube-apiserver**

    API 서버는 쿠버네티스 컨트롤 플레인의 프론트엔드입니다.
    Kubernetes API 서버는 파드, 서비스, 레플리카 컨트롤러 등을 포함하는 API 개체에 대한 데이터를 검증하고 구성합니다. 
    kubectl에서 오브젝트의 생성, 삭제 등 모든 커맨드를 실행할 때 이 kube-apiserver와 통신하여 해당 커맨드와 연관된 API를 호출합니다.

- **etcd**

    etcd는 쿠버네티스의 기본 저장소로써 구성 데이터, 메타 데이터 등 모든 데이터를 저장하는 분산 데이터 저장소입니다. etcd를 사용하면 Kubernetes 클러스터의 모든 노드가 데이터를 읽고 쓸 수 있으며 여기에는 SSH Key, API Key 등 주요 정보들을 포함한 모든 클러스터의 상태를 저장합니다.

- **kube-scheduler**

    kube-scheduler는 파드를 실행시킬 때 요구사항, 하드웨어, 정책 등을 포함한 여러가지 요소들을 고려하여 최적의 노드를 선택합니다.
    만약 적합한 노드가 없을 경우 스케줄러가 배치할 수 있을 때까지 대기하였다가 적합한 노드가 생기면 해당 노드에 Pod를 배치하고 이 정보를 apiserver로 전송합니다.

- **kube-controller manager**

    마스터 노드에서 구성하는 컨트롤러 매니저이며 앞서 언급한 kube-apiserver를 통해 클러스터의 상태를 모니터링하고, 만약 현재 구성이 원하는 설정 상태와 다를 경우 현재 상태를 원하는 상태로 이동하기 위해 변경 및 제어합니다. 

    현재 쿠버네티스에는 아래와 같은 네 가지 컨트롤러가 존재합니다.

    1. 노드 컨트롤러: 노드를 생성하고 초기화하며 노드에 대해서 응답 여부를 체크하여 비활성화/삭제/수정을 진행합니다.
    2. 레플리케이션 컨트롤러: 설정된 값보다 파드가 많이 생성될 경우 삭제, 또는 오류가 있을 경우 삭제 및 재생성하는 작업 등을 통해 지정된 수의 파드 실행을 보장합니다.
    3. 엔드포인트 컨트롤러: 서비스와 파드를 연결하는 역할입니다.
    4. 서비스 & 계정 컨트롤러 : 새로운 네임스페이스 및 계정에 대한 권한 및 인증을 수행하는 역할입니다.

(참조 : [https://kubernetes.io/docs/concepts/overview/components/](https://kubernetes.io/docs/concepts/overview/components/))
(참조 : [https://kubernetes.io/ko/docs/concepts/architecture/](https://kubernetes.io/ko/docs/concepts/architecture/))

<br />

### **Master (Node) / (Worker) Node** 

<div class="img-in-post">
<img src="/images/2/5.png" />
</div>

노드(Node)란 앞서 언급한 클러스터의 컴퓨팅 머신으로 불리는 구성 요소로써 하나의 물리 서버 혹은 VM 하나입니다.
노드는 마스터 노드와 워커 노드로 분류됩니다. 마스터 노드는 마스터, 워커 노드는 노드라고 부르기도 합니다.
마스터 노드는 다른 노드에 명령을 내리는 역할로, 앞서 설명했던 컨트롤 플레인을 포함하고 있습니다.
워커 노드는 마스터 노드가 아닌 노드들이고 마스터 노드에서 오는 명령을 수행하고 서비스가 실제로 동작하는 노드입니다.

#### **Object**

쿠버네티스를 이해하고 사용하는 과정에 있어서 가장 중요한 부분이 오브젝트(Object)입니다.
오브젝트에는 아래에서 설명할 **기본 오브젝트(Basic Object)**와, 이런 오브젝트를 생성하고 관리하는 **컨트롤러 오브젝트(Controller Object)**로 분류됩니다.

#### **Object Spec**

앞서 설명한 오브젝트들은 모두 설정 정보들을 기술한 스펙으로 정의되고, 커맨드 라인 혹은 아래와 같이 yaml, json으로 정의할 수 있습니다.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

(출처 : [https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/kubernetes-objects/](https://kubernetes.io/ko/docs/concepts/overview/working-with-objects/kubernetes-objects/))

#### **Basic Object**

쿠버네티스로 배포되는 가장 기본적인 오브젝트입니다.
Volume, Pod, Service, Namespace 4가지가 존재하며 각각 다음과 같습니다.

- **Volume**

<div class="img-in-post">
<img src="/images/2/6.png" />
</div>

컨테이너 내부에서 문제가 발생하여 컨테이너가 삭제될 경우, 일반적으로 컨테이너 내부에 존재하는 데이터 또한 함께 삭제됩니다.

배포된 이미지에 포함된 컨테이너가 삭제되어도 영향이 없지만, 로그나 데이터베이스 등 컨테이너가 삭제되면서 그 안의 데이터가 같이 삭제될 경우에는 서비스에 장애가 발생할 수 있습니다. 이러한 이유들로 인해 컨테이너를 사용할 때 보존해야 하는 주요 데이터는 Volume Object를 사용하여 보관합니다.

- **Pod** 

<div class="img-in-post">
<img src="/images/2/7.png" />
</div>

파드는 반드시 하나 이상의 컨테이너를 가지는 컨테이너의 집합체이며, 쿠버네티스에서 생성하고 배포할 수 있는 가장 작은 단위라고 생각하면 됩니다. 파드에 속하는 하나 이상의 컨테이너는 같은 노드에 배치되며 IP, Port, Disk, IPC 등을 공유합니다. 또한 파드 내 컨테이너끼리의 통신도 가능합니다.

- **Service** 

<div class="img-in-post">
<img src="/images/2/8.png" />
</div>

서비스는 앞서 말한 파드들을 레이블 셀렉터를 통하여 하나로 묶어서 같은 IP 혹은 도메인으로 접근할 수 있게끔 하는 오브젝트입니다.

- **NameSpace** 

<div class="img-in-post">
<img src="/images/2/9.png" />
</div>

Kubernetes의 네임스페이스는 서비스보다 큰 개념이며 이런 서비스들을 관리할 수 있는 하나의 논리적 구분 단위입니다. 앞서 설명했던 서비스들에 대하여 네임스페이스별로 권한/리소스 등을 분리하여 서비스 혹은 파드를 관리할 수 있습니다.

그 예로 개발 환경과 테스트 환경의 필요한 리소스 혹은 권한이 다르기 때문에 이를 분리하여 운영하지 않을 경우 혼선이 올 수 있는데, 논리적으로 이를 구분하여 개발 환경 따로 테스트 환경 따로 관리하고 사용할 때 네임스페이스를 이용합니다.

앞서 언급한 바와 같이 네임스페이스의 구분 및 분리는 물리적으로 한 것이 아니기 때문에 네임스페이스가 다른 파드끼리 서로 접근할 수도 있습니다.

<br />

#### **Controller Object** 

앞에서 소개한 4개의 기본 오브젝트들을 설정하고 배포하는 과정에서 조금 더 편리하게 생성하고 이를 관리하는 역할을 합니다.
컨트롤러는 Replication Controller, Replication Set, Daemon Set, Deployment 등 여러가지가 존재합니다.
그 중에서 이번 글에서는 ReplicaSet과 Deployment에 대해서 다루어 보도록 하겠습니다.

- **Replica Control(Set)**

    RC라고도 불리는 Replication Controller는 파드(Pod)를 배치하는 역할을 하는데, 지정된 숫자로 Pod를 기동 시키고 관리하는 역할을 합니다. 이는 Selector, Replica 개수, Pod Template 이렇게 세 가지로 구성되고 각각의 설명은 아래와 같습니다.

    ReplicaSet과 Replica Control이 있는데, 동작 방식은 유사하나 Replica Control의 상위 버전이 ReplicaSet이라고 보시면 될 것 같습니다.

    1. Selector : Label을 기반으로 Replication Controller가 파드를 배치할 노드를 가져올 때 사용합니다.
    2. Replica 개수 : Replication Controller 에 의해서 관리되는 파드(Pod)의 개수입니다. Replica의 개수만큼 Pod 의 수를 유지하도록 레플리카 컨트롤러에 의해 제어된다.
    3. Pod template : 파드(Pod)를 추가로 생성 및 운용 할 때 만드는 조건 및 정보 (포트, 도커 이미지 등)에 대하여 정의하는 부분입니다. 

    더 많은 정보는 ([https://kubernetes.io/ko/docs/concepts/workloads/controllers/replicationcontroller/](https://kubernetes.io/ko/docs/concepts/workloads/controllers/replicationcontroller/)) 에서 확인할 수 있습니다.

- **Deployment**

    쿠버네티스에서 어플리케이션 단위를 관리하는 Controller이며 Kubernetes를 배포할 때 배포의 최소 단위인 Pod에 대한 기준스펙을 정의합니다. 
    Kubernetes에서는 각 Object 및 Pod에 대해서 커맨드 라인을 통해 명령어를 전달하여 구성하는 방법도 존재하지만 Deployment를 통해서 생성하는 것을 권장하고 있습니다. 

    Deployment에 대해서 간단하게 설명하면 

    1. 파드 스케일의 기준을 정합니다.
    2. 파드의 배포 및 업데이트를 확인할 수 있습니다.
    3. 파드의 배포에 대해서 롤백을 진행할 수 있습니다. 

- **DaemonSet**

    데몬셋은 클러스터 전체에서 특정 파드를 생성 및 실행할 때 사용하는 컨트롤러 오브젝트입니다. 
    데몬셋을 통하여 파드를 생성하게 되면 현재 클러스터에 새롭게 노드가 추가될 때 해당 노드에
    자동으로 추가되며, 클러스터 전체에 항상 실행시켜두어야 하는 파드를 대상으로 사용합니다.

    사용 용도는 대표적으로 아래와 같습니다.

    1. 모든 노드에서 로그 수집을 진행할 때 사용합니다.
    2. 모든 노드에서 모니터링을 진행할 때 사용합니다.

<br />

## **YAML 작성 예제**

위와 같은 기본 개념들을 바탕으로 기본 오브젝트 및 오브젝트 스펙을 사용하여 실제 작성된 YAML을 분석해보겠습니다. 작성된 YAML은 쿠버네티스 공식 홈페이지에 있는 YAML을 참고했습니다. 

```yaml
apiVersion: apps/v1 #----------------------------------- 1
kind: Deployment 
metadata:
  name: nginx-deployment
  labels:
    app: nginx-deploy
spec: #------------------------------------------------- 2 
  replicas: 3
  selector:
    matchLabels:
      app: nginx-node
  template:
    metadata:
      labels:
        app: nginx-pod
    spec: #-------------------------------------------- 3
      containers:
      - name: nginx-container
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

- **`apiVersion`** : 오브젝트 생성에 필요한 api version
- **`kind`** : 실행할 오브젝트의 종류
- **`metadata`** : 이름이나 문자열, UID, 네임스페이스 등 오브젝트를 구분하는 데이터 
- **`labels`** : 객체를 구성하고 분류하는 데 사용할 수 있는 문자열 키 및 값의 맵
- **`spec`** : 리소스의 원하는 상태를 나타내는 구문
- **`replicas`** : 유지하여야 하는 파드의 개수 
- **`selector`** : 오브젝트 식별자
- **`template`** : 레플리카 수 유지를 위해 생성하는 신규 파드에 대한 데이터를 명시하는 것
- **`image`** : 컨테이너 제작에 사용될 이미지 ( Private Registry 혹은 Dockerhub 사용 ) 
- **`containerPort`** : 컨테이너에 접근할 수 있는 포트

위 내용을 정리하면 다음과 같습니다.

1. deployment 라는 오브젝트를 사용 하는데, 가장 먼저 현재 작성할 deployment의 라벨로 app: nginx-deploy를 설정합니다. 
2. selector를 통해서 현재 동작 중인 노드 중 app: nginx-node 라는 라벨이 부여된 노드에 파드를 생성합니다. 이 때 동작중인 파드의 개수는 3개로 보존하며 각각의 파드 라벨로 app: nginx-pod를 설정합니다. 
3. 동작할 파드의 이름을 nginx-container로 설정하고, 해당 이름을 기반으로 하는 실제 동작할 컨테이너가 생성되는데, 이미지는 도커 허브의 nginx 1.14.2 버전을 사용하며 해당 컨테이너를 80번 포트로 접근할 수 있도록 설정합니다.

여러 오브젝트 및 스펙은 아래 링크에서 참고할 수 있습니다.

(참고 : [https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.20](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.20))

위처럼 작성된 yaml 파일을 kubectl apply -f <yaml 파일 경로> 를 통해서 실행시킬 수 있습니다. 

이번 글에서는 추후 작성할 글의 이해에 필요한 기본적인 정보들에 대해서 다루어 보았습니다.
다음 글에서는 yaml을 작성할 때 일어날 수 있는 보안 문제에 대해서 다루어 보도록 하겠습니다.
