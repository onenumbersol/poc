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

**3. 보안 정책**

(1) 시나리오
![1]
- 동일한 네임스페이스 내에 np-test lable을 가진 Pod는 np-connetc lable을 가진 Pod에서만 접근할 수 있도록 정책 생성
- 다른 네임스페이스에서 np-test label을 가진 Pod는 ns: op라는 lable을 가진 네임스페이스에서 np-connect label을 가진 Pod에서만 접근할 수 있도록 정책 생성
- 정책 생성 시 기본적으로 허용해준 정책 이외에 모든 통신이 막히는 것 숙지

- 동일 네임스페이스 내에서 Pod 통신을 제한하는 network-policy
```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: default-policy
  namespace: dev
spec:
  podSelector:
    matchLabels:
      app: np-test
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: np-connect
  policyTypes:
    - Ingress
```

- 다른 네임스페이스 내에서 Pod 통신을 제한하는 network-policy
```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: test-namespace-policy
  namespace: dev
spec:
  podSelector:
    matchLabels:
      app: np-test
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: np-connect
          namespaceSelector:
            matchLabels:
              ns: op
  policyTypes:
    - Ingress
```

**4. 컨테이너 기본**

(1) config 파일을 통한 Pod 생성

- 기존 config를 (예시의 경우 tomcat의 server.xml) configmap으로 등록
- 해당 config를 통해 Pod가 기동될 수 있도록 마운트
- config를 마운트 할 때 디렉터리 단위로 마운트하게 되면 기존 디렉터리가 사라지므로 sub path 기능 사용

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: config-tomcat
  namespace: dev
  labels:
    app: config-tomcat
spec:
  volumes:
    - name: hello-home
      persistentVolumeClaim:
        claimName: home-pvc
    - name: log-data
      persistentVolumeClaim:
        claimName: log-pvc
    - name: server-volume
      configMap:
        name: server-xml-config
  containers:
    - name: tomcat
      image: 'custom-tomcat:latest'
      ports:
        - name: http
          containerPort: 8888
          protocol: TCP
      resources:
        limits:
          cpu: 200m
          memory: 200Mi
        requests:
          cpu: 100m
          memory: 100Mi
      volumeMounts:
        - name: hello-home
          mountPath: /usr/local/tomcat/webapps/ROOT
        - name: log-data
          mountPath: /usr/local/tomcat/logs
        - name: server-volume
          mountPath: /usr/local/tomcat/conf/server.xml
          subPath: server.xml
```

**5. CI/CD**

- CI와 CD를 구분

(1) 시나리오

![2]

- 깃랩 인스턴스 생성 (선)
- 프라이빗 레지스트리 생성 (선)
- CI/CD 테스트를 위한 apache 인스턴스 생성
- 소나큐브를 톻안 소스코드 빌드 확인
- 테스트 환경 배포 요청
- 테스트 환경 배포 확인 및 테스트 시작
- 테스트 종료 확인
- 운영 환경 배포 요청
- 운영 환경 배포 확인
- 커밋을 통한 자동 파이프라인런 생성을 위한 트리거 인스턴스 설정
- 깃랩 webhook을 통해 트리거 설정
- 커밋 이후 파이프라인 생성 확인
- 배포 확인 및 이미지 레지스트리에 이미지 저장 확인
- 이전 이미지로 롤백

**6. 어플리케이션 관리**

(1) 서비스 가용성 보장의 경우 일정 수의 파드를 삭제하여 서비스 연결에 문제가 생기는지 확인

(2) HA 구성 및 로드밸런싱의 경우 stress 테스트 혹은 컨테이너 이름을 노출해주는 web 사용

(3) 메트릭에 따른 scale in/out의 경우 hpa를 통해 확인

(4) 블루-그린 업데이트, 롤링 업데이트의 기본 개념 및 사용 방법 숙지

(5) 잡, 크론잡의 차이점과 크론잡의 사용법 숙지

(6) node-selector 혹은 node-affinity를 통한 특정 노드 스케줄링

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
  namespace: dev
spec:
  nodeSelector:
    kubernetes.io/hostname: [host-name]
  containers:
    - name: ubuntu
      image: 'ubuntu:trusty'
      command:
        - sh
        - '-c'
        - echo Hello HyperCloud! && sleep 3600

      resources:
        limits:
          cpu: 200m
        requests:
          cpu: 100m
```

**7. 백업**

(1) 백업의 경우 velero를 통해 특정 네임스페이스를 백업/삭제/복원을 통해 검증

**8. 모니터링**

(1) prometheus - grafana

- prometheus/grafana 각각의 alert 서비스가 존재하므로 이에 대해서 숙지
- grafana 대시보드 구성에 대하여 숙지

(2) EFK

- EFK의 경우 시스템 로그와 사이드카 컨테이너를 통한 어플리케이션 로그 수집 확인

- 어플리케이션 로그 수집 예
```yaml
kind: Pod
apiVersion: v1
metadata:
  name: config-tomcat
  namespace: dev
  labels:
    app: config-tomcat
spec:
  volumes:
    - name: hello-home
      persistentVolumeClaim:
        claimName: home-pvc
    - name: log-data
      persistentVolumeClaim:
        claimName: log-pvc
    - name: custom-parser
      configMap:
        name: custom-parser
    - name: fluent-bit-config
      configMap:
        name: fluent-bit-config
    - name: server-volume
      configMap:
        name: server-xml-config
  containers:
    - name: tomcat
      image: 'custom-tomcat
      ports:
        - name: http
          containerPort: 8888
          protocol: TCP
      resources:
        limits:
          cpu: 200m
          memory: 200Mi
        requests:
          cpu: 100m
          memory: 100Mi
      volumeMounts:
        - name: hello-home
          mountPath: /usr/local/tomcat/webapps/ROOT
        - name: log-data
          mountPath: /usr/local/tomcat/logs
        - name: server-volume
          mountPath: /usr/local/tomcat/conf/server.xml
          subPath: server.xml
    - name: fluent-bit
      image: 'fluent/fluent-bit:1.5-debug'
      resources: {}
      volumeMounts:
        - name: log-data
          mountPath: /tomcat-logs
        - name: custom-parser
          mountPath: /fluent-bit/etc/custom-parser.conf
          subPath: custom-parser.conf
        - name: fluent-bit-config
          mountPath: /fluent-bit/etc/fluent-bit.conf
          subPath: fluent-bit.conf
```

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: custom-parser
  namespace: dev
data:
  custom-parser.conf: |-
    [PARSER]   
        Name   custom
        Format regex
        Regex  ^(?<host>[^ ]*) [^ ]* (?<user>[^ ]*) \[(?<time>[^\]]*)\] "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<code>[^ ]*) (?<size>[^ ]*)(?: "(?<referer>[^\"]*)" "(?<agent>[^\"]*)")?$

```

```yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: fluent-bit-config
  namespace: dev
data:
  fluent-bit.conf: |-
    [SERVICE]
        Flush     5
        Daemon    off
        Log_Level debug
        Parsers_File /fluent-bit/etc/custom-parser.conf

    [INPUT]
        Name    tail
        Path    /tomcat-logs/localhost_access_log.*
        Tag     app

    [FILTER]
        Name    parser
        Match   app
        Parser  custom
        Key_name log
        Reserve_Data On
        Preserve_Key On

    [OUTPUT]
        Name  stdout
        Match *

    [OUTPUT]
        Name  es
        Match app
        Host  elasticsearch.kube-logging.svc.cluster.local
        Port  9200
        Logstash_Format True
        Logstash_Prefix app_index
```

(3) ISTIO

- fluentd가 로그를 수집하면 elastic search가 로그를 저장하고 kibana를 통해 대시보드를 제공
- kiali를 통해 마이크로 서비스 호출 관계 파악
- Jeager를 통해 로그 트레이싱 확인

**9. 멀티 클라우드**

(1) 시나리오
- 멀티클러스터 환경 생성
- 해당 환경에 네임스페이스 생성
- 해당 환경에 디플로이먼트 생성
- 해당 환경에 서비스 생성
- 해당 환경에 도메인 생성
- 도메인을 통한 Endpoint 생성 확인
- 서비스를 통해 접속 및 운영 확인

**10. kubevirt를 통한 VM 생성**

(1) 사용 yaml만 참고로 작성

```yaml
apiVersion: cdi.kubevirt.io/v1alpha1
kind: DataVolume
metadata:
  name: dv-ubuntu-1
  namespace: dev
spec:
  source:
    registry:
            url: "docker://vm/ubuntu-18.04-server-cloudimg-amd64:latest"
  pvc:
    storageClassName: hdd-ceph-block
    volumeMode: Block
    reclaimPolicy:
    - Delete
    accessModes:
    - ReadWriteMany
    resources:
      requests:
        storage: 15Gi
```

```yaml
#생성 VM 비번 Qwer12345
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  name: vm-ubuntu-1
  namespace: dev
spec:
  running: true
  template:
    metadata:
      labels:
        ubuntu-lb: lb1
    spec:
      domain:
        cpu:
          cores: 8
        devices:
          disks:
            - disk:
                bus: virtio
              name: rootdisk
            - cdrom:
                bus: sata
                readonly: true
              name: cloudinitdisk
          interfaces:
            - bridge: {}
              model: virtio
              name: default
        machine:
          type: q35
        memory:
          guest: 8Gi
        resources:
          limits:
            cpu: '8'
            memory: 8Gi
          requests:
            cpu: '8'
            memory: 8Gi
      networks:
        - name: default
          pod: {}
      volumes:
        - name: rootdisk
          persistentVolumeClaim:
            claimName: dv-ubuntu-1
        - cloudInitConfigDrive:
            userData: |
              #cloud-config
              disable_root: false
              ssh_pwauth: true
              lock_passwd: false
              users:
                - name: tmax
                  sudo: ALL=(ALL) NOPASSWD:ALL
                  passwd: $6$bLLmCtnk51$21/Fq0vSHCwDODP2hXA.wo/0k91QIw/lUy6qWPOX1vx5z0CF9Acj9vGLFlQVbjSzmh.1r7wWd0kQS9RMm51HE.
                  shell: /bin/bash
                  lock_passwd: false
          name: cloudinitdisk
```

[1]: https://github.com/onenumbersol/poc/blob/main/202011/network-policy.png
[2]: https://github.com/onenumbersol/poc/blob/main/202011/cicd.png
