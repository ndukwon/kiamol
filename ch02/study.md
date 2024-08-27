## 2.1. 쿠버네티스는 어떻게 컨테이너를 실행하고 관리하는가
- 실습
  - 컨테이너 하나 실행
    ```bash
    kubectl run hello-kiamol --image=kiamol/ch02-hello-kiamol
    # pod/hello-kiamol created
    ```
  - 컨테이너 준비상태 기다리기
    ```bash
    kubectl wait --for=condition=Ready pod hello-kiamol
    # pod/hello-kiamol condition met
    ```
  - 클러스터에 있는 모든 pod 목록을 출력한다.
    ```bash
    kubectl get pods
    : '
    NAME           READY   STATUS    RESTARTS   AGE
    hello-kiamol   1/1     Running   0          8m54s
    '
    ```
  - pod 상세 정보를 확인한다.
    ```bash
    kubectl describe pod hello-kiamol
    : '
    (결과에서 IP 마스킹 함)
    Name:             hello-kiamol
    Namespace:        default
    Priority:         0
    Service Account:  default
    Node:             lima-rancher-desktop/192.168.xx.xx
    Start Time:       Tue, 27 Aug 2024 23:00:41 +0900
    Labels:           run=hello-kiamol
    Annotations:      <none>
    Status:           Running
    IP:               10.xx.xx.xx
    IPs:
    IP:  10.xx.xx.xx
    Containers:
    hello-kiamol:
    Container ID:   docker://72b70d5492df2f94214c44e18a3a0175be732011e098b5d38222931119b5f739
    Image:          kiamol/ch02-hello-kiamol
    Image ID:       docker-pullable://kiamol/ch02-hello-kiamol@sha256:8a27476444b4c79b445f24eeb5709066a9da895b871ed9115e81eb5effeb5496
    Port:           <none>
    Host Port:      <none>
    State:          Running
    Started:      Tue, 27 Aug 2024 23:01:17 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
    /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-bj99n (ro)
    Conditions:
    Type                        Status
    PodReadyToStartContainers   True
    Initialized                 True
    Ready                       True
    ContainersReady             True
    PodScheduled                True
    Volumes:
    kube-api-access-bj99n:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
    QoS Class:                   BestEffort
    Node-Selectors:              <none>
    Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
    node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
    Events:
    Type    Reason     Age    From               Message
      ----    ------     ----   ----               -------
    Normal  Scheduled  10m    default-scheduler  Successfully assigned default/hello-kiamol to lima-rancher-desktop
    Normal  Pulling    10m    kubelet            Pulling image "kiamol/ch02-hello-kiamol"
    Normal  Pulled     9m38s  kubelet            Successfully pulled image "kiamol/ch02-hello-kiamol" in 30.902s (30.902s including waiting). Image size: 23422228 bytes.
    Normal  Created    9m37s  kubelet            Created container hello-kiamol
    Normal  Started    9m36s  kubelet            Started container hello-kiamol
    '
    ```
  - pod 기본정보 확인
    ```bash
    kubectl get pod hello-kiamol
    : '
    NAME           READY   STATUS    RESTARTS   AGE
    hello-kiamol   1/1     Running   0          47m
    '
    ```
  - 네트워크 상세 정보 중 특정 항목 지정 출력
    ```bash
    kubectl get pod hello-kiamol --output custom-columns=NAME:metadata.name,NODE_IP:status.hostIP,POD_IP:status.podIP
    : '
    (결과에서 IP 마스킹 함)
    NAME           NODE_IP        POD_IP
    hello-kiamol   192.168.xx.xx   10.xx.xx.xx
    '
    ```
    - NODE_IP: 머신의 주소
    - POD_IP: 클러스터 속 pod의 가상 IP 주소
  - JSONPath로 복잡한 출력을 구성
    ```bash
    kubectl get pod hello-kiamol -o jsonpath='{.status.containerStatuses[0].containerID}'
    # docker://72b70d5492df2f94214c44e18a3a0175be732011e098b5d38222931119b5f739%
    ```
- CRI: Container Runtime Interface
  - 컨테이너 생성과 삭제, 정보 확인 기능이 표준 API로 제공
  - 
- 실습
  - pod에 포함된 컨테이너 찾기
    ```bash
    docker container ls -q --filter label=io.kubernetes.container.name=hello-kiamol
    ```
  - 해당 컨테이너 삭제하기
    ```bash
    docker container rm -f $(docker container ls -q --filter label=io.kubenetes.container.name=hello-kiamol)
    ```
  - pod 상태 확인
    ```bash
    kubectl get pod hello-kiamol
    ```
  - 이전 컨테이너 다시 찾이보기
    ```bash
    docker container ls -q --filter label=io.kubernetes.container.name=hello-kiamol
    ```

- kubectl 기능 - 네트워크 트래픽 노드에서 파드로 전달 할 수 있는 기능
- 실습
  - 8080 --> 80으로 전달
    ```bash
    kubectl port-forward pod/hello-kiamol 8080:80
    : '
    Forwarding from 127.0.0.1:8080 -> 80
    Forwarding from [::1]:8080 -> 80
    Handling connection for 8080
    Handling connection for 8080
    ^Z
    [1]  + 43047 suspended  kubectl port-forward pod/hello-kiamol 8080:80
    '
    ```
    - http://localhost:8080 에 접근해서 확인