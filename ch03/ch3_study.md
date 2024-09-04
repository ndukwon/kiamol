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


- bla
  - bla
    ```bash
    ```