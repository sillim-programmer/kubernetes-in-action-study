
# 9 디플로이먼트: 애플리케이션을 선언적으로 업데이트

## 9.3 선언적으로 애플리케이션을 업데이트하기 위한 디플로이먼트 사용

디플로이먼트:
* 애플리케이션을 배포하고 이를 선언적으로 업데이트하는 데 사용하는 상위 수준 리소스
* 디플로이먼트가 생성하는 레플리카셋이 실제로 포드를 관리

![그림9.8](https://dpzbhybb2pdcj.cloudfront.net/luksa/Figures/09fig08.jpg)

### 9.3.1 디플로이먼트 생성

* 디플로이먼트 설정은 레플리카셋과 유사
* 원하는 복제본 수, 라벨 실렉터, 포드 템플릿으로 구성

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kubia
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
        - image: luksa/kubia:v1
          name: nodejs
```

생성: kubectl create -f kubia-deployment-v1.yaml

**디플로이먼트 생성하자마자 포드 상태**
```
[root@k8s-master vagrant]# kubectl get pod
NAME                     READY   STATUS              RESTARTS   AGE
kubia-6b646f577b-8z2mx   0/1     ContainerCreating   0          16s
kubia-6b646f577b-hdd6k   0/1     ContainerCreating   0          16s
kubia-6b646f577b-skvpt   0/1     ContainerCreating   0          16s
```

**디플로이먼트 롤아웃 상태**
```
# kubectl rollout status deployment kubia
Waiting for deployment "kubia" rollout to finish: 0 of 3 updated replicas are available...
Waiting for deployment "kubia" rollout to finish: 1 of 3 updated replicas are available...
deployment "kubia" successfully rolled out
```

**디플로이먼트 이후 포드 상태**
```
[root@k8s-master vagrant]# kubectl get pod
NAME                     READY   STATUS    RESTARTS   AGE
kubia-6b646f577b-8z2mx   1/1     Running   0          9m35s
kubia-6b646f577b-hdd6k   1/1     Running   0          9m35s
kubia-6b646f577b-skvpt   1/1     Running   0          9m35s
```

**디플로이먼트 이후 리플리카셋 상태**
```
[root@k8s-master ~]# kubectl get rs
NAME               DESIRED   CURRENT   READY   AGE
kubia-6b646f577b   3         3         3       14m
```
*포드의 이름은 레플리카셋 이름에 포함된 해시값 사용*

```
[root@k8s-master ~]# kubectl get services
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP   6h35m
kubia        ClusterIP   10.96.132.53   <none>        80/TCP    8s

[root@k8s-master ~]# curl 10.96.132.53
This is v1 running in pod kubia-6b646f577b-p9vh2

[root@k8s-master ~]# curl 10.96.132.53
This is v1 running in pod kubia-6b646f577b-sfgqk

[root@k8s-master ~]# curl 10.96.132.53
This is v1 running in pod kubia-6b646f577b-p9vh2
```

### 9.3.2 디플로이먼트 업데이트

디플로이먼트 전략
* RollingUpdate: 롤링 업데이트
* Recreate: 기존 포드 모구 삭제 후 새로운 포드 생성

롤링 업데이트 확인
* 일정 주기로 서비스를 호출해서 변경 확인
* kubia V1을 V2로 변경

**일정 주기로 서비스 호출**
```
[root@k8s-master ~]# while true; do curl http://10.96.132.53; sleep 2; done
```

**kubia V1을 V2로 변경**
```
[root@k8s-master ~]# kubectl replace -f kubia-deployment-v2.yaml
deployment.apps/kubia replaced
```

**변경 후 출력**
```
This is v1 running in pod kubia-6b646f577b-p9vh2
This is v1 running in pod kubia-6b646f577b-p9vh2
This is v2 running in pod kubia-6969c946dc-xhqrf
This is v2 running in pod kubia-6969c946dc-xhqrf
This is v1 running in pod kubia-6b646f577b-9kp7q
This is v1 running in pod kubia-6b646f577b-sfgqk
This is v1 running in pod kubia-6b646f577b-sfgqk
This is v2 running in pod kubia-6969c946dc-xhqrf
This is v2 running in pod kubia-6969c946dc-xhqrf
This is v2 running in pod kubia-6969c946dc-xhqrf
This is v1 running in pod kubia-6b646f577b-p9vh2
This is v1 running in pod kubia-6b646f577b-sfgqk
This is v2 running in pod kubia-6969c946dc-xhqrf
This is v1 running in pod kubia-6b646f577b-sfgqk
This is v2 running in pod kubia-6969c946dc-cc6v8
This is v2 running in pod kubia-6969c946dc-xhqrf
This is v2 running in pod kubia-6969c946dc-cc6v8
This is v2 running in pod kubia-6969c946dc-kw4vj
This is v2 running in pod kubia-6969c946dc-xhqrf
This is v2 running in pod kubia-6969c946dc-xhqrf
This is v2 running in pod kubia-6969c946dc-cc6v8
This is v2 running in pod kubia-6969c946dc-kw4vj
This is v2 running in pod kubia-6969c946dc-kw4vj
This is v2 running in pod kubia-6969c946dc-cc6v8
```

**업데이트 이후 리플리카셋**
```
[root@k8s-master ~]# kubectl get rs
NAME               DESIRED   CURRENT   READY   AGE
kubia-6969c946dc   3         3         3       9m11s
kubia-6b646f577b   0         0         0       35m
```

![그림9.10](https://dpzbhybb2pdcj.cloudfront.net/luksa/Figures/09fig10_alt.jpg)

이전 리플리카셋은 롤백 용으로 사용

### 9.3.3 디플로이먼트 롤백

오류 발생 버전으로 업데이트 후 롤백 확인

**오류 발생 버전으로 업데이트**
```
[root@k8s-master ~]# kubectl replace -f kubia-deployment-v3.yaml
deployment.apps/kubia replaced
```

**출력 로그**
```
[root@k8s-master ~]# while true; do curl http://10.96.132.53; sleep 2; done
...생략
This is v2 running in pod kubia-6969c946dc-cc6v8
This is v2 running in pod kubia-6969c946dc-xhqrf
This is v3 running in pod kubia-dc86d7c4-h9h8z
This is v2 running in pod kubia-6969c946dc-cc6v8
This is v3 running in pod kubia-dc86d7c4-k8jrz
This is v3 running in pod kubia-dc86d7c4-h9h8z
This is v2 running in pod kubia-6969c946dc-xhqrf
This is v2 running in pod kubia-6969c946dc-xhqrf
This is v3 running in pod kubia-dc86d7c4-h9h8z
This is v3 running in pod kubia-dc86d7c4-k8jrz
This is v3 running in pod kubia-dc86d7c4-8zbch
This is v3 running in pod kubia-dc86d7c4-k8jrz
This is v3 running in pod kubia-dc86d7c4-h9h8z
Some internal error has occurred! This is pod kubia-dc86d7c4-h9h8z
This is v3 running in pod kubia-dc86d7c4-8zbch
Some internal error has occurred! This is pod kubia-dc86d7c4-h9h8z
This is v3 running in pod kubia-dc86d7c4-8zbch
This is v3 running in pod kubia-dc86d7c4-k8jrz
Some internal error has occurred! This is pod kubia-dc86d7c4-h9h8z
This is v3 running in pod kubia-dc86d7c4-8zbch
Some internal error has occurred! This is pod kubia-dc86d7c4-k8jrz
Some internal error has occurred! This is pod kubia-dc86d7c4-8zbch
Some internal error has occurred! This is pod kubia-dc86d7c4-k8jrz
Some internal error has occurred! This is pod kubia-dc86d7c4-k8jrz
```

**롤아웃 되돌리기**

```
[root@k8s-master ~]# kubectl rollout undo deployment kubia
deployment.extensions/kubia rolled back
```

**출력 로그**
```
Some internal error has occurred! This is pod kubia-dc86d7c4-k8jrz
Some internal error has occurred! This is pod kubia-dc86d7c4-8zbch
Some internal error has occurred! This is pod kubia-dc86d7c4-k8jrz
This is v2 running in pod kubia-6969c946dc-vjv6r
Some internal error has occurred! This is pod kubia-dc86d7c4-k8jrz
Some internal error has occurred! This is pod kubia-dc86d7c4-k8jrz
Some internal error has occurred! This is pod kubia-dc86d7c4-k8jrz
Some internal error has occurred! This is pod kubia-dc86d7c4-k8jrz
This is v2 running in pod kubia-6969c946dc-wwd8h
This is v2 running in pod kubia-6969c946dc-wwd8h
Some internal error has occurred! This is pod kubia-dc86d7c4-k8jrz
This is v2 running in pod kubia-6969c946dc-nrwtp
This is v2 running in pod kubia-6969c946dc-vjv6r
This is v2 running in pod kubia-6969c946dc-nrwtp
This is v2 running in pod kubia-6969c946dc-wwd8h
```

**롤아웃 히스토리**

```
[root@k8s-master ~]# kubectl rollout history deployment kubia
deployment.extensions/kubia
REVISION  CHANGE-CAUSE
1         <none>
3         <none>
4         <none>
```

**특정 디플로이먼트 리비전으로 롤백**
```
kubectl rollout undo deployment kubia --to-revision=1
```

히스토리 보관 제한 설정:
* spec.revisionHistoryLimit: 리비전 히스토리 개수 설정. 기본값은 10.

### 9.3.4 롤링 업데이트 롤아웃 속도 통제

spec.spec.strategy.rollingUpdate의 두 속성으로 제어
* maxSurge: 지정한 리플리카 개수를 초과할 수 있는 포드 인스턴스(비율 또는 개수)
* maxUnavailable: 지정한 리플리카 개수 대비 사용할 수 없는 포드 인스턴수(비율 또는 개수)

예: replicas=3, maxSurge=1, maxUnavailable=0
![그림9.12](https://dpzbhybb2pdcj.cloudfront.net/luksa/Figures/09fig12_alt.jpg)

예: replicas=3, maxSurge=1, maxUnavailable=1
![그림9.13](https://dpzbhybb2pdcj.cloudfront.net/luksa/Figures/09fig13_alt.jpg)

### 9.3.5 롤아웃 프로세스 일시 중지

**중지**
```
# kubectl rollout pause deployment kubia
```

**재개**
```
# kubectl rollout resume deployment kubia
```

### 9.3.6 잘못된 버전 롤아웃 방지

레디니스프로브를 이용해서 잘못된 버전 롤아웃 방지

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubia
spec:
  minReadySeconds: 10
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: kubia
  template:
    metadata:
      name: kubia
      labels:
        app: kubia
    spec:
      containers:
        - image: luksa/kubia:v3
          name: nodejs
          readinessProbe:
            periodSeconds: 1
            httpGet:
              path: /
              port: 8080
```

**레디니스프로브 설정 적용 후 잘못된 버전 롤아웃**

```
[root@k8s-master ~]# kubectl replace -f kubia-deployment-v3-readiness.yaml
```

**롤아웃 후 포드 상태**

```
[root@k8s-master ~]# kubectl get po
NAME                     READY   STATUS    RESTARTS   AGE
kubia-6b646f577b-n6nzx   1/1     Running   0          36m
kubia-6b646f577b-nq7nk   1/1     Running   0          35m
kubia-6b646f577b-zlbvf   1/1     Running   0          36m
kubia-bb464699b-gjw7p    0/1     Running   0          41s
```

![그림9.14](https://dpzbhybb2pdcj.cloudfront.net/luksa/Figures/09fig14_alt.jpg)

```
[root@k8s-master ~]# kubectl describe deploy kubia
Name:                   kubia
Namespace:              default
CreationTimestamp:      Fri, 08 Feb 2019 21:41:32 +0900
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 6
Selector:               app=kubia
Replicas:               3 desired | 1 updated | 4 total | 3 available | 1 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        10
RollingUpdateStrategy:  0 max unavailable, 1 max surge
Pod Template:
  Labels:  app=kubia
  Containers:
   nodejs:
    Image:        luksa/kubia:v3
    Port:         <none>
    Host Port:    <none>
    Readiness:    http-get http://:8080/ delay=0s timeout=1s period=1s #success=1 #failure=3
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    ReplicaSetUpdated
OldReplicaSets:  kubia-6b646f577b (3/3 replicas created)
NewReplicaSet:   kubia-bb464699b (1/1 replicas created)
```

**롤아웃 취소**

```
[root@k8s-master ~]# kubectl rollout undo deployment kubia
deployment.extensions/kubia rolled back
```

**취소후 포드 상태: 잘못된 포드 삭제**

```
[root@k8s-master ~]# kubectl get po
NAME                     READY   STATUS        RESTARTS   AGE
kubia-6b646f577b-n6nzx   1/1     Running       0          41m
kubia-6b646f577b-nq7nk   1/1     Running       0          41m
kubia-6b646f577b-zlbvf   1/1     Running       0          41m
kubia-bb464699b-gjw7p    0/1     Terminating   0          6m16s
```
