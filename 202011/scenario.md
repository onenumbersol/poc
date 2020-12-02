**0. 환경**

PoC 진행 환경에 대한  물리 구성, 소프트웨어 구성 설명이 우선시 되어야 함.

(1) HyperCloud 4.1, Kube 17.6, ProLinux 7.7
- cri-o, calico, ceph

(2) 3 master node, 16 worker node, 2 GPU node, 6 Storage node


**1. HyperCloud 소개**

(1) admin으로 로그인 이후 간단한 메뉴 소개

- 홈 : 로그인시 첫 화면, 클러스터 상태, 클러스터 로그, 모니터링 메뉴
- 서비스 카탈로그 : 미리 정의 된 템플릿을 사용할 수 있는 메뉴
- 워크로드 : Kubernetes의 기본적인 구성 요소를 조회 및 관리할 수 있는 메뉴
- 서비스 메시 : Istio를 사용하고 관리할 수 있는 메뉴
- 네트워크 : Kubernetes에 올라간 리소스를 외부로 노출시키는 서비스, 인그레스를 관리하는 메뉴
- 스토리지 : PV, PVC를 관리하는 메뉴
- CI/CD : CI/CD를 관리하는 메뉴
- AI DevOps : 머신러닝을 위해 Kubeflow를 사용하고 관리할 수 있는 메뉴
- 보안 : 네트워크 폴리시, 파드 보안 정책을 관리할 수 있는 메뉴
- 이미지 : 프라이빗 레지스트리를 생성하고 관리할 수 있는 메뉴
- 매니지먼트 : 네임스페이스, 네임스페이스에 할당된 리소스를 관리할 수 있는 메뉴
- 호스트 : 클러스터를 구성하고 있는 노드들의 상태와 현황을 조회할 수 있는 메뉴
- 인증/인가 : 롤, 롤 바인딩, 유저, 서비스 어카운트를 관리할 수 있는 메뉴

(2) 홈 화면 소개

- 좌측의 홈 메뉴에서 조회할 수 있는 상태, 감사로그, 이벤트 로그 소개

**2. 권한 관리**

(1) 시나리오
- HyperCloud 홈페이지에서 회원 가입
- HyperAuth를 통한 가입 확인
- 신규 유저로 로그인 이후 네임스페이스 클레임을 통해 네임스페이스 생성/리소스 할당 요청
- 관리자가 네임스페이스 클레임 승인
- 네임스페이스의 생성과 네임스페이스 리소스 쿼터 확인
- 신규 네임스페이스에 대한 롤을 인증/인가
- 롤 바인딩을 통해 부여
- 다른 네임스페이스의 특정 리소스에 접근할 수 있는 롤을 생성
- 롤 바인딩을 통해 부여 및 확인

(2) 사용되는 yaml

- deployment를 조회할 수 있는 롤
```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: new-role
  namespace: dev
rules:
  - verbs:
      - '*'
    apiGroups:
      - events.k8s.io/v1beta1
    resources:
      - '*'
  - verbs:
      - '*'
    apiGroups:
      - ''
    resources:
      - '*'
---> 아래 부분 수정, 삭제를 통해 확인
  - verbs:                 
      - '*'
    apiGroups:
      - apps
    resources:
      - deployments
```

- 네임스페이스 리소스 쿼터
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
  namespace: dev
spec:
  containers:
    - name: ubuntu
      image: 'ubuntu:trusty'
      command:
        - sh
        - '-c'
        - echo Hello HyperCloud! && sleep 3600
      resources:
        limits:
          cpu: 6
          memory: 1Gi
        requests:
          cpu: 1
          memory: 1Gi
```

