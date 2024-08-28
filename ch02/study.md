# 2. Pod와 deployment로 컨테이너 실행하기
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

- kubectl port-forward 기능 - 네트워크 트래픽 노드에서 파드로 전달 할 수 있는 기능
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


## 2.2. Controller 객체와 함께 Pod 실행하기
- 쿠버네티스가 Deployment를 생성하면 Deployment가 Pod를 생성한다.
- 실습 - Deployment를 사용하여 실행. Deployment의 이름과 Container를 실행할 이미지 이름이 필요
  - 'hello-kiamol-2' 를 생성
    ```bash
    kubectl create deployment hello-kiamol-2 --image=kiamol/ch02-hello-kiamol
    # deployment.apps/hello-kiamol-2 created
    ```
  - Pod 목록 출력
    ```bash
    kubectl get pods
    : '
    NAME                             READY   STATUS    RESTARTS      AGE
    hello-kiamol                     1/1     Running   1 (23h ago)   25h
    hello-kiamol-2-5f5ddcfc5-rjpjk   1/1     Running   0             20m
    '
    # deployment가 선택한 Pod으로 전달된다.
    ```
- Deployment만 만들면 Pod을 대신 만들어 준다.
  - Deployment가 부여한 Pod 레이블 확인
    ```bash
    kubectl get deploy hello-kiamol-2 -o jsonpath='{.spec.template.metadata.labels}'
    # {"app":"hello-kiamol-2"}%
    # app 이라는 레이블에 hello-kiamol-2 라는 이름을 달았다.
    ```
  - 같은 레이블을 가진 Pod의 목록 보기
    ```bash
    kubectl get pods -l app=hello-kiamol-2
    : '
    NAME                             READY   STATUS    RESTARTS   AGE
    hello-kiamol-2-5f5ddcfc5-rjpjk   1/1     Running   0          51m
    '
    ```
- Deployment와 Pod은 직접적인 관계는 없고, Deployment에서는 label 셀렉터와 일치하는 Pod을 찾고 이 label이 수정되면 Deployment는 더이상 인지 하지 못한다.
  - 모든 Pod 이름과 Label 확인
    ```bash
    kubectl get pods -o custom-columns=NAME:metadata.name,LABELS:metadata.labels
    : '
    NAME                             LABELS
    hello-kiamol                     map[run:hello-kiamol]
    hello-kiamol-2-5f5ddcfc5-rjpjk   map[app:hello-kiamol-2 pod-template-hash:5f5ddcfc5]
    '
    ```
  - 'app' 레이블 수정
    ```bash
    kubectl label pods -l app=hello-kiamol-2 --overwrite app=hello-kiamol-x
    # pod/hello-kiamol-2-5f5ddcfc5-rjpjk labeled
    ```
  - 다시 확인
    ```bash
    kubectl get pods -o custom-columns=NAME:metadata.name,LABELS:metadata.labels
    : '
    NAME                             LABELS
    hello-kiamol                     map[run:hello-kiamol]
    hello-kiamol-2-5f5ddcfc5-mp9bm   map[app:hello-kiamol-2 pod-template-hash:5f5ddcfc5]
    hello-kiamol-2-5f5ddcfc5-rjpjk   map[app:hello-kiamol-x pod-template-hash:5f5ddcfc5]
    '
    # Deployment가 "hello-kiamol-2" Pod이 없어졌기 때문에 새로 만듦
    ```
  - 원복
    ```bash
    kubectl label pods -l app=hello-kiamol-x --overwrite app=hello-kiamol-2
    # pod/hello-kiamol-2-5f5ddcfc5-rjpjk labeled
    ```
  - 다시 확인
    ```bash
    kubectl get pods -o custom-columns=NAME:metadata.name,LABELS:metadata.labels
    : '
    NAME                             LABELS
    hello-kiamol                     map[run:hello-kiamol]
    hello-kiamol-2-5f5ddcfc5-rjpjk   map[app:hello-kiamol-2 pod-template-hash:5f5ddcfc5]
    '
    # Pod 하나만 유지하는 설정이므로 둘 중 하나를 삭제 규칙에 따라 삭제 되었다.
    # 여기선 나중에 새로 생긴 pod이 삭제되었다.
    ```
- kubectl port-forward 기능 - 네트워크 트래픽 노드에서 파드로 전달 할 수 있는 기능
  - port-forward 를 Deployment에 맡긴다.
    ```bash
    kubectl port-forward deploy/hello-kiamol-2 8080:80
    : '
    Forwarding from 127.0.0.1:8080 -> 80
    Forwarding from [::1]:8080 -> 80
    Handling connection for 8080
    Handling connection for 8080
    '
    ```
  