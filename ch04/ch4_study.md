# 4. ConfigMap 과 비밀값으로 앱 설정하기
4.1. Kubernetes에서 앱에 설정이 전달되는 과정
4.2. ConfigMap에 저장한 설정 파일 사용하기
4.3. ConfigMap에 담긴 설정 값 데이터 주입하기
4.4. 비밀값을 이용하여 민감한 정보가 담긴 설정값 다루기
4.5. Kubernetes의 앱 설정 관리
4.6. 연습 문제

## 4.1. Kubernetes에서 앱에 설정이 전달되는 과정
- ConfigMap, 비밀값은 컨테이너 환경의 일부가 되고 컨테이너가 이 값을 읽는다.
- ConfigMap: Pod에서 읽어들이는 data를 저장하는 리소스
  - Key-value, Text, Binary 등 다양한 data 가능
  - Key-value ex) 환경변수 형태로 주입
  - Text ex) JSON, XML, YAML, TOML, INI 등 설정파일을 Pod에 저장
  - Binary ex) 라이선스 키 등
  - 여러개의 ConfigMap 전달 가능
  - ConfigMap을 여러 Pod에 전달할 수도 있다.
  - 생성방법
    1. kubectl 명령어 실행시 같이 전달
- 실습1 - Pod 속 컨테이너에 설정된 환경변수 읽어보기
  - 설정값 없이 sleep 이미지 Pod 실행
    ```bash
    kubectl apply -f sleep/sleep.yaml
    # deployment.apps/sleep created
    ```
  - Pod이 준비될때 까지 대기
    ```bash
    kubectl wait --for=condition=Ready pod -l app=sleep
    # pod/sleep-58b5d6d56-nxkkw condition met
    ```
  - Pod 속 컨테이너에 설정된 몇가지 환경 변수의 값을 확인 1/3
    ```bash
    kubectl exec deploy/sleep -- printenv HOSTNAME KIAMOL_CHAPTER
    :'
    sleep-58b5d6d56-nxkkw
    command terminated with exit code 1
    '
    # printenv 는 환경 변수를 출력하는 리눅스 명령어
    # HOSTNAME 는 Kubernetes 가 Pod 이름을 모든 컨테이너에 설정
    # KIAMOL_CHAPTER 는 아직 설졍되지 않았으므로 오류코드와 함께 종료
    ```
  - yaml에 환경변수 추가 (간단한 설정인 경우)
    ```yaml
    apiVersion: v1
    kind: Deployment
    metadata:
      name: sleep
    spec:
      selector:
        matchLabels:
          app: sleep
      template:
        metadata:
          labels:
            app: sleep
        spec:
          containers:
            - name: sleep
              image: kiamol/ch03-sleep
    # added
              env:
                - name: KIAMOL_CHAPTER
                  value: "04"
    ```
  - Deployment 업데이트
    ```bash
    kubectl apply -f sleep/sleep-with-env.yaml
    # deployment.apps/sleep configured
    ```
  - 환경변수 확인 2/3
    ```bash
    kubectl exec deploy/sleep -- printenv HOSTNAME KIAMOL_CHAPTER
    :'
    sleep-5768df5557-xt857
    04
    '
    ```
  - yaml에 ConfigMap 읽어들이기 추가
    ```yaml
    apiVersion: v1
    kind: Deployment
    metadata:
      name: sleep
    spec:
      selector:
        matchLabels:
          app: sleep
      template:
        metadata:
          labels:
            app: sleep
        spec:
          containers:
            - name: sleep
              image: kiamol/ch03-sleep
              env:
                - name: KIAMOL_CHAPTER
                  value: "04"
    # added
                - name: KIAMOL_SECTION
                  valueFrom:
                    configMapKeyRef:
                      name: sleep-config-literal
                      key: kiamol.section
    ```
  - kubectl 로 ConfigMap 생성
    ```bash
    kubectl create configmap sleep-config-literal --from-literal=kiamol.section='4.1'
    # configmap/sleep-config-literal created
    ```
  - ConfigMap에 있는 데이터 확인
    ```bash
    kubectl get cm sleep-config-literal
    :'
    NAME                   DATA   AGE
    sleep-config-literal   1      66s
    '
    ```
  - ConfigMap의 상세 정보를 출력
    ```bash
    kubectl describe cm sleep-config-literal
    :'
    Name:         sleep-config-literal
    Namespace:    default
    Labels:       <none>
    Annotations:  <none>

    Data
    ====
    kiamol.section:
    ----
    4.1


    BinaryData
    ====

    Events:  <none>
    '
    ```
  - Deployment 업데이트
    ```bash
    kubectl apply -f sleep/sleep-with-configMap-env.yaml
    # deployment.apps/sleep configured
    ```
  - 환경변수 확인 3/3
    ```bash
    kubectl exec deploy/sleep -- sh -c 'printenv | grep "^KIAMOL"'
    :'
    KIAMOL_SECTION=4.1
    KIAMOL_CHAPTER=04
    '
    ```

## 4.2. ConfigMap 에 저장한 설정 파일 사용하기
- Kubernetes 는 ConfigMap 생성하고 만드는 방법에 여러 변화가 있었음
- 보편적인 설정 전략
  1. 기본 설정값은 컨테이너 이미지에 포함시킨다.
     - docker run 으로 실행될 수 있게
  2. ConfigMap에 담아 컨테이너에 전달
     - 컨테이너에 파일 형태로 주입, 또는 파일을 덮어쓰는 형태
     - 앱에서 지정된 경로에서 설정 데이터 파일을 확인
  3. 변경이 필요한 설정 값 --> deployment 내 Pod 정의에서 환경변수로 적용

- 실습1 - 환경변수 파일로 ConfigMap 생성하고 사용
  - env 파일 - ch04.env
    ```env
    KIAMOL_CHAPTER=ch04
    KIAMOL_SECTION=ch04-4.1
    KIAMOL_EXERCISE=try it now
    ```
  - 환경파일의 내용으로 ConfigMap 생성
    ```bash
    kubectl create configmap sleep-config-env-file --from-env-file=sleep/ch04.env
    # configmap/sleep-config-env-file created
    ```
  - ConfigMap의 상세 정보 확인
    ```bash
    kubectl get cm sleep-config-env-file
    :'
    NAME                    DATA   AGE
    sleep-config-env-file   3      26s
    '
    ```
  - 새로운 ConfigMap의 설정을 적용
    ```bash
    kubectl apply -f sleep/sleep-with-configMap-env-file.yaml
    # deployment.apps/sleep configured
    ```
  - 컨테이너에 적용된 환경 변수의 값 확인
    ```bash
    kubectl exec deploy/sleep -- sh -c 'printenv | grep "^KIAMOL"'
    :'
    KIAMOL_EXERCISE=try it now
    KIAMOL_SECTION=4.1
    KIAMOL_CHAPTER=04
    '
    # KIAMOL_CHAPTER, KIAMOL_SECTION 항목이 선언한 것과 다름
    # yaml에서 envFrom보다 env로 선언된 것이 우선하기 때문
    ```
  - 순서 변경된 ConfigMap의 설정을 적용
    ```bash
    kubectl apply -f sleep/sleep-with-configMap-env-file_v2.yaml
    # deployment.apps/sleep unchanged
    ```
  - 컨테이너에 적용된 환경 변수의 값 확인 v2
    ```bash
    kubectl exec deploy/sleep -- sh -c 'printenv | grep "^KIAMOL"'
    :'
    KIAMOL_EXERCISE=try it now
    KIAMOL_SECTION=4.1
    KIAMOL_CHAPTER=04
    '
    # 순서 바꾼다고 달라지지 않음
    ```
  - env 부분을 삭제한 ConfigMap의 설정을 적용
    ```bash
    kubectl apply -f sleep/sleep-with-configMap-env-file_v3.yaml
    # deployment.apps/sleep configured
    ```
  - 컨테이너에 적용된 환경 변수의 값 확인 v3
    ```bash
    kubectl exec deploy/sleep -- sh -c 'printenv | grep "^KIAMOL"'
    :'
    KIAMOL_EXERCISE=try it now
    KIAMOL_SECTION=ch04-4.1
    KIAMOL_CHAPTER=ch04
    '
    # env 항목을 삭제하면 적용됨
    ```
- 실습2 - todo-list 앱 띄우기
  - yaml 구성 살펴보기
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: todo-web
    spec:
      ports:
        - port: 8080
          targetPort: 80
      selector:
        app: todo-web
      type: LoadBalancer
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: todo-web
    spec:
      selector:
        matchLabels:
          app: todo-web
      template:
        metadata:
          labels:
            app: todo-web
        spec:
          containers:
            - name: web
              image: kiamol/ch04-todo-list
              env:
              - name: Logging__LogLevel__Default  # .net의 설정
                value: Warning
    ```
  - Service와 함께 앱 배치
    ```bash
    kubectl apply -f todo-list/todo-web.yaml
    # service/todo-web created
    # deployment.apps/todo-web created
    ```
  - Pod가 준비 상태가 될때까지 대기
    ```bash
    kubectl wait --for=condition=Ready pod -l app=todo-web
    # pod/todo-web-557496b8c4-qtrwm condition met
    ```
  - 앱에 접근하기 위한 주소를 파일로 출력
    ```bash
    kubectl get svc todo-web
    :'
    NAME       TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)          AGE
    todo-web   LoadBalancer   10.43.54.237   192.168.5.15   8080:31763/TCP   23h
    '
    ```
  - 웹 페이지 확인 - http://localhost:8080
  - config 페이지 확인 - http://localhost:8080/config
    - 접근되지 않음
  - 앱 로그 확인
    ```bash
    kubectl logs -l app=todo-web
    :'
    info: ToDoList.Pages.NewModel[0]
      New item created
    warn: ToDoList.Pages.ConfigModel[0]
      Attempt to view config settings
    '
    ```
- 실습3 - 
  1. yaml 에 config 정의
     ```yaml
     apiVersion: v1
     kind: ConfigMap
     metadata:
       name: todo-web-config-dev
     data:
       config.json: |-
         {
           "ConfigController": {
             "Enabled": true
           }
         }
     ```
  2. JSON 이 담긴 ConfigMap 생성
     ```bash
     kubectl apply -f todo-list/configMaps/todo-web-config-dev.yaml
     # configmap/todo-web-config-dev created
     # yaml 의 설정대로 todo-web-config-dev 이름의 ConfigMap이 만들어졌다. 
     ```
  3. ConfigMap 참조 하도록 앱 업데이트
     ```bash
     kubectl apply -f todo-list/todo-web-dev.yaml
     # deployment.apps/todo-web configured
     ```
  4. config 페이지 확인 - http://localhost:8080/config
    - 즉각 적용은 안되었지만 1분정도 후 적용되었다.
    - 이 부분이 활성화 적용된다. https://github.com/sixeyed/kiamol/blob/master/ch04/docker-images/todo-list/src/Pages/Config.cshtml.cs#L27
     