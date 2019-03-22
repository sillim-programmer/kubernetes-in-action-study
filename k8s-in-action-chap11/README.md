<!-- $theme: gaia -->
<!-- template: invert -->
<!-- page_number:true -->
<!-- $size: 16:9 -->

# Chapter 11
# 쿠버네티스 내부

-----------------------------------------

<div style="font-size:75%">

## 아키텍처 이해

* Kubernetes Control Plane
	* etcd Distributed Storage
	* API Server
	* scheduler
	* control manager
* Worker Nodes
	* Kubelet
	* Kubernetes Service Proxy (kube-proxy)
	* container runtime

</div>

-----------------------------------------

<div style="font-size:75%">

## 아키텍처 이해

* Add-On Components
	* Kubernetes DNS Server
	* Dashboard
	* Ingress Controller
	* Heapster
	* Container Network Interface Plugin (CNI)

</div>

-----------------------------------------

<div style="font-size:75%">
  
## 컴포넌트간 상호 종속성

<img src="architecture-01.png" width="900px" />

</div>

--------------------------------------

<div style="font-size:75%">
  
## 컴포넌트의 상태 확인
  

~~~
kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health": "true"}
~~~

</br></br>

## 컴포넌트와 커뮤니케이션 하는 방법

* API Server와만 통신
* 거의 항상 컴포넌트가 API Server로 요청
* kubectl attach, kubectl port-forward 등의 일부 경우에만 API Server가 kubelet에 먼저 연결을 시도

</div>

-----------------------------------------

<div style="font-size:75%">

## 컴포넌트의 고가용성

* Control Plane
	* etcd는 Clustering
	* API Server는 Multi-Active
	* Control Manager, Scheduler는 Active-Standby
* Worker Nodes
	* 전체 컴포넌트가 각각의  Worker Node에서 실행

</div>

-----------------------------------------

<div style="font-size:75%">

## 쿠버네티스와 etcd

* 모든 오브젝트(Pod, RC, Service 등..)는 API Server가 재시작되거나 실패하더라도 유지되거나 복원되기 위해 영구적으로 저장되어야 함.
* Distributed Key-Value Storage인 etcd에 저장.
* API Server외의 컴포넌트는 API Server를 통해 간접적으로 etcd에 접근

</br>

## 쿠버네티스와 API Server를 통해 etcd에 읽고 쓰는 이유

* 저장소 추상화
* 유효성 검사
* 낙관적(Optimistic) Tx (metadata.resourceVersion)

</div>

-----------------------------------------

<div style="font-size:75%">

## etcd에 리소스를 저장하는 방법

~~~
$ export ETCDCTL_API=3
$ etcdctl --endpoints https://127.0.0.1:2379 \
   --cacert /etc/kubernetes/pki/etcd/ca.crt \
   --cert /etc/kubernetes/pki/etcd/server.crt \
   --key /etc/kubernetes/pki/etcd/server.key \
   get / --prefix=true --keys-only

/registry/configmaps
/registry/daemonsets
/registry/deployments
/registry/event
/registry/namespaces
/registry/pods
...
~~~

* /registry/{objectType}/{namespace}/{objectName}

</div>

-----------------------------------------

<div style="font-size:75%">

## etcd 클러스터

* Quorum 유지를 통한 분산된 etcd 클러스터의 일관성
	* 여러 사람의 합의로 운영되는 의사기관에서 의결을 하는데 필요한 최소한의 참석자 수
* RAFT Consensus Algorithm (http://swalloow.github.io/raft-consensus)
* split-brain 방지를 위해 홀수 권장

<img src="architecture-02.jpg" width="650px"/>

</div>

-----------------------------------------

<div style="font-size:75%">

## API 서버가 하는 일

* 클러스터 상태를 조회 및 수정할 수 있는 인터페이스 제공.
* 변경된 상태를 etcd에 저장.
* object의 유효성 검사.
* Optimistic Locking 처리

</div>

-----------------------------------------

<div style="font-size:75%">

## API 서버의 동작

* 인증 플러그인 - 클라이언트 인증
* 권한 승인 플러그인 - 클라이언트 권한 승인
* 승인 제어 플러그인 - 요청 받은 리소스를 조회 및 수정
* resource validation - 리소스의 검증 및 저장

<img src="architecture-03.jpg" width="1000px" />

</div>

-----------------------------------------

<div style="font-size:75%">

## API 서버가 리소스 변경을 통지하는 방법

* API 서버는 etcd의 watch api를 이용하여 subscribe
* API 서버에 리소스 변경 요청이 오면 해당 요청을 etcd에 저장.
* etcd에서 변경이 일어난 key를 publish
* notification을 받은 API 서버는 watch api를 요청한 클라이언트에 notify
* http1.0 : 응답의 일부만 전달해주어 connection을 유지함.
* http1.1 : Chunked Streaming 방식으로 구현 
* Why not implement it as http2?


<img src="architecture-04.jpg" width="800px" />

</div>

-----------------------------------------

<div style="font-size:75%">

## API 서버가 리소스 변경을 통지하는 방법

```
$ vi /etc/kubernetes/manifests/kube-apiserver.yaml
$ systemctl restart kubelet

apiVersion: v1
kind: Pod
metadata:
  ...
spec:
  containers:
  - command:
    ...
    - --insecure-port=8080
```

```
$ curl --http1.0 http://localhost:8080/api/v1/pods?watch=true

$ curl http://localhost:8080/api/v1/pods?watch=true
```


</div>

-----------------------------------------

<div style="font-size:75%">

## API 서버가 리소스 변경을 통지하는 방법

* http1.1
```
$ tcpdump -nlA -i lo port 8080
05:33:24.628863 IP 127.0.0.1.44242 > 127.0.0.1.8080: Flags [P.], seq 1:101, ack 1, win 342, options [nop,nop,TS val 925974024 ecr 925974024], length 100: HTTP: GET /api/v1/pods?watch=true HTTP/1.1
E....w@.@.o..............Q..jn.....V.......
71>.71>.GET /api/v1/pods?watch=true HTTP/1.1
Host: localhost:8080
User-Agent: curl/7.58.0
Accept: */*
05:33:24.629526 IP 127.0.0.1.8080 > 127.0.0.1.44242: Flags [P.], seq 1:117, ack 101, win 342, options [nop,nop,TS val 925974025 ecr 925974024], length 116: HTTP: HTTP/1.1 200 OK
E...;_@.@...............jn...Q.=...V.......
71>	71>.HTTP/1.1 200 OK
Content-Type: application/json
Date: Fri, 22 Mar 2019 05:33:24 GMT
Transfer-Encoding: chunked
9cf
{"type":"ADDED","object":{ ... }}
19a2
{"type":"ADDED","object":{ ... }}
aab
{"type":"MODIFIED","object":{ ... }}
....
```

</div>

-----------------------------------------

<div style="font-size:75%">

## API 서버가 리소스 변경을 통지하는 방법

* http1.0
```
$ tcpdump -nlA -i lo port 8080

05:42:32.087199 IP 127.0.0.1.47318 > 127.0.0.1.8080: Flags [P.], seq 1:101, ack 1, win 342, options [nop,nop,TS val 926521512 ecr 926521512], length 100: HTTP: GET /api/v1/pods?watch=true HTTP/1.0
E...9<@.@.."............rC.u...,...V.......
79..79..GET /api/v1/pods?watch=true HTTP/1.0
Host: localhost:8080
User-Agent: curl/7.58.0
Accept: */*

05:42:32.087785 IP 127.0.0.1.8080 > 127.0.0.1.47318: Flags [P.], seq 1:89, ack 101, win 342, options [nop,nop,TS val 926521513 ecr 926521512], length 88: HTTP: HTTP/1.0 200 OK
E...`c@.@..................,rC.....V.......
79..79..HTTP/1.0 200 OK
Content-Type: application/json
Date: Fri, 22 Mar 2019 05:42:32 GMT

05:42:32.090370 IP 127.0.0.1.8080 > 127.0.0.1.47318: Flags [P.], seq 56470:60566, ack 101, win 342, options [nop,nop,TS val 926521516 ecr 926521515], length 4096: HTTP
{"type":"ADDED", ... "oper
...
```

</div>

-----------------------------------------

<div style="font-size:75%">

## Scheduler의 이해

* pod이 생성되고, node에 할당되지 않았을 때 scheduler가 특정 노드에 pod을 할당 (api-server를 통해 resource만 update)
* pod이 update되면 (pod이 node에 할당되면) 해당 노드의 kubelet이 실제 pod을 생성

</div>

-----------------------------------------

<div style="font-size:75%">

## 기본 스케줄링

* pod이 schedule될 수 있는 노드의 목록을 필터링
* 허용하는 노드 중 우선순위로 정렬하고 최적의 노드를 선택한다

</br>

<img src="architecture-05.jpg" width="1000px" />

</div>

-----------------------------------------

<div style="font-size:75%">

## 수용 가능한 노드 찾기

* node가 pod를 호스팅할 수 있으려면 아래의 사항들을 통과해야 함

	* node가 pod의 request resource 이상의 여분이 있는가?
	* node에 리소스가 부족한가?
	* node가 pod 스펙의 노드 셀렉터에 맞는 라벨을 가졌는가?
	* pod이 특정 호스트 포트에 바인딩을 요청하는 경우 해당 node에 포트가 이미 사용되고 있는가?
	* pod이 특정 볼륨 유형을 요청하는 경우, 이 볼륨을 node의 pod에 마운트할 수 있는가?
	* pod는 node의 taints를 허용하는가?

</div>

-----------------------------------------

<div style="font-size:75%">
  
## Controller 소개

* API 서버는 Resource를 etcd에 저장하고 변경 사항을 통지하기만 함.
* 변경 사항 반영은 Controller Manager에서 실행되는 Controller가 수행.

</br>

## Controller가 하는 일과 동작 방식

* API 서버를 통해 리소스의 변경을 감시하고, 오브젝트의 생성, 갱신, 삭제 등의 동작을 수행.
* 실제 상태를 원하는 상태로 변경을 시도.
* 변경 된 실제 상태를 Resource의 Status 필드에 반영.
* Controller는 Kubelet과 직접 통신하지 않음.

</div>

-----------------------------------------

<div style="font-size:75%">
  
### Replication Manager, ReplicaSet Controller

<img src="architecture-06.jpg" width="800px" />

</div>

-----------------------------------------

<div style="font-size:75%">
  
### DaemonSet, Job Controller

* Pod Resource를 각 Resource에 정의된 Pod Template으로 생성.
* Pod 정의를 API 서버로 전달하여 Kubelet이 Container를 생성하고 실행하게 만듬.

</br>

### Deployment Controller

* API 서버에 저장 된 오브젝트와 동기를 맞추기 위해 실제 Deployment 상태를 유지 관리.
* Deployment Object가 변경될 때 마다 새로운 버전으로 Roll-Out.

</br>

### StatefulSet Controller

* StatefulSet Resource의 Spec에 따라 Pod를 생성, 관리, 삭제.
* 각 Pod Instance의 PersistentVolumeClaim을 인스턴스화 하고 관리.

</div>

-----------------------------------------

<div style="font-size:75%">

### Node Controller

* Cluster의 Node Resource를 관리
* API 서버에서의 Resource 이름은 Nodes, etcd에서의 Key 이름은 minions

### Service Controller

* LoadBalancer Type의 Service가 생성되면 인프라스트럭처로 LoadBalancer를 요청.

</div>

-----------------------------------------

<div style="font-size:75%">

### Endpoint Controller

* Service, Pod, Endpoint Resource를 모두 감시
* Service의 label selector와 매칭되는 pod의 ip와 port를 찾아 Endpoint Resource를 최신 상태로 유지

<img src="architecture-07.jpg" width ="700px" />

</div>

-----------------------------------------

<div style="font-size:75%">

### Namespace Controller

* Namespace Resource가 제거될 때 해당 Namespace에 속한 모든 Resource를 제거.

</br>

### PersistentVolume Controller

* PersistentVolumeClaim이 생성되면 적절한 PersistentVolume을 찾아 Binding.

</div>

-----------------------------------------

<div style="font-size:75%">

## Kubelet

* Worker Node에서 실행되는 모든 것에 책임을 가짐.
* 초기 실행 시 Kubelet이 실행되는 Host를 Node Resource로 등록.
* 해당 Node에 Schedule된 Pod을 모니터링하여 Pod의 Container를 실행.
* 실행 중인 Container를 지속적으로 모니터링하고 상태와 이벤트, 리소스 소모를 API 서버에 통지.
* readness, liveness probe를 실행하는 컴포넌트.
* Pod Resource가 삭제됐을 때 Container를 중지하고 완전히 중지되면 API 서버에 통지.

</div>

-----------------------------------------

<div style="font-size:75%">
  
## Kubelet

* 특정 local directory의 manifests 파일 기반으로 Pod 생성 가능
* Control Plane Pods (ex. kube-apiserver, kube-control-manager ... )
```
$ ls /etc/kubernetes/manifests
etcd.yaml  kube-apiserver.yaml  kube-controller-manager.yaml  kube-scheduler.yaml
```

<img src="architecture-08.jpg" width="700px"/>

</div>

-----------------------------------------

<div style="font-size:75%">

## Kubernetes Proxy

* 모든 Worker Node는 kube-proxy를 실행.
* Service Object로 Client가 연결할 수 있도록 하는 것이 목적.
* Service가 둘 이상의 Pod으로 연결되면 해당 노드에서 LoadBalancing을 수행.
* 초기 버전의 구현은 iptables의 userspace redirect를 이용하여 구현.
* 현재는 일반적인 iptables의 Rule로 구현.

<img src="architecture-09.jpg" width="700px" />

<img src="architecture-10.jpg" width="700px" />

</div>

-----------------------------------------

<div style="font-size:75%">
  
## Kubernetes Add-On

</div>

-----------------------------------------

<div style="font-size:75%">

</div>

-----------------------------------------

<div style="font-size:75%">

</div>

-----------------------------------------

<div style="font-size:75%">

</div>

-----------------------------------------

<div style="font-size:75%">

</div>

-----------------------------------------

<div style="font-size:75%">

</div>

-----------------------------------------

<div style="font-size:75%">

</div>

-----------------------------------------
