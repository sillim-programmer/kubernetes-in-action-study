오늘 사용했던 tool
- vscode : 코드 에디터 => 사용 extension : kubernetes, Swagger Viewer
- typora : 마크 다운 에디터
- zsh : 쉘

## 8장 애플리케이션에서 포드 메타 데이터와 그 외의 리소스 접근하기

우리는 지난 장에서 포드들이 환경변수나 설정할때 필요한 기본 데이터를 ConfigMap과 Secret 을 통해 전달 받는 방법을 배웠다.



### 8.1 Downward API 를 통한 메타 데이터 전달

- 만약 포드가 설치되고 나서 이후의 데이터(pod의 ip, host node의 이름, pod의 특정 이름  <= 리플리카로 만들경우 등)이 필요한 경우는 어떻하지?
  - ConfigMap 이나 Secret으로는 해결할 수 없다.
- 해당 문제를 해결하기 위한 것이 바로 **Downward API** 이다.
- 애플리케이션에서 API를 요청하여 얻는 데이터가 아니라 k8s api server가 환경변수와 downward API 볼륨에 담아준다.
- 사용가능한 데이터
  - 포드 이름 
  - 포드 ip
  - 포드가 속한 namespace 이름
  - 포드가 실행되는 node의 이름
  - 포드가 실행 중인 service  account 이름
  - 컨테이너의 메모리 및 cpu 한계 및 요청
    - 커널에 요청해서 알 수 있긴함
    - 한계는 넘어가면(swap을 사용하거나 메모리 초과로 컨테이너가 죽음) <=> 요청은 넘어가도 안죽음
  - 포드의 라벨 및 주석
    - 보통 해당 데이터는 볼륨을 통해서 노출됨
    - 그 이유는 실행중에 라벨과 주석이 변경 가능해서 첫 환경변수로 초기화된 컨테이너에서 환경변수를 바꾸는 기능을 도커가 지원하지 않기 때문
- 결과적으로 실행될때 환경설정값을 담는 기능 => 실행중에 변경 x
- 라벨과 주석은 변경가능하기 때문에 => 환경변수 보다는 볼륨에 담아서 자주 씀
- 또는 볼륨은 환경변수에 쓰이는 변수가 다른 컨테이너(동일한 포드에서)에 전달하는 패턴에도 사용한다.
- 메타데이터가 상당히 제한적 이지만 해당 정보가 필요하다면 셸 스크립트나 코드 필요없이 간단히 쓰기 좋다.

### 8.2 쿠버네티스 API 서버와 통신하기

- Downward API의 경우 현재 포드의 메타데이터만 접근가능하다. 다른 포드의 데이터 혹은 다른 리소스에 대해서 접근할 필요가 생길 경우에는 접근이 불가능하다.
- 다른 리소스에 대한 데이터가 필요하거나 실시간 정보가 필요한 경우 k8s api와 직접 통신하는게 제일 좋다



### 8.2.1 k8s api 탐색해보기

k8s master 서버에 직접 전송해서 테스트 해볼 수 있지만 인증토큰을 전달해줘야 되기 때문에 간편하게 proxy로 하자.

#### 통신 테스트 해보기

```Shell
$ kubectl proxy
```

#### 결과

```shell
Starting to serve on 127.0.0.1:8001
```

127.0.0.1:8001에다가 curl을 쏴주면 통신해 볼 수 있다.



#### api 테스트

```shell
$ curl 127.0.0.1:8001
```

#### 결과

```shell
{
  "paths": [
    "/api",
    "/api/v1",
    "/apis",
    "/apis/",
    "/apis/admissionregistration.k8s.io",
    "/apis/admissionregistration.k8s.io/v1beta1",
    "/apis/apiextensions.k8s.io",
    "/apis/apiextensions.k8s.io/v1beta1",
    "/apis/apiregistration.k8s.io",
    "/apis/apiregistration.k8s.io/v1",
    "/apis/apiregistration.k8s.io/v1beta1",
    "/apis/apps",
    "/apis/apps/v1",
    "/apis/apps/v1beta1",
    "/apis/apps/v1beta2",
    "/apis/authentication.k8s.io",
    "/apis/authentication.k8s.io/v1",
    "/apis/authentication.k8s.io/v1beta1",
    "/apis/authorization.k8s.io",
    "/apis/authorization.k8s.io/v1",
    "/apis/authorization.k8s.io/v1beta1",
    "/apis/autoscaling",
    "/apis/autoscaling/v1",
    "/apis/autoscaling/v2beta1",
    "/apis/batch",
    "/apis/batch/v1",
    "/apis/batch/v1beta1",
    "/apis/certificates.k8s.io",
    "/apis/certificates.k8s.io/v1beta1",
    "/apis/certmanager.k8s.io",
    "/apis/certmanager.k8s.io/v1alpha1",
    "/apis/crd.k8s.amazonaws.com",
    "/apis/crd.k8s.amazonaws.com/v1alpha1",
    "/apis/events.k8s.io",
    "/apis/events.k8s.io/v1beta1",
    "/apis/extensions",
    "/apis/extensions/v1beta1",
    "/apis/management.cattle.io",
    "/apis/management.cattle.io/v3",
    "/apis/networking.k8s.io",
    "/apis/networking.k8s.io/v1",
    "/apis/policy",
    "/apis/policy/v1beta1",
    "/apis/project.cattle.io",
    "/apis/project.cattle.io/v3",
    "/apis/rbac.authorization.k8s.io",
    "/apis/rbac.authorization.k8s.io/v1",
    "/apis/rbac.authorization.k8s.io/v1beta1",
    "/apis/scheduling.k8s.io",
    "/apis/scheduling.k8s.io/v1beta1",
    "/apis/storage.k8s.io",
    "/apis/storage.k8s.io/v1",
    "/apis/storage.k8s.io/v1beta1",
    "/healthz",
    "/healthz/autoregister-completion",
    "/healthz/etcd",
    "/healthz/ping",
    "/healthz/poststarthook/apiservice-openapi-controller",
    "/healthz/poststarthook/apiservice-registration-controller",
    "/healthz/poststarthook/apiservice-status-available-controller",
    "/healthz/poststarthook/bootstrap-controller",
    "/healthz/poststarthook/ca-registration",
    "/healthz/poststarthook/generic-apiserver-start-informers",
    "/healthz/poststarthook/kube-apiserver-autoregistration",
    "/healthz/poststarthook/rbac/bootstrap-roles",
    "/healthz/poststarthook/scheduling/bootstrap-system-priority-classes",
    "/healthz/poststarthook/start-apiextensions-controllers",
    "/healthz/poststarthook/start-apiextensions-informers",
    "/healthz/poststarthook/start-kube-aggregator-informers",
    "/healthz/poststarthook/start-kube-apiserver-admission-initializer",
    "/healthz/poststarthook/start-kube-apiserver-informers",
    "/metrics",
    "/openapi/v2",
    "/swagger-2.0.0.json",
    "/swagger-2.0.0.pb-v1",
    "/swagger-2.0.0.pb-v1.gz",
    "/swagger.json",
    "/swaggerapi",
    "/version"
  ]
}
```



나머지는 해당 path와 통신하면서 테스트 해보자.



### 8.2.2 포드안에서 api 서버에 통신하기

default 네임스페이스에 생성된 kubernetes 에 요청하면 가능하다.



지난 5장쯤에서 서비스는 DNS로 요청할 수 있다는 것을 알았다.

```
$ curl https://kubernetes
```

여기서 kubernetes는 service 이름이 kubernetes이기 때문에 가능한거다.



### 8.2.3 앰버서더 컨테이너와 API 서버 통신 간소화

이번절에서 앞에서 kubectl proxy를 통해서 요청했던 것을 포드에서 구현하는 방식을 설명한다

![ambassador containerì ëí ì´ë¯¸ì§ ê²ìê²°ê³¼](https://www.oreilly.com/library/view/designing-distributed-systems/9781491983638/assets/ddis_04in01.png)



포드안에 컨테이너는 동일한 loopback 네트워크 인터페이스를 공유해서 localhost로 접근할 수 있다.



Kubectl-proxy 컨테이너를 생성하여 localhost를 통해서 요청해보자.



### 8.2.4 클라이언트 라이브러리를 사용해 API 서버와 통신

앞에 앰버서더 컨테이너를 통해서 요청할 수 있듯이 라이브러리를 통해 코드로 해결할 수 있다.

책에서는 자바로 하는 것을 보여준다.
