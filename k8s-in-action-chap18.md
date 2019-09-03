<<<<<<< Updated upstream
# 18장. 쿠버네티스 확장하기
- 거의 다옴 
 
## 18.1 사용자 지정 API 객체 정의하기 Defining Custom API Object
-  쿠버네티스 에코시스템이 진화함에 따라, 하이레벨의 객체를 볼것이다.
- 쿠버네티스가 지금 지원하는 리소스보다 더 구체화 될것이이다.
- 디플로이먼트 서비스 Configmap 을 다루는 대신에 
 전체 어플리캐션 이나 소프트웨어서비스를 나타내는 객체를 만들고 관리할것이다. 

 예를 들어 쿠버네티스 클러스트에서 메시징 브로커를 실행하려면 큐 리소스의 인스턴스를 만들기만 하면 되며 모든 시크릿 ,디플로이먼트 , 서비스 가 사용자 지정 리소스 컨트롤러에 의해 생성된다.
 쿠버네티스는 이미 이와 같은 사용자 리소스를 추가하는 방법을 제공하고 있다.


### 18.1.1 CustomResourceDefinitions 소개
- 새로운 리소스 유형을 정의하려면 CustomResourceDefinition 객체 CRD 를 쿠버네티스
API 서버에 게시만 하면 된다.

- 사용자 지저 리소스 유형에 대한 설명이다.

- CRD가 게시되면 사용자는 다른 리소스와 마친가지로 JSON 또는 YAML매니페스트를 API에 게시해 사용자 지정 리소스의 인스턴스를 만들 수 있다.

`쿠버네티스 1.7 이전에는 CRD 와 유사하지만 1.8에서 제거된 Third-PartyResource 객체를 통해 사용자 리소스를가 정의됐다.`

- 사용자가 새로운 유형의 객체를 만들 수 있도록 CRD를 생성하는 것은 클러스터에서 가시적인 일이 발생하지 않는 경우 유용한 기능이 아니다.

- 각 CRD 에는 대게 관련 컨트롤러가 있다. 이는 모든 핵심 쿠버네티스 리소스에 연결된 컨트롤러가 있는 것과 같은 방식

- 사용자 지정 객체 인스턴스 추가하는것 외에 CRD 에서 수행할 수 있는 작업을 제대로 표시하려면 컨트롤러도 배포해야 한다.

####  CustomResourceDefinition 예제 소개 

1. 팟, 서비스 및 기타 Kubernetes 리소스를 다루지 않고도 Kubernetes 클러스터 사용자가 가능한 한 쉽게 정적 웹 사이트를 실행할 수 있도록 허용한다고 가정 해 봅시다. 당신이 얻고 자하는 것은 사용자가 웹 사이트의 이름과 웹 사이트의 파일 (HTML, CSS, PNG 및 기타)을 가져와야하는 소스 이상을 포함하는 유형의 웹 사이트 개체를 만드는 것입니다. 
2. Git 저장소를 해당 파일의 소스로 사용합니다. 
3. 사용자가 웹 사이트 리소스의 인스턴스를 생성하면 Kubernetes가 그림 18.1과 같이 새로운 웹 서버 팟을 회전시켜 서비스를 통해 노출 시키길 원할 것입니다. 
4. 웹 사이트 리소스를 만들려면 사용자가 다음 목록에 표시된 줄에 매니페스트를 게시해야합니다.

![텍스트](https://dpzbhybb2pdcj.cloudfront.net/luksa/Figures/18fig01_alt.jpg)  
**각각의 웹사이트 객체는 서비스 및 HTTP 서버 팟을 생성해야 한다.**

`가상의 사용자 지정 리소스 : imaginary-kubia-website.yml p.733`
```
kind: Website
metadata:
  name: kubia
spec:
  gitRepo: https://github.com/luksa/kubia-website-example.git
```
- 리소스에는 kind 및 metadata.name 필드가 포함돼 있다.
- 대부분의 리소스와 마찬가지로 spec 세션 포함
- -it는 웹사이트의 파일이 들어있는 git repo를 지정
- appVersion 필드를 포함해야 하지만 사용자 지정 리소스의 값이 무언이지 아직 모른다.
- 이 리소스를 쿠버네티스에 게시하려고 하면 쿠퍼네티스가 웹사이트 객체가 아직 무엇인지 알지 못하기 때문에 오류 발생
  
```
$ kubectl create -f imaginary-kubia-website.yaml
error: unable to recognize "imaginary-kubia-website.yaml": no matches for
➥  /, Kind=Website
```
`사용자 지정 객체의 인스턴스를 만들기 전에 쿠버네티스가 해당 객체를 인식 하도록 해야함`

####  CustomResourceDefinition 객체 생성 객체 생성 
 - 쿠버네티스가 사용자 지정 웹사이트 리소스 인스턴스 허용하려면 CRD를 API 서버에 게시해야햠

`CustomerResourceDefinition 매니페스트 : website-crd.yml p.734`
```
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: websites.extensions.example.com
spec:
  scope: Namespaced
  group: extensions.example.com
  version: v1
  names:
    kind: Website
    singular: website
    plural: websites
```

`디스크립터를 쿠버네티스에 게시하면 사용자 지정 웹사이트 리소스의 인스턴스를 원하는 만큼 만들 수 있음`

```
$ kubectl create -f website-crd-definition.yaml
customresourcedefinition "websites.extensions.example.com" created
```
- CRD 긴이름을 사용하는 이유는 충돌 방지 위함  

#### 사용자 지정 리소스의 인스턴스 생성

`사용자 지정 웹사이트 리소스 kubia-website.yml p.735`
```
apiVersion: extensions.example.com/v1
kind: Website
metadata:
  name: kubia
spec:
  gitRepo: https://github.com/luksa/kubia-website-example.git
```

```
$ kubectl create -f kubia-website.yaml
website "kubia" created
```

#### 사용자 지정 리소스의 인스턴스 검색
- 클러스터의 웹사이트 모드를 나열해보자  
  
```
$ kubectl get websites
NAME      KIND
kubia     Website.v1.extensions.example.com
```

`API 서버에서 가져온 전체 웹사이트 리소스 정의`

```
$ kubectl get website kubia -o yaml
apiVersion: extensions.example.com/v1
kind: Website
metadata:
  creationTimestamp: 2017-02-26T15:53:21Z
  name: kubia
  namespace: default
  resourceVersion: "57047"
  selfLink: /apis/extensions.example.com/v1/.../default/websites/kubia
  uid: b2eb6d99-fc3b-11e6-bd71-0800270a1c50
spec:
  gitRepo: https://github.com/luksa/kubia-website-example.git
```


#### 사용자 지정 리소스의 인스턴스 삭제
- 사용장 지정 객체 인스턴스 삭제도 할 수 있다.

```
$ kubectl delete website kubia
website "kubia" deleted
```

- CustomResourceDefinition 객체 생성하면 API를 서버를 통해 사용자 지정 객체를 저장,검색, 삭제할 수 있다.
- 뭔가를 동작하게 하려면 컨트롤러를 만들어야 한다.

- 일반적으로 사용자 지정 객체를 생성하는 것이 항상 객체를 생성할 때 일어나는것은 아니다.
- 특정 사용자 지정 객체는 ConfigMap과 같은 더 일반적인 메카니즘을 사요하는 데신 데이터를 저장하는데 사용된다.
- 팟 내부에서 실행되는 어플리케이션은 해당 객체의 API서버를 조회하고 그 객체에 저장된 객체를 읽을 수 있다. 그러나 이경우 웹사이트 객체가 존재함으로 인해 객체에서 참조회 Git 내용을 제공하는 웹서버가 스핀업 하게 된다.
  
  
### 18.1.2 사용자 지정 컨트롤러를 사용한 사용자 지정 리소스 자동화
- 웹사이트 객체가 서비스를 통해 노출된 웹서버 팟을 실행하게 하려면 웹사이트 컨트롤러를 빌드하고 배포해야 한다.  
![18.2](https://dpzbhybb2pdcj.cloudfront.net/luksa/Figures/18fig02.jpg  ) 


#### 웹사이트 컨트롤러가 하는일
시작하면 컨트롤러는 다음 URL로 웹사이트 객체 감시

`http://localhost:8001/apis/extensions.example.com/v1/websites?watch=true`

- API 서버에 직접 연결하지 않음
- 동일한 팟의 사이트카 컨테이너에서 실행되고 API 서버의 앰배서더 역할을 하는 kubectl 프록시 프로세스에 연결
- 프로시는 TLS암호화 및 인증을 처리하면서 요청을 API 서버로 전달.
  
![18.3](https://dpzbhybb2pdcj.cloudfront.net/luksa/Figures/18fig03_alt.jpg  ) 

- HTTP GET 요청에 의해 API 서버는 모든 웹사이트 객체의 변경시마마다 감시 이벤트를 보낸다.
- API 서버는 새로운 웹사이트 객체가 만들어 질 때마다 ADDED 감시 이벤트를 보낸다.
- 컨트롤러는 ADDED를 받으면 watch 이벤트에 받은 객체에서 웹사이트 이름 Git Repo URL 추출 JSON 매니페스트를 API서버에 게시하여 디플로이먼트 및 서비스 객체를 만든다.
  
![18.4](https://dpzbhybb2pdcj.cloudfront.net/luksa/Figures/18fig04.jpg  ) 
- 디플로이먼트 리소스에는 두개의 컨테이너 가 있는 팟 템플릿이 있다.
- 하나는 nginx 서버를 실행 다른 하나는 gitsync 프로세스를 실행해 로컬 Git Repo를 동기화댄 상태로 유지
- 로컬 디렉터리가 emptyDir 볼륨을 통해 nginx를 컨테이너와 공유 한다.
- 서비스는 각 노드의 임의 포트(모든 노드에서 동일한 포트 사용)를 통해 웹서버 팟을 호ㅉ출하는 노드 포트 서비스
- 팟이 디플로이먼트 객체에 의해 생성되면 클라이언트는 노드 포트를 통해 웹사이트에 엑세스 할 수 있다.
- API 서버는 웹사이트 리소스 인스턴스 삭제될 때 DELETED 감시 이벤트 보냄 이벤트 수신시 컨트롤러는 이전에 작성한 디플로이먼트 밋 서비스 리소스를 삭제한다.
- 사용자가 웹사이트 인스턴스를 삭제하면 컨트롤러는 해당 웹사이트 게공하는 웹서버 종료 및 제거 
   


#### 팟으로 컨트롤러 실행
- 개발 과정에서 로컬의 개발 래탑에서 컨트롤러를 실행하고 쿠버네티스 API 서버의 앰배서더로 로컬에서 실행되는 kubectl 프록시 프로세스 (팟으로 실행되지 않음)를 사용.
- 소스 코드 변경시마다 컨테이너 이미지를 만들후 쿠버네티스에서 실행할 필요가 없었기에 빨리 개발 할 수 있었다.
- 프로덕션에 컨트롤러를 배포가 준비 되었을때. 가장 좋은 방법은 쿠버네티스 자체적으로 컨트롤러를 실행하는 것이다.

`웹사이트 컨트롤러 디플로이먼트 : webside-controller.yaml p.741`

```
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: website-controller
spec:
  replicas: 1
  template:
    metadata:
      name: website-controller
      labels:
        app: website-controller
    spec:
      serviceAccountName: website-controller
      containers:
      - name: main
        image: luksa/website-controller
      - name: proxy
        image: luksa/kubectl-proxy:1.6.2
```
- 디플로이먼트에서 두 컨테이너 팟의 단일 복제본을 배포한다.
- 하나의 컨테이너는 컨트롤러를 실행한다.
- 다른 컨테이너는 API 서버와의 좀 더 간다한 통신에 사용되는 앰베서더 컨테이너이다.
- 팟 자체 ServiceAccount 아래에서 실행되기 때문에 컨트롤러 배포전 팟을 만들어야 한다.

```
$ kubectl create serviceaccount website-controller
serviceaccount "website-controller" created
```

- Role Based Access Control (RBAC) 을 사용하는 경우 쿠버네티스는 컨트롤러가 웹사이트 리소스를 보거나 디플로이먼트 또는 서비스를 만들지 못하게 합니다.
- 이를 강킁헤 하려면 클러스트롤바인딩을 생성해 웹사이트 컨트롤러 ServiceAccount를 clust-admin 클러스트롤에 바인드 해야 한다.  
 
```
$ kubectl create clusterrolebinding website-controller
➥  --clusterrole=cluster-admin
➥  --serviceaccount=default:website-controller
clusterrolebinding "website-controller" created
```

`ServiceAccount와 클러스터롤바인딩을 제자리에 두면 컨트롤러의 디플로이먼트를 배포할 수 있다`

#### 동작죽인 컨트롤러 보기
- 컨트롤러 실행 중일 때 kubia 웹사이트 리소스를 다시 만든다.

```
$ kubectl create -f kubia-website.yaml
website "kubia" created
```

```
$ kubectl logs website-controller-2429717411-q43zs -c main
2017/02/26 16:54:41 website-controller started.
2017/02/26 16:54:47 Received watch event: ADDED: kubia: https://github.c...
2017/02/26 16:54:47 Creating services with name kubia-website in namespa...
2017/02/26 16:54:47 Response status: 201 Created
2017/02/26 16:54:47 Creating deployments with name kubia-website in name...
2017/02/26 16:54:47 Response status: 201 Created
```
`복붙 오졌다. 2017년`

```
$ kubectl get deploy,svc,po
NAME                        DESIRED   CURRENT   UP-TO-DATE   AVAILABLE  AGE
deploy/kubia-website        1         1         1            1          4s
deploy/website-controller   1         1         1            1          5m

NAME                CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
svc/kubernetes      10.96.0.1      <none>        443/TCP        38d
svc/kubia-website   10.101.48.23   <nodes>       80:32589/TCP   4s

NAME                                     READY     STATUS    RESTARTS   AGE
po/kubia-website-1029415133-rs715        2/2       Running   0          4s
po/website-controller-1571685839-qzmg6   2/2       Running   1          5m
```


### 18.1.3 사용자 지정 객체 검증하기
- 웹사이트 CustomResourceDefinition 에서 유효성 검사 스키마도 지정하지 않았다.
- 사용자는 웹사이트 객체에 원하는 필드를 포함 시킬수 있다. 
- API 서버는 apVersion, kind, metadata 같은 일반적인 필트 제외하고 YAML  내용을 확인하지 않으므로 잘못된 웹사이트 객체를 만들수 있다 (ex: gitRepo 필드 없이)
- 컨트롤러에 유효성 추가하고 잘못된 객체를 API 서버에서 거부 할 수 없다.
- API 서버가 객체 저장후 다음 클라이언트(kubeclt)에 성공 응답을 반화하고 모든 감시자에게 알린다. (컨트롤도 객체중 하나)
- 컨트롤러는 감시 이벤트에서 객체 수신시 객체를 검증하고 문제 발생시 웹사이트 객체에 오류 메지지를 작성한다.
- 사용자에게 오류를 자동으로 알리지 않는다.
- 웹사이트 객체에 대한 API서버를 쿼리해 오류 메세지를 확인해야 한다.
- API 서버에서 객체의 유효성 검사후 잘못된 객체를 즉시 거부하도록 할 수 있다.
- 사용자 지정 객체의 유효성 검사는 알파 기능으로 1.8에 도입됐다.
- API 서버에서 CRD 검증하려면 API 서버에서 CustomeResourceValidation 기능 게이트를 사용하도록 설정하고 CRD에서 JSON 스키마를 지정해야 한다. 

### 18.1.4 사용자 지정 객체에 사용자 지정 API 서버 제공
- 쿠버네티스에서 사용자 지정 객체의 지원을 추가하는 더 좋은 방법은 자체 API 서버를 구현하고 클라이언트가 직접 API 서버와 통신토록 하는 것이다.


#### API 서버 에그리게이션 소개
- 1.7 버전에서는 API 서버 에그리게이션을 통해 사용자 지정 API 서버를 주요 쿠퍼네티스 API서버와 통합할 수 있다. (p.745)
 
![18.5](https://dpzbhybb2pdcj.cloudfront.net/luksa/Figures/18fig05_alt.jpg  ) 

- 웹사이트 객체 처리를 담당하는 API서버를 만들수 있다.
- 코어 쿠버네티스 API 서버가 유효성을 검사하는 방식으로 개당 객체의 유효성을 검사할 수 있다.

#### 사용자 지정 API 서버 등록
- 사용자 지정 API서버를 클러스터에 추가하려면 API 서버를 팟으로 배포하고 서비스를 통해 익스포즈
- 메인 API 서버에 통합하려면 APIService 리소스를 설명하는 YAML 매니페스트를 배포하면 된다.

```
apiVersion: apiregistration.k8s.io/v1beta1
kind: APIService
metadata:
  name: v1alpha1.extensions.example.com
spec:
  group: extensions.example.com
  version: v1alpha1
  priority: 150
  service:
    name: website-api
    namespace: default
```
- APIService 리소스를 생성한 후 클라이언트 요청을 extensions.example.com API 그룹의 리소스를 포함하고 있는 주 API 서버로 전송했다.
- 버전 v1/alpha1은 website-api 서비스를 통해 노출되고 있는 사용자 정의 API 서버 팟으로 전달된다.


## 18.2 쿠버네티스 서비스 카탈로그를 이용한 쿠버네티스 확장
## 18.3 쿠버네티스 기반 플랫폼 (김동현님의 생생한 실무 이야기) 
