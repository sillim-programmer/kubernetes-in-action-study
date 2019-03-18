<!-- $theme: gaia -->
<!-- template: invert -->
<!-- page_number:true -->
<!-- $size: 16:9 -->

# Chapter 05 
# 서비스

-----------------------------------------

<div style="font-size:75%">

## 서비스가 필요한 이유

* 포드는 일회성
* scale, failure 등의 이유로 노드간 이동이 발생
* 포드가 노드에 할당된 후 시작하기 전에 IP를 할당
* 클라이언트에서 서버의 IP를 알 수 없음

<br><br>

## 서비스?

* 동일한 서비스를 제공하는 Pod 그룹에 대한 단일 진입 지점
* 서비스가 존재하는 동안 변경되지 않는 IP 주소와 Port를 가짐
* 해당 서비스와 연결된 Pod 중 하나로 전달

</div>

-----------------------------------------

<div style="font-size:75%">

<img src="service-1.jpg" width="900px" />

</div>

-----------------------------------------


<div style="font-size:75%">

## 서비스 생성

~~~
apiVersion: v1
kind: Service
metadata:
  svc: sample
spec:
  ports:
    - port: 8080
      targetPort: 8080
  selector:
    svc: sample
~~~

~~~
$ kubectl expose pod sample --type=LoadBalancer --name sample-http \
  --port=8080 --target-port=8080 --selector='svc=sample'
~~~

<img src="service-2.png" />

</div>

-----------------------------------------

<div style="font-size:75%">

## 서비스의 실체..?

* KUBE-NODEPORTS chain
<img src="service-15.png" />
* KUBE-SERVICES chain
<img src="service-16.png" />
* KUBE-SVC-? chain
<img src ="service-17.png" />

</div>

--------------------------------------

<div style="font-size:75%">

## 컨테이너에 명령 실행

<img src="service-4.png" />

<img src="service-3.jpg" width="900px" />

</div>

-----------------------------------------

<div style="font-size:75%">

## 서비스의 Session Affinity 구성

~~~
spec:
  sessionAffinity: ClientIP
~~~

~~~
$kubectl expose pod sample --type=LoadBalancer --name sample-http \
  --session-affinity=ClientIP'
~~~

<br>

* 고정 세션을 구성
* Sticky Session
* None, ClientIP 타입만 지원
* Srouce IP 기반 고정 세션을 구현 (by netfilter)

</div>

-----------------------------------------

<div style="font-size:75%">

## 서비스 검색

* 환경변수를 이용한 서비스 조회

<img src="service-5.png" width="550px" />
<img src="service-6.png" width="550px" />

</div>

-----------------------------------------

<div style="font-size:75%">

## 서비스 검색

* DNS & FQDN을 이용한 서비스 검색

~~~
root@sample-test:/ $ curl -XGET 'http://sample-http:8080'
~~~

~~~
root@sample-test:/ $ curl -XGET 'http://sample-http.default.svc:8080'
~~~

</div>

-----------------------------------------

<div style="font-size:75%">

## 클러스터 외부 서비스 연결

* k8s 바깥의 서비스들을 수동으로 연결 가능
* ExternalName 타입의 서비스는 DNS 레벨에서 구현 (iptables에 관련 룰은 없음)

<img src="service-8.jpg" width="1000px"/>

</div>

-----------------------------------------

<div style="font-size:75%">

## 클러스터 외부 서비스 연결

* k8s 바깥의 서비스들을 수동으로 연결 가능

~~~
apiVersion: v1
kind: Service
metadata:
    name: external-service
spec:
    type: ExternalName
    externalName: external.com
    ports:
        - port: 1234
~~~

~~~
apiVersion: v1
kind: Endpoints
metadata:
  name: external-service
subsets:
  - addresses:
    - ip: 10.0.2.21
    - ip: 10.0.2.22
    ports:
      - port: 1234
~~~

</div>

-----------------------------------------

<div style="font-size:75%">

## 외부 클라이언트

* NodePort 타입의 서비스를 통해 외부 클라이언트에서 접근 할 수 있도록 해줌

~~~
apiVersion: v1
kind: Service
metadata:
  name: sample-http
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30123
  selector:
    svc: sample
~~~

</div>

-----------------------------------------

<div style="font-size:75%">

## 외부 클라이언트

<img src="service-9.jpg" width="700px" />

</div>

-----------------------------------------

<div style="font-size:75%">

## 외부 로드 밸런서를 이용한 노출

<img src="service-10.jpg" width="700px" />

</div>

-----------------------------------------

<div style="font-size:75%">

## 외부 연결의 특성

* 불필요한 네트워크 홉의 방지

~~~
spec:
	externalTrafficPolicy: Local
~~~

<img src="service-14.jpg" width="800px" />

</div>

-----------------------------------------

<div style="font-size:75%">

## 외부 연결의 특성

* KUBE-NODEPORTS Chain (worker node 전체 공통)
<img src="service-11.png" />
* KUBE-XLB Chain (worker node 별로 룰이 달라짐)
* 해당 노드에 전달할 pod가 없는 경우
<img src="service-12.png" />
* 해당 노드에 전달할 pod가 있는 경우
<img src="service-13.png" />

</div>

-----------------------------------------

<div style="font-size:75%">

## 인그레스

<img src="service-18.jpg" width=1200px />

* install
~~~
$ kubectl apply -f \
  https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
$ kubectl apply -f \
  https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/cloud-generic.yaml
~~~

</div>

-----------------------------------------

<div style="font-size:75%">

## 인그레스

~~~
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: sample-ingress
spec:
  rules:
  - http:
      paths:
      - path: /sample
        backend:
          serviceName: sample-service
          servicePort: 8080
~~~

</div>

-----------------------------------------

<div style="font-size:75%">

## 인그레스

<img src="service-19.jpg" width=1000px />

</div>

-----------------------------------------

<div style="font-size:75%">

## Readness Probe

* Exec, Http Get, Tcp 타입 사용 가능

<img src="service-20.jpg" width="1000px"/>

</div>

-----------------------------------------

<div style="font-size:75%">

## Readness Probe

~~~
apiVersion: v1
kind: ReplicationController
...
spec:
  ...
  template:
    ...
    spec:
      containers:
      - name: kubia
        image: luksa/kubia
        readinessProbe:
          exec:
            command:
            - ls
            - /var/ready
        ...
~~~
</div>

-----------------------------------------

<div style="font-size:75%">

## Headless Service

* 각 개별 pod의 ip를 전부 얻어오기 위한 서비스
* CluterIP를 None으로 설정하면 DNS 서버는 단일 서비스 IP 대신 pod의 ip 목록을 전달
* 준비 된 Pod의 목록만 전달 (default)

~~~
apiVersion: v1
kind: Service
metadata:
    name: sample-headless-service
spec:
    clusterIP: None
    ports:
    - port: 8080
      targetPort: 8080
    selector:
        svc: sample
~~~

</div>

-----------------------------------------

<div style="font-size:75%">

## Headless Service

* 준비되지 않은 모든 pod를 찾으려면 아래 주석을 추가

~~~
metadata:
    name: sample-headless-service
    annotations:
        service.alpha.kubernetes.io/tolerate-unready-endpoints: "true"
...
~~~

</div>
