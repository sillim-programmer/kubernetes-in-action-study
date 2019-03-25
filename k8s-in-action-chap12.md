`# 쿠버네티스 API 서버 보안

## 12.1 인증 이해하기
- Kubernetes API 서버는 하나 이상의 인증 플러그인으로 구성(승인 플러그인도 마찬가지)
- 기본적으로 인증 플러그인은 사용자 이름, 사용자 ID, 클라이언트가 속한 그룹을 반환

### 사용자와 그룹
#### 사용자
- 사용자는 실제 사람, Pod(Pod 내부 애플리케이션) 두가지로 나눠짐
- 사용자는 SSO(Single Sign On) 시스템과 같은 외부 시스템을 통해서 관리됨
- Pod는 Service Accounts 라는 메커니즘 사용
- 이번 12장은 주로 Service Accounts에 대해서 살펴볼 예정
#### 그룹
- 사용자 및 Service Account는 하나 이상의 그룹에 속할 수 있음
- 다른 도구들과 비슷하게 사용자 그룹 전체에 권한 부여할 때 사용
- 시스템에서 제공하는 기본 그룹의 의미
    * `system:unauthenticated`: 어느 인증 플러그인도 인증할 수 없는 클라이언트의 요청, 즉 비인증 클라이언트에게 할당되는 그룹
    * `system:authenticated`: 성공적으로 인증된 클라이언트에게 자동으로 할당되는 그룹 
    * `System:serviceaccounts`: 모든 서비스 어카운트에게 할당되어있는 그룹
    * `System:serviceaccounts:<namespace>`: 특정 네임스페이스의 모든 서비스 어카운트에게 할당되어있는 그룹 
    

### 서비스 어카운트 소개
- 서비스어카운트는 Pod, Secret, ConfigMap과 같은 리소스이며 개별 네임스페이스로 범위가 지정됨
- 기본적으로 네임스페이스별 디폴트 서비스어카운트가 자동적으로 생성되며, 서비스어카운트 설정을 안할 경우 디폴트 서비스어카운트로 설정됨.
- Pod의 컨테이너 내부에 볼륨으로 마운트된`/var/run/secrets/kubernetes.io/serviceaccount/token` 파일이 서비스어카운트 인증 토큰
- Pod에 각기 다른 서비스어카운트를 할당하여 접근 가능한 리소스 제어 가능 -> 역할 기반 접근 제어가 거의 표준 플러그인
- 서비스어카운트를 사용하는 이유는 클러스터 보안 때문 -> k8s API 서버의 침입자가 클러스터를 점유할 가능성을 줄이는것

### 서비스 어카운트 생성하기

서비스 어카운트 생성
```
kubectl create serviceaccount foo
```

서비스 어카운트 확인
```
kubectl describe sa foo
```

서비스 어카운트 시크릿 확인
```
kubectl describe secret foo-token-qenlk
```
#### UNDERSTANDING A SERVICEACCOUNT’S MOUNTABLE SECRETS ???

#### UNDERSTANDING A SERVICEACCOUNT’S IMAGE PULL SECRETS
이미지 풀 시크릿은 컨테이너 이미지를 가져오는데 필요한 자격증명을 가지고 있는 시크릿으로, 서비스어카운트에 이 시크릿을 설정할 경우에 각 파드에 개별적으로 추가할 필요없이 파드가 서비스어카운트를 통해서 이미지를 가져올 수 있게 된다.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
imagePullSecrets:
- name: my-dockerhub-secret
```

### 서비스 어카운트 할당하기
- 파드의 spec.serviceAccountName 필드에 서비스어카운트의 이름을 설정

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: curl-custom-sa
spec:
  serviceAccountName: foo
  containers:
  - name: main
    image: tutum/curl
    command: ["sleep", "9999999"]
  - name: ambassador
    image: luksa/kubectl-proxy:1.6.2
```

엠버서더를 이용하여 서비스어카운트 테스트해보기
```sh
kubectl exec -it curl-custom-sa -c main curl localhost:8001/api/v1/pods
```


## 12.2 역할 기반 접근 제어 클러스터 보안
- k8s 1.8.0 버전부터 RBAC 승인 플러그인은 GA(General Availability)로 승격하여 이제는 많은 클러스터에서 기본적으로 활성화됨
- RBAC는 권한이 없는 사용자가 클러스터의 상태를 보거나 수정할 수 없도록 함

### RBAC 인증 플러그인 소개

#### Mapping HTTP methods to authorization verbs
HTTP method | Verb for single resource | Verb for collection
---|---|---
GET, HEAD | get (and watch for watching) | list (and watch)
POST | create | n/a
PUT | update |n/a
PATCH | patch |n/a
DELETE | delete |deletecollection

전체 (쿠버네티스) 리소스 유형에 보안 권한을 적용하는 것 이외에도 `healthz`, `api`, `version` 등 리소스와 무관한 경로에도 권한 적용 가능

### RBAC 리소스 소개
- 리소스 등에 대한 권한을 부여하는 롤과 클러스터롤
- 위의 롤을 특정 사용자, 그룹, 서비스어카운트에 바인딩하는 롤바인딩과 클러스터롤바인딩

![](https://dpzbhybb2pdcj.cloudfront.net/luksa/Figures/12fig02_alt.jpg)

- 롤/롤바인딩 : 네임스페이스 수준의 리소스 권한 관리, 
- 클러스터롤/클러스터롤바인딩 : 클러스터 수준의 리소스 권한 관리

![](https://dpzbhybb2pdcj.cloudfront.net/luksa/Figures/12fig03_alt.jpg)


#### RBAC 동작확인
- :warning: GKE에서 사용자에게 관리자 권한 부여하기
```
gcloud projects add-iam-policy-binding ${PROJECT_NAME} \
  --member=user:${USER_EMAIL}   --role=roles/container.admin

kubectl create clusterrolebinding cluster-admin-binding \
  -- clusterrole=cluster-admin --user=${USER_EMAIL}
```

- foo, bar 네임스페이스를 만들고 kubectl-proxy 단일 컨테이너의 파드를 각각 네임스페이스에서 실행
```
kubectl create ns foo
kubectl run test --image=luksa/kubectl-proxy -n foo

kubectl create ns bar
kubectl run test --image=luksa/kubectl-proxy -n bar
```

- 컨테이너 내부로 들어가서 API 호출 테스트
```
kubectl exec -it ${test_POD_NAME} -n foo sh
```

```
curl localhost:8001/api/v1/namespace/foo/services
```
- 권한 없다는 에러가 발생하면 RBAC가 잘 동작하는 것

### 롤과 롤바인딩 사용하기
#### 롤 정의 
```
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: foo
  name: service-reader
rules:
- apiGroups: [""]
  verbs: ["get", "list"]
  resources: ["services"]

```

- :warning: 리소스는 복수
- 각 리소스의 규칙에 맞게 apiGroups를 설정한다. 서비스는 core apiGroup이라 `""`이라고 표기한다.
- resourceNames 필드를 이용하여 특정 서비스 인스턴스에만 권한을 정의할 수 있다.

![](https://dpzbhybb2pdcj.cloudfront.net/luksa/Figures/12fig04_alt.jpg)

- 롤 생성
```
kubectl create -f service-reader.yaml -n foo

kubectl create role servcie-reader --verb=get --verb=list --resource=services -n bar
```

#### 롤 바인딩
- 롤 바인딩 생성
```
kubectl create rolebinding test --role=service-reader --serviceaccount=foo:default -n foo
```

![](https://dpzbhybb2pdcj.cloudfront.net/luksa/Figures/12fig05_alt.jpg)


- 롤 바인딩 확인
```
kubectl get rolebinding test -n foo -o yaml
```
```
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: 2019-03-24T23:37:24Z
  name: test
  namespace: foo
  ...
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: service-reader
subjects:
- kind: ServiceAccount
  name: default
  namespace: foo
```

#### 다른 네임스페이스의 서비스 어카운트 포함하기
- 롤바인딩의 경우 다른 네임스페이스의 서비스어카운트 참조 가능
- 다른 네임스페이스에 롤바인딩이 정의된 네임스페이스의 권한 부여 가능
- bar 네임스페이스에서 foo 네임스페이스에 접근 가능하도록 롤바인딩 수정
```
kubectl edit rolebinding test -n foo
```
```
subjects:
- kind: ServiceAccount
  name: default
  namespace: foo
- kind: ServiceAccount
  name: default
  namespace: bar
```

![](https://dpzbhybb2pdcj.cloudfront.net/luksa/Figures/12fig06_alt.jpg)

- 테스트
```
k exec -it -n foo ${test_POD_NAME} curl localhost:8001/api/v1/namespaces/foo/services
```
```
{
  "kind": "ServiceList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/namespaces/foo/services",
    "resourceVersion": "7378771"
  },
  "items": []
}
```

```
k exec -it -n bar ${test_POD_NAME} curl localhost:8001/api/v1/namespaces/foo/services
```
```
{
  "kind": "ServiceList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/namespaces/foo/services",
    "resourceVersion": "7379089"
  },
  "items": []
```

### 클러스터롤과 클러스터롤바인딩 사용하기
- 노드, 퍼시스턴트볼륨, 네임스페이스 등의 리소스는 네임스페이스와 연관없는 경우가 있다.
- 특정 URI의 경우 리소스를 표현하지 않는 경우(`healthz`)가 있다.
- 위의 경우에 클러스터롤을 이용하여 권한 제어를 할 수 있다.
- 롤바인딩은 클러스터롤을 바인딩하더라도 클러스터 수준의 리소스에 대한 접근 권한을 부여하지 않는다.

  ![](https://dpzbhybb2pdcj.cloudfront.net/luksa/Figures/12fig07_alt.jpg)

- 클러스터롤을 이용하여 클러스터 수준에서 파드에 대한 접근 권한을 설정하더라도 롤바인딩으로 바인딩하면 자기 네임스페이스의 파드만 접근 가능하다. (ex, `view` 클러스터롤)

  ![](https://dpzbhybb2pdcj.cloudfront.net/luksa/Figures/12fig10_alt.jpg)

- 클러스터롤/클러스터롤바인딩의 경우 생성 방법은 롤/롤바인딩과 같다.

#### 롤, 롤바인딩, 클러스터롤, 클러스터롤바인딩 조합 요약

접근 | 롤타입 | 사용할 바인딩 타입
---|---|---
Cluster-level resources (Nodes, PersistentVolumes, ...) | 클러스터롤 | 클러스터롤바인딩
Non-resource URLs (/api, /healthz, ...) | 클러스터롤 | 클러스터롤바인딩
Namespaced resources in any namespace (and across all namespaces) | 클러스터롤 | 클러스터롤바인딩
Namespaced resources in a specific namespace (reusing the same ClusterRole in multiple namespaces) | 클러스터롤 | 롤바인딩
Namespaced resources in a specific namespace (Role must be defined in each namespace) | 롤 | 롤바인딩 

### system:discovery 클러스터롤과 클러스터롤바인딩 살펴보기
#### system:discovery - clusterrole
```
kubenetes get clusterrole system:discovery -o yaml
```
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: 2019-02-18T09:56:32Z
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:discovery
  resourceVersion: "45"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterroles/system%3Adiscovery
  uid: 7806fa0b-3363-11e9-ac9d-42010a9201be
rules:
- nonResourceURLs:
  - /api
  - /api/*
  - /apis
  - /apis/*
  - /healthz
  - /openapi
  - /openapi/*
  - /swagger-2.0.0.pb-v1
  - /swagger.json
  - /swaggerapi
  - /swaggerapi/*
  - /version
  - /version/
  verbs:
  - get
```
- 리소스가 아닌 URL들에 대한 권한을 부여하고 있다.

#### system:discovery - clusterrolebinding
```
k get clusterrolebinding system:discovery -o yaml
```
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: 2019-02-18T09:56:32Z
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:discovery
  resourceVersion: "99"
  selfLink: /apis/rbac.authorization.k8s.io/v1/clusterrolebindings/system%3Adiscovery
  uid: 78552d89-3363-11e9-ac9d-42010a9201be
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:discovery
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:authenticated
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:unauthenticated
```
- subejects를 보면 인증되거나 인증 되지 않은 사용자 모두에게 권한이 부여됨을 알 수 있다.

### 디폴트 클러스터롤과 클러스터롤바인딩 이해하기
#### view 클러스터롤
- 롤, 롤바인딩, 시크릿을 제외한 네임스페이스 내의 대부분의 리소스 읽기 권한
- 시크릿을 볼 수 없는 이유는 기존보더 더 큰 권한의 시크릿을 읽어 권한 상승을 방지하기 위해서이다.
#### edit 클러스터롤
- 네임스페이스의 리소스 뿐만 아니라 시크릿을 읽고 수정 가능한 권한
- 권한 상승을 방지하기 위해 롤 또는 롤바인딩을 보거나 수정하는 것은 허용되지 않는다.

#### admin 클러스터롤
- ResourceQuotas가 이 롤을 가진 주체
- 네임스페이스 자체를 제외한 네임스페이스 내의 리소스를 읽고 수정 가능하며, 롤과 롤바인딩 또한 수정 가능한 권한

#### cluster-admin 클러스터롤 
- 클러스터의 모든 권한을 가지고 있다

### 인증 권한을 현명하게 부여하기
- 최소 권한 원칙
- 각 파드에 특정 서비스어카운트를 작성하고 롤/롤바인딩을 통해 맞춤형롤 제공

## 12.3 요약
- API 서버 클라이언트는 사용자, 파드에서 실행중인 앱
- 파드는 서비스어카운트를 가지고 있다
- 사용자와 서비스 어카운트는 그룹에 속한다.
- 롤, 클러스터롤은 어떤 리소스 또는 URI에 대해 수행할 수 있는 작업에 대해 정의한다.
- 롤바인딩, 클러스터바인딩은 롤과 클러스터롤을 그룹 및 서비스어카운트에 바인딩한다.
- 각 클러스터에는 기본 클러스터와 클러스터롤바인딩이 있다.
`
