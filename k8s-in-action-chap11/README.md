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
* Long-Polling(http1.0), Chunked Streaming(http1.1) 방식으로 구현 (Why not implement it as http2?).

</br>

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

```
$ tcpdump -nlX -i lo port 8080


```

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

<div style="font-size:75%">

</div>

-----------------------------------------

<div style="font-size:75%">

</div>

-----------------------------------------
