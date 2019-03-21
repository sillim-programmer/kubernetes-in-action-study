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

* 저장소 추상화 (like DIP)
* 유효성 검사 위임
* 낙관적(Optimistic) Tx (metadata.resourceVersion)

</div>

-----------------------------------------

<div style="font-size:75%">

## etcd에 리소스를 저장하는 방법

~~~
$ etcdctl ls /registry                    # <= v2
or 
$etcdctl get /registry --prefix=true      # v3 <=

/registry/configmaps
/registry/daemonsets
/registry/deployments
/registry/events
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
