# 3. 네트워크를 통해 서비스에 Pod 연결하기
- Pod 끼리는 서로 통신할 수 있다.
  - Kubernetes 에서 TCP, UDP 지원
  - Pod 의 IP 주소는 교체될때 변경되는 문제 --> Service 에서 Address Discovery 기능을 제공하여 해결
- Service: 통신 트래픽의 라우팅을 담당, Pod의 외부<->내부 모두 포함

## 3.1. Kubernetes 내부의 Network traffic routing
- Pod 특징
  - Kubenetes에서 부여한 IP를 가진 가상환경
  - Controller에 의해 Lifecycle을 관리 받는 "쓰고 버리는" 리소스
- 문제 - 새로운 Pod의로 교체될때 IP 주소가 변경되고 이를 찾기가 어렵다.
- 가상 네트워크는 cluster 전체에 적용 - IP 주소만 있으면 통신 가능
- Kubenetes cluster에 전용 DNS 서버 존재 - 서비스 이름과 IP 주소를 대응 시켜줌
- Serive에서 자신만의 IP주소를 가지고 DNS에 등록(삭제될때 까지 바뀌지 않음), Pod은 label selector를 이용하여 느슨한 연결
- 실습 - Pod 끼리 서로의 IP 주소를 알아내기
  - 두개의 Deployment를 생성
    ```bash
    kubectl apply -f sleep/sleep1.yaml -f sleep/sleep2.yaml
    # deployment.apps/sleep-1 created
    # deployment.apps/sleep-2 created
    ```
  - Pod가 완전히 시작될 때까지 기다린다.
    ```bash
    kubectl wait --for=condition=Ready pod -l app=sleep-1
    # pod/sleep-1-5bfcffdc46-vh8fd condition met
    kubectl wait --for=condition=Ready pod -l app=sleep-2
    # pod/sleep-2-594d57479-wmkmd condition met
    ```
  - 두 번째 Pod의 IP 주소를 확인한다.
    ```bash
    kubectl get pod -l app=sleep-2 --output jsonpath='{.items[0].status.podIP}'
    # 10.xx.xx.xx%
    ```
  - 두 번째 Pod에서 첫 번째 Pod으로 ping을 보낸다.
    ```bash
    kubectl exec deploy/sleep-1 -- ping -c 2 $(kubectl get pod -l app=sleep-2 --output jsonpath='{.items[0].status.podIP}')
    :'
    PING 10.xx.xx.xx (10.xx.xx.xx): 56 data bytes
    64 bytes from 10.xx.xx.xx: seq=0 ttl=64 time=0.362 ms
    64 bytes from 10.xx.xx.xx: seq=1 ttl=64 time=0.079 ms

    --- 10.xx.xx.xx ping statistics ---
    2 packets transmitted, 2 packets received, 0% packet loss
    round-trip min/avg/max = 0.079/0.220/0.362 ms
    '
    # ping -c 2 ==> (-c <count>         stop after <count> replies)
    ```
- 실습2 - Pod 새로 뜰때 IP 변화 관찰
  - 현재 Pod의 IP 확인
    ```bash
    kubectl get pod -l app=sleep-2 --output jsonpath='{.items[0].status.podIP}'
    # 10.42.0.60%
    ```
  - Pod 삭제
    ```bash
    kubectl delete pods -l app=sleep-2
    # pod "sleep-2-594d57479-wmkmd" deleted
    ```
  - 새로운 Pod의 IP 확인
    ```bash
    kubectl get pod -l app=sleep-2 --output jsonpath='{.items[0].status.podIP}'
    # 10.42.0.62%
    ```
- 실습3 - Service 생성
  - Service의 yaml 정의
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: sleep-2       # 서비스 이름이 도메인 네임
    spec:
      selector:
        app: sleep-2      # app label의 값이 sleep-2인 모든 파드가 대상
      ports:
        - port: 80        # 80번 포트를 주시하다가 파드의 80번 포트로 트래픽을 전달
    ```
  - Service 배포
    ```bash
    kubectl apply -f sleep/sleep2-service.yaml
    # service/sleep-2 created
    ```
  - Service 상세 정보
    ```bash
    kubectl get svc sleep-2
    :'
    NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
    sleep-2   ClusterIP   10.43.44.128   <none>        80/TCP    12s
    '
    ```
  - Pod 와 통신이 잘 되는지 확인 - 실패해야함
    ```bash
    kubectl exec deploy/sleep-1 -- ping -c 1 sleep-2
    :'
    PING sleep-2 (10.43.44.128): 56 data bytes

    --- sleep-2 ping statistics ---
    1 packets transmitted, 0 packets received, 100% packet loss
    command terminated with exit code 1
    '
    # ping 명령이 Kubernetes Service에서 지원하지 않는 ICMP 프로토콜을 사용하기 때문
    ```

## 3.2. Pod와 Pod 간 통신
- ClusterIP
  - 클러스터내에서 통용되는 IP 주소를 생성
  - Pod이 어느 노드에서도 이 IP 주소로 접근 가능
  - 즉, Pod to Pod 통신에서만 쓰인다.
  - 내부 접근은 허용하되 외부 접근은 차단해야 하는 분산 시스템의 컴포넌트에 적합한 구조
- 실습1 - Web app, API 역할 각각 Deployment 실행
  - 각각 Deployment 실행
    ```bash
    kubectl apply -f numbers/api.yaml -f numbers/web.yaml
    # deployment.apps/numbers-api created
    # deployment.apps/numbers-web created
    ```
  - Pod의 준비가 끝날 때까지 기다린다.
    ```bash
    kubectl wait --for=condition=Ready pod -l app=numbers-web
    # pod/numbers-web-f456bfcbf-zrs77 condition met
    ```
  - Web 앱에 port-forward 적용
    ```bash
    kubectl port-forward deploy/numbers-web 8080:80
    :'
    Forwarding from 127.0.0.1:8080 -> 80
    Forwarding from [::1]:8080 -> 80
    Handling connection for 8080
    Handling connection for 8080
    Handling connection for 8080
    Handling connection for 8080
    Handling connection for 8080
    '
    ```
  - http://localhost:8080
    - Go를 누르면 오류가 발생하는것을 확인
    - Service는 생성하지 않았기 때문에 서로 통신할 수 없다.
  - port-forward 중단
    - ctrl-c
- 실습2 - Service를 배포
  - Service 정의 - api-service.yaml
    ```yaml
    apiVersion: v1
    kind: Service
    
    metadata:
      name: numbers-api
    spec:
      ports:
        - port: 80
      selector:
        app: numbers-api
      type: ClusterIP         # 기본 유형이 ClusterIP 이므로 생략 할 수 있다. 하지만 의미를 분명히 하기 위해 명시
    ```
  - Service 배포
    ```bash
    kubectl apply -f numbers/api-service.yaml
    # service/numbers-api created
    ```
  - Service 상세 정보 1
    ```bash
    kubectl get service/numbers-api
    :'
    NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
    numbers-api   ClusterIP   10.43.119.107   <none>        80/TCP    79m
    '
    ```
  - Service 상세 정보 2
    ```bash
    kubectl get svc numbers-api
    :'
    NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
    numbers-api   ClusterIP   10.43.119.107   <none>        80/TCP    80m
    '
    ```
  - port-forward 적용
    ```bash
    kubectl port-forward deploy/numbers-web 8080:80
    :'
    Forwarding from 127.0.0.1:8080 -> 80
    Forwarding from [::1]:8080 -> 80
    Handling connection for 8080
    Handling connection for 8080
    ^Z
    [1]  + 19466 suspended  kubectl port-forward deploy/numbers-web 8080:80
    '
    ```
  - http://localhost:8080
    - Go를 누르면 오류 없이 잘 실행 됨
  - port-forward 중단
    - ctrl-c
- 실습3 - Pod 삭제 후 관찰
  - 현재 Pod 의 이름과 IP 주소 확인
    ```bash
    kubectl get pod -l app=numbers-api -o custom-columns=NAME:metadata.name,POD_IP:status.podIP
    :'
    NAME                           POD_IP
    numbers-api-76d585d955-8sbj6   10.xx.xx.77
    '
    ```
  - Pod 수동 삭제(실제 운영상으로는 늘상 저절로 일어날 수 있는 일)
    ```bash
    kubectl delete pod -l app=numbers-api
    # pod "numbers-api-76d585d955-8sbj6" deleted
    ```
  - 새로 생성된 Pod 의 이름과 IP 주소 확인
    ```bash
    kubectl get pod -l app=numbers-api -o custom-columns=NAME:metadata.name,POD_IP:status.podIP
    :'
    NAME                           POD_IP
    numbers-api-76d585d955-q6f28   10.xx.xx.79
    '
    ```
  - port-forward 적용
    ```bash
    kubectl port-forward deploy/numbers-web 8080:80
    :'
    Forwarding from 127.0.0.1:8080 -> 80
    Forwarding from [::1]:8080 -> 80
    Handling connection for 8080
    Handling connection for 8080
    ^Z
    [1]  + 21888 suspended  kubectl port-forward deploy/numbers-web 8080:80
    '
    ```
  - http://localhost:8080
    - Go를 누르면 오류 없이 잘 실행 됨
  - port-forward 중단
    - ctrl-c
      - npx kill-port 8080
      - 이 필요할 수도 있음

## 3.3. 외부 트래픽을 Pod로 전달하기
- 여러가지 방법 중
  - LoadBalancer - 간단하고 유연한 방법
  - NodePort - 

### 3.3.1. LoadBalancer
- 외부 로드벨런서 --> 로드벨런서 서비스(Cluster 전체) --> Pod
  - 외부 로드벨런서와 함께 동작
  - label selector 와 일치하는 Pod 으로 전달
- Pod이 많아지면 노드를 선택해야 하는 까다로울 수 있는 문제를 Kubernetes가 해결해준다.
- 실습 1 - LoadBalancer 서비스를 Cluster에 배치하고, IP 주소를 찾아라
  - LoadBalancer yaml 정의
    ```yaml
    apiVersion: v1
    kind: Service
    
    metadata:
      name: numbers-web
    
    spec:
      ports:
        - port: 8080
          targetPort: 80
      selector:
        app: numbers-web
      type: LoadBalancer
    ```
  - 로드밸런서 서비스를 배치
    ```bash
    kubectl apply -f numbers/web-service.yaml
    # service/numbers-web created
    ```
  - 서비스의 상세 정보를 확인
    ```bash
    kubectl get svc numbers-web
    ## 
    :'
    NAME          TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)          AGE
    numbers-web   LoadBalancer   10.xx.xx.xx   192.168.xx.xx   8080:32219/TCP   57s
    '
    # CLUSTER-IP: 로드벨런서 서비스는 CLUSTER-IP가 부여됨. 실제 공인 IP를 부여받는다.
    # EXTERNAL-IP: 이 외부 주소로 들어오는 트래픽을 Cluster로 전달됨. Cluster에서 제공되는 주소
    ```
  - app의 URL을 EXTERMAL-IP 필트로 출력
    ```bash
    kubectl get svc numbers-web -o jsonpath='http://{.status.loadBalancer.ingress[0].*}:8080'
    # http://192.168.xx.xx VIP:8080%
    # ingress: 입구, Kubernetes에서는 traffic을 다른 Backend에 전달하는 것을 뜻함
    ```
- local이나 K3s 에서 여러개의 LoadBalancer service를 사용하려면 이들 port를 다르게 지정

### 3.3.2. NodePort
- 외부 로드밸런서가 필요 없다.
- 노드1, 노드2 ... 노트n번 --> NodePort service --> Pod
- 단점
  - 서비스에서 설정된 포트가 모든 노드에서 개방되어 있어야 함
    - LoadBalancer service 보다 유연하지 못한 점
  - 다중 노드 클러스터에서 로드밸런싱 효과를 얻을 수 없다.
  - 분산 수준 조금 더 좋지 않다.
  - 클러스터 환경에 따라 동일하게 동작하지 않음
  - manifest 가 일관될 수 없어서 실제로 사용할 일이 별로 없다.

- 예제1
  - nodePort 정의 - web-service-nodePort.yaml
    ```yaml
    apiVersion: v1
    kind: Service
    
    metadata:
      name: numbers-web-node
    
    spec:
      ports:
        - port: 8080                  # 다른 Pod에서 이 Service에 접근하기 위해 사용하는 Port 
          targetPort: 80              # 대상 Pod에 트래픽을 전달하는 Port
          nodePort: 30080             # Service가 외부에 공개되는 Port
      selector:
        app: numbers-web
      type: NodePort                  # 노드의 IP 주소를 통해 접근 가능한 Service
    ```

## 3.4. Kubernetes 클러스터 외부로 트래픽 전달하기
- Cluster 외부와 연동
 
### 3.4.1. ExternalName 서비스 사용
- 외부 도메인에 대한 별명 달기
- 앱 내에서는 로컬 네임을 사용 - Kubenetes DNS 서버가 외부 도메인으로 변환해주는 방식
- 앱 설정에 포함하기 어려운 환경 간 차이를 반영할때 유용
  - Database name 대신 지정된 문자열 사용
- 단점
  - 앱이 사용하는 주소가 가리키는 대상을 치환할 뿐 요청의 내용 자체를 바꾸어 주지 못한다.
    - TCP 프로토콜은 문제 없음
    - HTTP 서비스에는 문제
      - HTTP 요청 header에 대상 hostname이 들어가는데 ExternalName service의 응답과 다르다면 요청이 실패
      - 해결위해 작접 header의 hostname을 수정해야 함
- 실습 1 - 기존 ClusterIP Service 삭제하고 ExternalName Service 로 대체한다.
  - 기존 ClusterIP Service 확인
    ```bash
    kubectl get svc
    :'
    NAME          TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)          AGE
    kubernetes    ClusterIP      10.43.0.1       <none>         443/TCP          15d
    numbers-api   ClusterIP      10.43.119.107   <none>         80/TCP           2d23h
    numbers-web   LoadBalancer   10.43.242.13    192.168.5.15   8080:32219/TCP   47h
    sleep-2       ClusterIP      10.43.44.128    <none>         80/TCP           5d1h
    '
    ```
  - numbers-api ClusterIP Service 삭제
    ```bash
    kubectl delete svc numbers-api
    # service "numbers-api" deleted
    ```
  - ExternalName Service 로 새로 배포
    ```bash
    kubectl apply -f numbers-services/api-service-externalName.yaml
    # service/numbers-api created
    ```
  - Service 상세 정보 확인
    ```bash
    kubectl get svc numbers-api
    :'
    NAME          TYPE           CLUSTER-IP   EXTERNAL-IP                 PORT(S)   AGE
    numbers-api   ExternalName   <none>       raw.githubusercontent.com   <none>    79s
    '
    ```
  - localhost:8080
    - 계속 42만 나옴
- 실습2 - nslookup으로 도메인 네임 조회
  - ExternalName 서비스 정의 - api-service-externalName.yaml
    ```yaml
    apiVersion: v1
    kind: Service
    
    metadata:
      name: numbers-api                           # 클러스터 안에서 쓰이는 로컬 도메인 네임
    
    spec:
      type: ExternalName
      externalName: raw.githubusercontent.com     # 로컬 도메인 네임을 치환해줄 외부 도메인. CNAME(Canonical NAME)
    ```
  - nslookup으로 도메인 네임 조회
    ```bash
    kubectl exec deploy/sleep-1 -- sh -c 'nslookup numbers-api | tail -n 5'
    :'
    Name:   raw.githubusercontent.com
    Address: 185.199.108.133

    numbers-api.default.svc.cluster.local   canonical name = raw.githubusercontent.com
    '
    # tail -n 5 --> 마지막 5줄 출력
    ```
  - 

### 3.4.2. Headless Service
- ExternalName과 유사한 방식
- ClusterIP 형태로 정의
- labelSelector 없으므로 대상 Pod 도 없음
- 제공할 IP 주소 목록 Endpoint resource 와 함께 배포
- YAML에서 각 리소스의 정의를 구분하기 위해 하이픈 3개 사용
- 실습 1 - ExternalName Service를 Headless로 대체
  - Headless 서비스 정의 - api-service-headless.yaml
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: numbers-api
    spec:
      type: ClusterIP             # selector 필드가 없으므로 Headless Service 가 됨
      ports:
        -- port: 80
    ---
    kind: Endpoints               # 한 파일에 두 번째 리소스 정의
    apiVersion: v1
    metadata:
      name: numbers-api
    subsets:
      - addresses:
          - ip: 192.168.123.234   # 정적 IP 주소 목록
        ports:
          - port: 80              # 각 IP 주소에서 주시할 포트
    ```
  - 기존 Service 조회
    ```bash
    kubectl get svc
    :'
    NAME          TYPE           CLUSTER-IP     EXTERNAL-IP                 PORT(S)          AGE
    kubernetes    ClusterIP      10.43.0.1      <none>                      443/TCP          15d
    numbers-api   ExternalName   <none>         raw.githubusercontent.com   <none>           62m
    numbers-web   LoadBalancer   10.43.242.13   192.168.5.15                8080:32219/TCP   2d
    sleep-2       ClusterIP      10.43.44.128   <none>                      80/TCP           5d2h
    '
    ```
  - 기존 ExternalName 삭제
    ```bash
    kubectl delete svc numbers-api
    # service "numbers-api" deleted
    ```
  - Headless service 배포
    ```bash
    kubectl apply -f numbers-services/api-service-headless.yaml
    # service/numbers-api created
    # endpoints/numbers-api created
    ```
  - 서비스의 상세정보 확인
    ```bash
    kubectl get svc numbers-api
    :'
    NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
    numbers-api   ClusterIP   10.43.221.128   <none>        80/TCP    65s
    '
    ```
  - Endpoint의 상세 정보를 확인
    ```bash
    kubectl get endpoints numbers-api
    :'
    NAME          ENDPOINTS            AGE
    numbers-api   192.168.123.234:80   3m4s
    '
    ```
  - DNS 조회 결과를 확인
    ```bash
    kubectl exec deploy/sleep-1 -- sh -c 'nslookup numbers-api | grep "^[^*]"'
    :'
    Server:         10.43.0.10
    Address:        10.43.0.10:53
    Name:   numbers-api.default.svc.cluster.local
    Address: 10.43.221.128
    '
    ```
  - localhost:8080
    - 오류 발생 - Endpoint가 실재하지 않으므로 에러

## 3.5. Kubernetes Service의 해소 과정
- Kubernetes는 대부분의 네트워크 설정을 제공
  - 안정적인 네트워크 기술에 기반을 둔 것 - 어떻게?
    - https://github.com/kubernetes/design-proposals-archive/blob/main/network/networking.md
- ClusterIP는 네트워크상 실재하지 않는 가상 IP 주소
  - Pod -> 네트워크 프록시 -> 패킷 필터링 -> 가상 IP 주소 -> 실제 Endpoint로 연결
- Service 리소스는 삭제될때까지 IP 주소가 바뀌지 않음
- Service 내부 Controller가 Pod 변경시 마다 Endpoint 목록을 최신으로 업데이트
- 실습1 - Service의 Endpoint 목록 출력
  - sleep-2 Service의 Endpoint 목록 출력
    ```bash
    kubectl get endpoints sleep-2
    :'
    NAME      ENDPOINTS       AGE
    sleep-2   10.42.0.95:80   6d
    '
    ```
  - Pod 삭제
    ```bash
    kubectl delete pods -l app=sleep-2
    # pod "sleep-2-594d57479-qz9r6" deleted
    ```
  - Endpoint 다시 확인
    ```bash
    kubectl get endpoints
    :'
    NAME          ENDPOINTS            AGE
    kubernetes    192.168.5.15:6443    16d
    numbers-api   192.168.123.234:80   21h
    numbers-web   10.42.0.96:80        2d22h
    sleep-2       10.42.0.100:80       6d
    '
    ```
  - Deployment 삭제
    ```bash
    kubectl delete deploy sleep-2
    # deployment.apps "sleep-2" deleted
    ```
  - Endpoint 다시 확인
    ```bash
    kubectl get endpoints
    :'
    NAME          ENDPOINTS            AGE
    kubernetes    192.168.5.15:6443    16d
    numbers-api   192.168.123.234:80   22h
    numbers-web   10.42.0.96:80        2d22h
    sleep-2       <none>               6d
    '
    # Endpoint가 사라짐
    ```
### Namespace of Kubernetes
- Namespace는 Cluster를 논리적 파티션으로 분할하는 역할
  - default Namespace가 항상 존재
  - 다른 Namespace를 주가하여 여러 리소스를 묶는다.
  - Namespace 안에서는 도메인 네임을 이용하여 서비스에 접근
  - 완전한 도메인 네임으로도 Service에 접근할 수 있다.
    - numbers-api.default.svc.cluster.local
  - kube-system Namespace
    - DNS server, Kubernetes API 등 내장 컴포넌트가 소속되어 Pod에서 동작
  - 보안을 해치지 않고도 Cluster 활용도를 높일 수 있는 강력한 수단
  - Pod의 IP를 우리가 직접 참조할 일은 거의 없음
- Service는 자신이 속하지 않은 Namespace에도 접근 가능
- 실습 - 다른 Namespace에 접근
  - default Namespace의 Service resource 목록 확인
    ```bash
    kubectl get svc --namespace default
    :'
    NAME          TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)          AGE
    kubernetes    ClusterIP      10.43.0.1       <none>         443/TCP          16d
    numbers-api   ClusterIP      10.43.221.128   <none>         80/TCP           22h
    numbers-web   LoadBalancer   10.43.242.13    192.168.5.15   8080:32219/TCP   2d23h
    sleep-2       ClusterIP      10.43.44.128    <none>         80/TCP           6d1h
    '
    ```
  - kube-system Namespace Service resource 목록 확인
    ```bash
    kubectl get svc -n kube-system
    :'
    NAME             TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                      AGE
    kube-dns         ClusterIP      10.43.0.10      <none>         53/UDP,53/TCP,9153/TCP       16d
    metrics-server   ClusterIP      10.43.149.91    <none>         443/TCP                      16d
    traefik          LoadBalancer   10.43.167.168   192.168.5.15   80:32513/TCP,443:30266/TCP   16d
    '
    ```
  - 완전한 도메인 네임으로 DNS 조회
    ```bash
    kubectl exec deploy/sleep-1 -- sh -c 'nslookup numbers-api.default.svc.cluster.local | grep "^[^*]"'
    :'
    Server:         10.43.0.10
    Address:        10.43.0.10:53
    Name:   numbers-api.default.svc.cluster.local
    Address: 10.43.221.128
    '
    ```
  - Kubernetes 시스템 Namespace의 완전한 도메인 네임으로 DNS 조회하기
    ```bash
    kubectl exec deploy/sleep-1 -- sh -c 'nslookup kube-dns.kube-system.svc.cluster.local | grep "^[^*]"'
    :'
    Server:         10.43.0.10
    Address:        10.43.0.10:53
    Name:   kube-dns.kube-system.svc.cluster.local
    Address: 10.43.0.10
    '
    ```
- 실습2 - Service를 삭제할 때 Deployment는 같이 삭제 되지 않음을 확인
  - 모든 Deployment 삭제
    ```bash
    kubectl delete deploy --all
    :'
    deployment.apps "numbers-api" deleted
    deployment.apps "numbers-web" deleted
    deployment.apps "sleep-1" deleted
    '
    # Namespace 를 지정하지 않았으므로 삭제 대상은 default Namespace 에 속한 리소스
    ```
  - 모든 Deployment 조회
    ```bash
    kubectl get deploy
    # No resources found in default namespace.
    ```
  - 모든 Service 삭제
    ```bash
    kubectl delete svc --all
    :'
    service "kubernetes" deleted
    service "numbers-api" deleted
    service "numbers-web" deleted
    service "sleep-2" deleted
    '
    # Service는 명시적으로 자정하여 삭제 필요 --> default Namespace 에 속한 kubernetes API 까지도 삭제된다.
    ```
  - 남아있는 리소스 확인
    ```bash
    kubectl get all
    :'
    NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
    service/kubernetes   ClusterIP   10.43.0.1    <none>        443/TCP   117s
    '
    # Kubernetes 리소스를 관리하는 Controller 객체가 kube-system Namespace에 있어서 Kubernetes API를 복구해준다. 
    ```

## 3.6. 연습문제
- 목표
  - ch03/lab/deployments.yaml 사용
  - Pod 확인
  - 도메인 네임이 numbers-api인 pod에서 api에 접근이 가능하도록 하는 서비스 정의를 작성
  - 웹 앱 버전2를 8088 포트를 이용하여 외부에서 접근할 수 있도록 하는 서비스 정의를 작성
  - 파드의 레이블에 주의해야 한다.
- 풀이
  1. ch03/lab/deployments.yaml 내용 확인
     ```yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: lab-numbers-api
     spec:
       selector:
         matchLabels:
           app: lab-numbers-api
           version: v1
       template:
         metadata:
           labels:
             app: lab-numbers-api
             version: v1
        spec:
         containers:
           - name: api
             image: kiamol/ch03-numbers-api
     ---
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: lab-numbers-web
     spec:
       selector:
         matchLabels:
           app: lab-numbers-web
           version: v1
       template:
         metadata:
           labels:
             app: lab-numbers-web
             version: v1
         spec:
           containers:
             - name: web
               image: kiamol/ch03-numbers-web
     ---
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: lab-numbers-web-v2
     spec:
       selector:
         matchLabels:
           app: lab-numbers-web
           version: v2
       template:
         metadata:
           labels:
             app: lab-numbers-web
             version: v2
         spec:
           containers:
             - name: web
               image: kiamol/ch03-numbers-web:v2
     ```
  2. deployment 배포
     ```bash
     kubectl apply -f lab/deployments.yaml
     # deployment.apps/lab-numbers-api created
     # deployment.apps/lab-numbers-web created
     # deployment.apps/lab-numbers-web-v2 created
     ```
  3. Pod 확인
     ```bash
     kubectl get pods --show-labels
     :'
     NAME                                 READY   STATUS    RESTARTS   AGE     LABELS
     lab-numbers-api-7cf65b888-nsl99      1/1     Running   0          3m55s   app=lab-numbers-api,pod-template-hash=7cf65b888,version=v1
     lab-numbers-web-95588fcfd-9nls4      1/1     Running   0          3m55s   app=lab-numbers-web,pod-template-hash=95588fcfd,version=v1
     lab-numbers-web-v2-66b89fc6f-btw42   1/1     Running   0          3m55s   app=lab-numbers-web,pod-template-hash=66b89fc6f,version=v2
     '
     ```
  4. numbers-api API 접근 가능하도록 하는 Service 정의 - [cluster-ip-service.yaml](./lab/cluster-ip-service.yaml)
     ```yaml
     apiVersion: v1
     kind: Service
     metadata:
       name: numbers-api
     spec:
       selector:
         app: lab-numbers-api
         version: v1
       ports:
         - port: 80
       type: ClusterIP
     ```
  5. 4번 [cluster-ip-service.yaml](./lab/cluster-ip-service.yaml) 서비스 실행
     ```bash
     kubectl apply -f lab/cluster-ip-service.yaml
     # service/numbers-api created
     ```
  6. 서비스 확인
     ```bash
     kubectl get services
     :'
     NAME          TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
     kubernetes    ClusterIP   10.43.0.1      <none>        443/TCP   23h
     numbers-api   ClusterIP   10.43.15.111   <none>        80/TCP    14s
     '
     ```
  7. web v2를 8088 포트로 외부에서 접근할 수 있도록 서비스 정의 - [load-balancer-service.yaml](./lab/load-balancer-service.yaml)
     ```yaml
     apiVersion: v1
     kind: Service
     metadata:
       name: numbers-web
     spec:
       selector:
         app: lab-numbers-web
         version: v2
       ports:
         - port: 8088
           targetPort: 80
       type: LoadBalancer
     ```
  8. 7번 [load-balancer-service.yaml](./lab/load-balancer-service.yaml) 서비스 실행
     ```bash
     kubectl apply -f lab/load-balancer-service.yaml
     # service/numbers-web created
     ```
  9. 서비스 확인
     ```bash
     kubectl get services
     :'
     NAME          TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)          AGE
     kubernetes    ClusterIP      10.43.0.1      <none>         443/TCP          23h
     numbers-api   ClusterIP      10.43.15.111   <none>         80/TCP           13m
     numbers-web   LoadBalancer   10.43.67.60    192.168.5.15   8888:30905/TCP   48s
     '
     ```
  10. 웹 확인
      - http://localhost:8088


- bla
  - bla
    ```bash
    ```