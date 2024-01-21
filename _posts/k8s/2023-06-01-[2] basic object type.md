---
title: [2] basic object type
author: mouseeye
date: 2023-06-01 21:00:00 +0900
categories: [pod, replica set, ]
tags: [k8s, kubectl, pod, replica set]
---

## Pod
### 기본 개념
- 여러개의 컨테이너 묶음을 하나의 추상화된 하나의 애플리케이션으로 동작
  - 1개의 주요 app 컨테이너와 app 컨테이너의 부가적인 동작을 돕는 sidecar 컨테이너의 조합으로 사용 가능
- 파드 내 컨테이너들은 리눅스 네임스페이스를 공유
  - 즉 네트워크 네임스페이스도 공유된다. (docker engine에서 `container network` 타입이라고 생각하면 쉬움)
  - 각 pod마다 pause라는 컨테이너가 생성되며, 이 컨테이너에 의해 pod내 컨테이너들이 리눅스 네임스페이스를 공유함

### kubectl 명령어
  - Pod 조회 : `kubectl get pods` / `kubectl get pods --show-labels` (라벨과 함께 표시) / `kubectl get pods -l app:nginx-pod` (특정 라벨을 가지는 파드 조회)
  - Pod 상세 정보 확인 : `kubectl describe pods [pod name]`
  - Pod 내부에 명령어 전달 : `kubectl exec -it [pod name] -c [container name] bash`
  - Pod 로그 확인 : `kubectl logs [pod name]`
  - Pod 삭제 : `kubectl delete pods [pod name]`
  - pod 정의 수정 : `kubectl edit pods [pod name]`

### yaml 파일 정의
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx-pod
spec:
  containers:
    - name: my-nginx-container
      image: nginx:latest
      ports:
      - containerPort: 80
        protocol: TCP

    - name: ubuntu-sidecar-container
      image: ubuntu:latest
      command: ["tail"]
      args: ["-f", "/dev/null"]
```

## Replica Set
### 기본 개념
- 일정 개수의 파드를 유지하는 컨트롤러
- 노드 장애 등의 이유로 파드를 사용할 수 없다면 다른 노드에서 파드를 다시 생성
- 라벨 셀렉터에 의해서 파드와 느슨하게 연결이 되고 일정한 갯수를 유지한다.
  - 라벨 : k8s 내에서 서로 다른 오브젝트가 서로를 찾아야 할 때 사용 / 또는 오브젝트의 부가적인 설명

### kubectl 명령어
- 레프리카셋 조회 : `kubectl get rs`
- 레프리카셋 상세 정보 확인 : `kubectl describe rs`
- 레프리카셋 삭제 : `kubectl delete rs -f [rs 이름]` (삭제 시 당연하게도 파드도 같이 삭제됨)
- 레프리카셋 정의 수정 : `kubectl edit rs [rs 이름]` (파드 갯수 수정)

### yaml 파일 정의
```yaml
appVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replicaset-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-nginx-label
## 여기까지 replica set 정의
  template:
    metadata:
      name: my-nginx-pod
      labels:
        app: my-nginx-label
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
## pod template 정의
```
- spec.replicas : 유지할 파드 갯수
- spec.selector.matchLabels : 유지할 파드의 라벨
- spec.template : 유지할 파드의 스펙

## 디플로이먼트(Deployment)
### 기본 개념
- 레플리카셋, 파드의 배포 관리
  - 레플리카셋의 변경 사항을 저장하는 revision을 남겨 롤백을 가능하게 함
  - 무중단 서비스를 위해 롤링 업데이트 전략 지정 가능
- 여러개의 레플리카셋을 관리하는 상위 오브젝트
- 디플로이먼트를 생성하면 레플리카셋, 파드도 함께 생성된다.
- 반대로, 디플로이먼틀 삭제하면 레플리카셋, 파드도 함께 삭제된다.

### kubectl 명령어
- 디플로이먼트 조회 : `kubectl get deploy`
- 디플로이먼트 상세 정보 확인 : `kubectl describe deploy`
- 디플로이먼트 삭제 : `kubectl delete deploy -f [deploy 이름]` (삭제 시 당연하게도 파드도 같이 삭제됨)
- 디플로이먼트 정의 수정 : `kubectl deploy rs [deploy 이름]` (파드 갯수 수정)
- 디플로이먼트 배포(배포 명령어도 함께 저장) : `kubectl apply -f [yaml 파일] --record`
- 배포 히스토리 확인 : `kubectl rollout history deployment [deploy 이름]`
- 특정 리비전으로 롤백 : `kubectl rollout undo deployment [deploy 이름] --to-revision=1`

### yaml 파일 정의
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx-deployment
spec:
  replicas: 3 #replics 정의
  selector:
    matchLabels:
      app: my-nginx
  template:
    # pod 템플릿 정의
    metadata:
      name: my-nginx-pod
      labels:
        app: my-nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.12
          ports:
            - containerPort: 80
```

## 서비스(Service)
### 기본 개념
- 포드를 연결하고 외부에 노추ㄹ
