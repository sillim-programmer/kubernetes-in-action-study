# 볼륨

쿠버네티스는 실제 데이터가 있는 디렉토리를 보존하기 위해서 저장소 볼륨을 정의한다.  
볼륨은 포드의 일부로 정의되며 포드의 라이프사이클이 같다.  
이는 포드가 시작될 때 볼륨이 작성되고 포드가 삭제될 때 볼륨이 삭제됨을 의미한다.

- 볼륨은 포드의 컴포넌트이므로 컨테이너와 마찬가지로 포드의 스펙에 정의한다
- 포드의 모든 컨테이너에서 볼륨을 사용할 수 있지만 볼륨에 액세스해야 하는 각 컨테이너에 볼륨을 마운트 해야 한다.
  (각 컨테이너에서 파일 시스템의 임의 위치에 볼륨을 마운트 할 수 있다.)

## 볼륨 종류  
- emptyDir : 일시적인 데이터를 저장하는데 사용되는 비어있는 단순한 디렉토리
- hostPath : 워커노드의 파일 시스템에서 포드로 디렉토리를 마운트하는데 사용
- gitRepo :  git 스토리지의 내용을 체크아웃해 초기화된 볼륨
- nfs : 포드에 마운트된 NFS 공유
- gcePersistentDisk(구글 컴퓨트 엔진 영구 디스크), awsElastic-BlockStore(AWS 탄력적 블록 스토리지 볼륨), azureDisk(마이크로소프트 애저 디스크 볼륨)  
: 클라우드에서 제공하는 전용 스토리지
- cinder, cephfs, iscsi, flocker, glusterfs, quobyte, rbd, flexVolume, vsphere-Volume, photonPersistentDisk, scaleIO : 다른 유형의 네트워크 스토리지를 마운트
- configMap, secret, downwardAPI : 특정 쿠버네티스 리소스 및 클러스터 정보를 포드에 노출하는 데 사용되는 특수한 유형의 볼륨
- persistentVolumeClaim : 사전 또는 동적으로 프로비저닝된 영구 스토리지를 사용하는 방법

### 1. emptyDir 볼륨 - 포드안에 존재
emptyDir 볼륨은 빈 디렉토리로 시작  
볼륨의 수명이 포드와 연관되어 있기 때문에 포드를 삭제하면 볼륨의 내용이 손실된다.  
emptyDir 볼륨은 동일한 포드에서 실행중인 컨테이너 같에 파일을 공유할 때 유용하다.  
읽기/쓰기의 권한을 부여 가능하다

● 포드 생성
fortune-pod.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: fortune
spec:
  containers:
  - image: luksa/fortune
    name: html-generator
    volumeMounts:
    - name: html
      mountPath: /var/htdocs
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    emptyDir: {} 
```
```
$ kubectl create -f fortune-pod.yaml

# fortune 메시즈를 보려면 포드의 액세스를 활성화해야 한다.
# 이 작업은 포트를 로컬 컴퓨터에서 포드로 전달을 수행한다
$ kubectl port-forward fortune 8080:80  

# 이제 로컬 컴퓨터의 포트 8080을 통해 Nginx 서버에 액세스 할수 있다.
$ curl http://localhost:8080
```
html-generator 컨테이너가 시작되면 10초마다 /var/htdocs/index.html 파일에 fortune 명령의 출력 결과를 기록한다.  
볼륨이 /var/htdocs에 마운트되므로 컨테이너의 최상위 계층 대신 index.html 파일이 볼륨에 기록된다.  
Nginx는  fortune 루프를 실행하는 컨테이너가 작상한 index.html파일을 제공한다.  
결과적으로 포트 80의 포드로 http 요청을 보내는 클라이언트는 응답으로 현재 fortune  메시지를 받는다.  

디스크 대신 메모리에 emptDir을 생성할 수 있다.  
아래와 같이 매체를 Memory로 설정한다.
```
volumes:
  -name:html
   emptyDir:
     medium: Memory
```

### 2. git repository 볼륨 - 포드안에 존재
gitRepo 볼륨은 깃 저장소에서 데이터를 복제하여 포드의 볼륨에 채워진다.  
gitRepo 볼륨이 생성된 후에는 참조하는  repo와 동기화 되지 않는다.  
포드를 새로 만들어야 한다.
gitrepo-volume-pod.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: gitrepo-volume-pod
spec:
  containers:
  - image: nginx:alpine
    name: web-server
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
      readOnly: true
    ports:
    - containerPort: 80
      protocol: TCP
  volumes:
  - name: html
    gitRepo:
      repository: https://github.com/haksup/kubia-website-example.git
      revision: master
      directory: .
```
```
$ kubectl create -f gitrepo-volume-pod.yaml

$ kubectl port-forward gitrepo-volume-pod 8080:80  
```
gitRepo 원본 주소  
https://github.com/luksa/kubia-website-example.git
   
### 3. hostPath - 워커노드 파일 시스템에 액세스 p226
hostPath 볼륨은 노드의 파일 시스템에 있는 측정 파일 또는 디렉토리를 가리킨다.  
hostPath 는 영구 스토리지 유형이다  
볼륨의 내용은 특정 노드의 파일 시스템에 저장되므로 포드가 다른 노드로 다시 실행되면 데이터가 나오지 않는다.  
즉 노드안에서만 포드와 볼륨이 공유된다.
```
$ kubectl get pod --namespace kube-system
``` 
host-path-pod.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: k8s.gcr.io/test-webserver
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      # directory location on host
      path: /data
      # this field is optional
      type: Directory
```
```
$ kubectl create -f host-path-pod.yaml
```

### 4-1. GCE 영구 디스트 - 모든 클러스터 노드에서 액세스 p268
GCE(구글 컴퓨터 엔진) 에서 클러스터 노드를 실행한다.

쿠버네티스 클러스터와 동일한 영역에서 생성해야 한다.  
쿠버네티스 클러스터를 나열하는 명령은 아래와 같다
```
$ gcloud container clusters list

# GCE 영구 디스크 생성
$ gcloud compute disks create --size=1Gib --zone=europe-west1-b mongodb
```

### 4-2. gcePersistentDisk 를 사용해 포드 생성
물리적 스토리지를 볼륨을 몽고 DB 포드내부에 사용할 수 있다.
```
apiVersion: v1
kind: Pod
metadata:
  name; mongodb
spec:
  volumes:
    - name: mongodb-data
      gcePersistentDisk:
        pdName: mongodb
        fsType: ext4
    containers:
    - image: mongo
      name: mongodb
      volumeMounts:
      - name: mongodb-data
        mountPath: /data/db
      ports:
      - containerPort: 27017
        protocol: TCP
```

### 5 awsElasticBlockStore 볼륨
aws의 영구 디스크
```
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  volumes:
  - name: mongodb-data
    awsElasticBlockStore:
      volumeId: my-volume
      fsType: ext4
    containers:
    - image: mongo
      name: mongodb
      volumeMounts:
      - name: mongodb-data
        mountPath: /data/db
      ports:
      - containerPort: 27017
        protocol: TCP
```

### 6. NFS 볼륨 사용
NFS(네트워크 파일시스템) 서버로 볼륨을 마운트 할 경우
```
apiVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  volumes:
  - name: mongodb-data
    nfs:
      server: 1.2.3.4
      path: /some/path
```
https://kubernetes.io/docs/concepts/storage/volumes/#nfs 참고

### 7. 이외에도 다양한 스토리지 기술이 존재한다
https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes

### 8. 기본 스토리지 기술에서 포드 분리
지금까지 모든 영구 볼륨 유형은 포드 개발자가 클러스터에서 사용할 수 있는 실제 네트워크 스토리지 인프라 지식을 갖추고 있어야 한다.  
이상적인 것은 쿠버네트스에 애플리케이션을 배포하는 개발자는 포드를 실행하는 데 사용되는 물리적 서버 유형을 알 수 없는 것과   
마찬가지로 사용되는 스토리지 기술의 종류를 몰라도 상관없도록 제공하도록 한다

### 9. persistentVolume과 persistentVolumeClaim
우리가 파일을 업로드 할때 어떤 물리 저장소에 어떤 경로에 저장해줘~ 라고 정의하면서 파일을 업로드 하지 않는다.  
우리는 파일을 업로드 하면 알아서 정의된 위치에 파일이 업로드가 된다.  
지금까지 pod에 세부적으로 볼륨을 정의해서 만들었지만 이제 그런 정의 없이 볼륨을 마운트 한다고 생각하면 된다.

1. 개발자가 포드에 기술적으로 세부적인 볼륨을 추가하는 대신 클러스터 관리자가 기본 스토리지를 설정한 다음 PersistentVolume 리소스를 생성해 쿠버네티스 API 서버에 등록한다.  
2. 클러스터 사용자가 포드 중 하나에서 영구 스토리지를 사용해야 하는 경우 먼저 필요한 최소 크기와 액세스 모드를 지정해 PersistentVolumeClaim 매니페스트를 생성한다.  
3. 사용자가 PersistentVolumeClaim 매니페스트를 쿠버네티스 API 서버에 제출하고 쿠버네티스가 적절한 영구 볼륨을 찾아 클레임을 바인딩한다.  

### 10-1. PersistentVolume 생성 p276
mongodb-pv-gcepd.yaml
```
apiversion: v1
kind: PersistentVolume
metadata:
  name: mongodb-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  - ReadOnlyMany
  PersistentVolumeReclaimPolicy: Retain
  gcePersistentDisk:
    pdName: mongodb
    fsType: ext4
``` 
capacity: storage: 1Gi => PersistentVolume 크기 정의
accessModes:- ReadWriteOnce- ReadOnlyMany => 단일 클라이언트가 읽기 및 쓰기용이나 여러 클라이언트가 읽기 전용으로 마안트 할수 있다
PersistentVolumeReclaimPolicy: Retain => 클레임이 해제된 후에는 PersistentVolume 을 삭제하거나 삭제된 상태로 유지해야 한다.
gcePersistentDisk:pdName: mongodb fsType: ext4 => PersistentVolume을 GCE 영구 디스크를 정의

```
$ kubectl create -f mongodb-pv-gcepd.yaml
```
kubectl create 명령을 사용해 PersistentVolume을 생성 후 할당할 준비가 되었는지 확인한다
```
$ kubectl get pv
```

● persistentVolume 은 네임스페이스에 속하지 않는다.
 
### 10-2.  PersistentVolumeClaim을 생성해 PersistentVolume 할당 p279
PersistentVolume을 포드에서 직접 사용할 수 없다 우선 그것을 클레임 해야 한다.
mongodb-pvc.yaml  
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  resources:
    requests:
      storage: 1Gi
  accessModes:
  - ReadWriteOnce
  storageClassName: ""
```
metadata: name: mongodb-pvc => 클레임의 네임 포드 볼륨으로 사용할 때 필요하다
requests:storage: 1Gi => 스토리지의 1Gb 요청
accessModes:- ReadWriteOnce => 스토리지가 단일 클라이언트를 지원(읽기 쓰기)
storageClassName => 동적 프로비저닝 

```
$ kubectl create -f mongodb-pvc.yaml  
$ kubectl get pvc
```

### 10-3 포드에서 PersistentVolumeClaim 사용
사용자가 볼륨을 해제할 때까지 다른 사용자는 동일한 볼륨을 할당할수 없다
mongodb-pod-pvc.yaml
```
apivVersion: v1
kind: Pod
metadata:
  name: mongodb
spec:
  containers:
  - image: mongo
    name: mongodb
    volumeMounts:
    - name: mongodb-data
      mountPath: /data/db
    ports:
    - containerPort: 27017
      protocol: TCP
  volumes:
  - name: monodb-data
    persistentVolumeClaim:
      claimName: mongodb-pvc
```
```
$ kubectl create -f mongodb-pod-pvc.yaml  

$ kubectl exec -it mondofb mongo # 몽고DB 셀을 실행한다
```
