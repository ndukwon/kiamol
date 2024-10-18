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

## 4.3. ConfigMap에 담긴 설정값 데이터 주입하기
- 컨테이너 파일시스템
  - 컨테이터 이미지 + 그 외 출처에서 온 파일로 구성된 가상 구조
  - 컨테이너 파일 시스템 속 파일로 설정값을 주입하여 전달이 가능
- Kubernetes가 컨테이너 파일시스템 구성에 ConfigMap 추가 가능 - 디렉토리 형태로 읽어올 수 있다.
  - ex)
    - /app <-- 컨테이너 이미지
    - /app/config <-- ConfigMap
- Pod 이 동작중인 상태에서 ConfigMap을 업데이트하면 컨테이너까지 전달된다.
  - 앱마다 참조하는 방식이 달라서 적용여부는 알 수 없다. 
- 실습1
  - ConfigMap을 Pod의 volume에 로드하고 특정 경로에 volumeMount 하는 todo-web-dev.yaml
    ```yaml
    spec:
      containers:
        - name: web
          image: kiamol/ch04-todo-list
          volumeMounts:                       # 컨테이너에 volume을 mount 
            - name: config                    # volume 부분에서 정의한 이름
              mountPath: "/app/config"        # volume 이 올라갈 경로
              readOnly: true                  # 읽기 전용 여부
      volumes:                                # Pod 레벨에서 volume 정의함
        - name: config                        # volume 의 이름 선언
          configMap:                          # volume 의 원본 - ConfigMap
            name: todo-web-config-dev         # ConfigMap 의 이름
    ```
  - 기본 설정값이 담긴 설정 파일 확인
    ```bash
    kubectl exec deploy/todo-web -- sh -c 'ls -l /app/app*.json'
    :'
    -rw-r--r--    1 root     root           469 Sep  1  2022 /app/appsettings.json
    '
    ```
  - 다른파일도 확인해봄
    ```bash
    kubectl exec deploy/todo-web -- sh -c 'ls'
    :'
    Microsoft.Data.Sqlite.dll
    Microsoft.EntityFrameworkCore.Abstractions.dll
    Microsoft.EntityFrameworkCore.Relational.dll
    Microsoft.EntityFrameworkCore.Sqlite.dll
    Microsoft.EntityFrameworkCore.dll
    Microsoft.Extensions.DependencyModel.dll
    Npgsql.EntityFrameworkCore.PostgreSQL.dll
    Npgsql.dll
    Prometheus.AspNetCore.dll
    Prometheus.NetCore.dll
    Prometheus.NetStandard.dll
    SQLitePCLRaw.batteries_v2.dll
    SQLitePCLRaw.core.dll
    SQLitePCLRaw.provider.e_sqlite3.dll
    ToDoList
    ToDoList.deps.json
    ToDoList.dll
    ToDoList.pdb
    ToDoList.runtimeconfig.json
    appsettings.json
    config
    runtimes
    web.config
    wwwroot
    '
    ```
  - volumeMount로 주입된 설정 파일 확인
    ```bash
    kubectl exec deploy/todo-web -- sh -c 'ls -l /app/config/*.json'
    :'
    lrwxrwxrwx    1 root     root            18 Sep 22 14:58 /app/config/config.json -> ..data/config.json
    '
    ```
  - volumeMount가 실제로 읽기 전용인지 확인
    ```bash
    kubectl exec deploy/todo-web -- sh -c 'echo ch04 >> /app/config/config.json'
    :'
    sh: cant create /app/config/config.json: Read-only file system
    command terminated with exit code 1
    '

- 실습2 - ConfigMap을 디렉토리 형태로 읽기
  - 앱 컨트롤러 설정, 로그설정을 두개의 JSON으로 분리 수정 yaml 
    ```yaml
    data:
      config.json: |-
        {
          "ConfigController": {
            "Enabled": true
          }
        }
      logging.json: |-
        {
          "Logging": {
            "LogLevel": {
              "ToDoList.Pages": "Debug"
            }
          }
        }
    ```
  - 앱 로그 확인
    ```bash
    kubectl logs -l app=todo-web
    ```
  - ConfigMap 업데이트
    ```bash
    kubectl apply -f todo-list/configMaps/todo-web-config-dev-with-logging.yaml
    # configmap/todo-web-config-dev configured
    ```
  - 업데이트 반영 대기
    ```bash
    sleep 120
    ```
  - 설정 파일에 반영되었는지 확인
    ```bash
    kubectl exec deploy/todo-web -- sh -c 'ls -l /app/config/*.json'
    :'
    lrwxrwxrwx    1 root     root            18 Sep 22 14:58 /app/config/config.json -> ..data/config.json
    lrwxrwxrwx    1 root     root            19 Sep 27 18:07 /app/config/logging.json -> ..data/logging.json
    '
    ```
  - 앱에 접근하여 로그출력 변경사항 확인
    ```bash
    kubectl logs -l app=todo-web
    :'
    dbug: ToDoList.Pages.IndexModel[0]
      GET / called
    dbug: ToDoList.Pages.IndexModel[0]
      Fetched count: 0 from service
    '
    # debug 레벨 로그가 나오기 시작
    ```
- 실습3 - configMap의 path 설정 잘못한 경우
  - path 설정 잘못된 yaml [todo-web-dev-broken.yaml](./todo-list/todo-web-dev-broken.yaml)
    ```yaml
    spec:
      containers:
        - name: web
          image: kiamol/ch04-todo-list
          volumeMounts:
            - name: config              # Mount할 ConfigMap volume의 이름
              mountPath: "/app""        # 잘못 설정된 path
    ```
  - 설정 오류 Pod 배치
    ```bash
    kubectl apply -f todo-list/todo-web-dev-broken.yaml
    # deployment.apps/todo-web configured
    ```
  - 앱 동작 확인 - http://localhost:8080
  - 앱 로그 확인
    ```bash
    kubectl logs -l app=todo-web
    :'
    dbug: ToDoList.Pages.ListModel[0]
          Fetched 0 items from service
    dbug: ToDoList.Pages.IndexModel[0]
          GET / called
    dbug: ToDoList.Pages.IndexModel[0]
          Fetched count: 0 from service
    dbug: ToDoList.Pages.ListModel[0]
          GET /list called
    dbug: ToDoList.Pages.ListModel[0]
          Fetched 0 items from service
      * You intended to execute a .NET application:
          The application 'ToDoList.dll' does not exist.
      * You intended to execute a .NET SDK command:
          No .NET SDKs were found.

    Download a .NET SDK:
    https://aka.ms/dotnet-download

    Learn about SDK resolution:
    https://aka.ms/dotnet/sdk-not-found
    '
    # 기존 Pod
    ```
  - Pod 상태 확인
    ```bash
    kubectl get pods -l app=todo-web
    :'
    NAME                        READY   STATUS             RESTARTS      AGE
    todo-web-68848b9c4d-plrwk   1/1     Running            2 (36h ago)   6d16h
    todo-web-7945b8fd6f-6vtl8   0/1     CrashLoopBackOff   4 (19s ago)   2m38s
    '
    # 3번(?)의 재시도 후 재시작을 멈춘다.
    # Pod 수가 2개
    ```
- 실습4 - 다시 되돌리기
  - 정상적으로 수정된 yaml - [todo-web-dev-no-logging.yaml](./todo-list/todo-web-dev-no-logging.yaml)
    ```yaml
    spec:
      containers:
        - name: web
          image: kiamol/ch04-todo-list
          volumeMounts:
            - name: config
              mountPath: "/app/config"
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: todo-web-config-dev
            items:                          # ConfigMap에서 전달할 데이터 항목을 지정
              - key: config.json            # config.json 항목 지정?
                path: config.json           # config.json 파일로 전달하도록 지정
    ```
  - 변경된 정의 배치
    ```bash
    kubectl apply -f todo-list/todo-web-dev-no-logging.yaml
    # deployment.apps/todo-web configured
    ```
  - /app/config/ 디렉터리의 내용 확인
    ```bash
    kubectl exec deploy/todo-web -- sh -c 'ls /app/config'
    # config.json
    ```
  - 앱 페이지를 두어 번 새로고침 - http://localhost:8080
  - 출력되는 로그 확인
    ```bash
    kubectl logs -l app=todo-web
    :'
    fail: Microsoft.AspNetCore.Antiforgery.DefaultAntiforgery[7]
      An exception was thrown while deserializing the token.
      Microsoft.AspNetCore.Antiforgery.AntiforgeryValidationException: The antiforgery token could not be decrypted.
       ---> System.Security.Cryptography.CryptographicException: The key {8a64dc35-84b6-46cd-8a93-fa6ba39435b4} was not found in the key ring. For more information go to http://aka.ms/dataprotectionwarning
         at Microsoft.AspNetCore.DataProtection.KeyManagement.KeyRingBasedDataProtector.UnprotectCore(Byte[] protectedData, Boolean allowOperationsOnRevokedKeys, UnprotectStatus& status)
         at Microsoft.AspNetCore.DataProtection.KeyManagement.KeyRingBasedDataProtector.Unprotect(Byte[] protectedData)
         at Microsoft.AspNetCore.Antiforgery.DefaultAntiforgeryTokenSerializer.Deserialize(String serializedToken)
         --- End of inner exception stack trace ---
         at Microsoft.AspNetCore.Antiforgery.DefaultAntiforgeryTokenSerializer.Deserialize(String serializedToken)
         at Microsoft.AspNetCore.Antiforgery.DefaultAntiforgery.GetCookieTokenDoesNotThrow(HttpContext httpContext)
    '
    # 책 내용과 조금 다름
    ```
  - Pod의 목록과 상태 확인
    ```bash
    kubectl get pods -l app=todo-web
    :'
    NAME                        READY   STATUS    RESTARTS   AGE
    todo-web-7644fb5948-2r8sb   1/1     Running   0          3m29s
    '
    ```
    
## 4.4. Secret 이용하여 민감한 정보가 담긴 설정값 다루기
- 비밀값은 클러스터 내부에서 별도로 관리
- 비밀값 객체에 접근할 권한이 있다면 평문으로 볼수 있다.
- 하지만 평문도 Base64로 인코딩된 상태
- Pod 컨테이너에 전달 된 후에는 평문이 담긴 텍스트파일이 된다.
  - 앱에서 파일 권한 설정으로 접근을 제어할 수 있다.
- 실습1 - secret 생성 및 조회
  - 평문 리터럴로 비밀값 생성
    ```bash
    kubectl create secret generic sleep-secret-literal --from-literal=secret=shh...
    # secret/sleep-secret-literal created
    ```
  - 비밀값의 상세 정보 확인
    ```bash
    kubectl describe secret sleep-secret-literal
    :'
    Name:         sleep-secret-literal
    Namespace:    default
    Labels:       <none>
    Annotations:  <none>

    Type:  Opaque

    Data
    ====
    secret:  6 bytes
    '
    ```
  - 비밀값의 평문 확인
    ```bash
    kubectl get secret sleep-secret-literal -o jsonpath='{.data.secret}'
    # c2hoLi4u%
    ```
  - 비밀값의 평문을 base64 디코딩 확인
    ```bash
    kubectl get secret sleep-secret-literal -o jsonpath='{.data.secret}' | base64 -d
    # shh...%
    ```
- 실습2 - Secret으로 환경변수 주입
  - 주입받는 앱 yaml 정의
    ```yaml
    spec:
      containers:
        - name: sleep
          image: kiamol/ch03-sleep
          env:                                  # 환경변수의 정의
          - name: KIAMOL_SECRET                 # 컨테이너에 전달될 환경변수의 이름
            valueFrom:                          # 환경 변수의 값은 외부에서 온다는 설정 
              secretKeyRef:                     # 비밀값에서 온다는 설정
                name: sleep-secret-literal      # 비밀값 이름
                key: secret                     # 비밀값의 항목 이름
    ```
  - sleep deployment 업데이트
    ```bash
    kubectl apply -f sleep/sleep-with-secret.yaml
    # deployment.apps/sleep configured
    ```
  - Pod속 환경변수 확인
    ```bash
    kubectl exec deploy/sleep -- printenv KIAMOL_SECRET
    # 첫번째 시도
    # command terminated with exit code 1
    # 두번째 시도
    # shh...
    ```
- 실습3 - yaml에 Secret을 환경변수로 정의하여 전달 
  - 로그인 정보 담긴 yaml [todo-db-secret-test.yaml](todo-list/secrets/todo-db-secret-test.yaml)
    ```yaml
    apiVersion: v1
    kind: Secret                            # 리소스 유형 - Secret
    metadata:
      name: todo-db-secret-test             # Secret 의 이름
    type: Qpaque                            # Qpaque: 임의의 text 데이터
    stringData:                             # Text data
      POSTGRES_PASSWORD: "kiamol-2*2*"      # 저장할 data(key-value pair)
    ```
  - Secret 생성
    ```bash
    kubectl apply -f todo-list/secrets/todo-db-secret-test.yaml
    # secret/todo-db-secret-test created
    ```
  - Data가 인코딩 되었는지 확인 1/2
    ```bash
    kubectl get secret todo-db-secret-test
    :'
    NAME                  TYPE     DATA   AGE
    todo-db-secret-test   Opaque   1      9m43s
    '
    ```
  - Data가 인코딩 되었는지 확인 1/2
    ```bash
    kubectl get secret todo-db-secret-test -o jsonpath='{.data.POSTGRES_PASSWORD}'
    # a2lhbW9sLTIqMio=%
    # value가 encoding 되어 있음
    ```
  - Secret 객체의 annotation에 저장된 내용 확인
    ```bash
    kubectl get secret todo-db-secret-test -o jsonpath='{.metadata.annotations}'
    # {"kubectl.kubernetes.io/last-applied-configuration":"{\"apiVersion\":\"v1\",\"kind\":\"Secret\",\"metadata\":{\"annotations\":{},\"name\":\"todo-db-secret-test\",\"namespace\":\"default\"},\"stringData\":{\"POSTGRES_PASSWORD\":\"kiamol-2*2*\"},\"type\":\"Opaque\"}\n"}%
    # value가 encoding 되어있지 않음
    ```
- 실습4 - yaml에 Secret을 파일로 전달
  - 파일로 전달하는 yaml - [todo-db-test.yaml](todo-list/todo-db-test.yaml)
    ```yaml
    spec:
      containers:
        - name: db
          image: postgres:11.6-alpine
          env:
            - name: POSTGRES_PASSWORD_FILE            # 설정 파일이 마운트될 경로
              value: /secrets/postgres_password
            volumeMounts:                             # Volume 마운트 설정
              - name: secret                          # 마운트할 Volume 이름
                mountPath: ".secrets"
      volumes:
        - name: secret
          secret:                                     # Secret 에서 볼륨 생성
            secretName: todo-db-secret-test           # Volume을 만들 Secret 이름
            defaultMode: 0400                         # 파일의 권한 설정(0400 = 소유자의 읽기 권한, 0200 = 소유자의 쓰기 권한, 0100 = 소유자의 실행 권한)
            items:                                    # Secret 의 특정 데이터 항목을 지정
              - key: POSTGRES_PASSWORD
                path: postgres_password
    ```
  - yaml 적용
    ```bash
    kubectl apply -f todo-list/todo-db-test.yaml
    # service/todo-db created
    # deployment.apps/todo-db created
    ```
  - Pod 의 로그로 확인
    ```bash
    kubectl logs -l app=todo-db --tail 1
    # 2024-10-05 16:49:13.916 UTC [1] LOG:  database system is ready to accept connections
    ```
  - Password 설정파일의 권한 확인
    ```bash
    kubectl exec deploy/todo-db -- sh -c 'ls -l $(readlink -f /secrets/postgres_password)'
    # -r--------    1 root     root            11 Oct  5 16:48 /secrets/..2024_10_05_16_48_29.2406213555/postgres_password
    # readlink 는 파일의 실제 경로를 알려줌
    # Kubernetes 에서 마운트된 파일은 symlink 를 통해 설정된 경로로 연결
    ```
  - Password 설정파일의 권한 확인 2
    ```bash
    kubectl exec deploy/todo-db -- sh -c 'ls -la /secrets/postgres_password'
    # lrwxrwxrwx    1 root     root            24 Oct  5 16:48 /secrets/postgres_password -> ..data/postgres_password
    ```

- 실습5 - 
  - configMap 적용
    ```bash
    kubectl apply -f todo-list/configMaps/todo-web-config-test.yaml
    # configmap/todo-web-config-test created
    ```
  - Secret 적용
    ```bash
    kubectl apply -f todo-list/secrets/todo-web-secret-test.yaml
    # secret/todo-web-secret-test created
    ```
  - Deployment 적용
    ```bash
    kubectl apply -f todo-list/todo-web-test.yaml
    # service/todo-web-test created
    # deployment.apps/todo-web-test created
    ```
  - 컨테이너 속 인증 정보 파일 확인
    ```bash
    kubectl exec deploy/todo-web-test -- cat /app/secrets/secrets.json
    :'
    {
      "ConnectionStrings": {
        "ToDoDb": "Server=todo-db;Database=todo;User Id=postgres;Password=kiamol-2*2*;"
      }
    }%
    '
    ```

## 4.5. Kubernetes의 앱 설정 관리
- 외부 설정값 주입 설계시 고려사항
  - 앱 중단없이 설정 변경에 대응이 필요한가? - Pod 교체도 안되는 무중단 업데이트?
    - Pod 교체 필요한 방식
      - 환경변수 - Pod 교체 필요
      - 볼륨을 수정하는 것
      - 장점: 변경의 이력이 남는다. 이전 설정으로 돌아가는 선택지가 있다.
    - Pod 교체 필요없이 해결
      - 볼륨 마운트를 이용하여 설정파일을 수정 - ConfigMap, Secret 을 업데이트 하는 방식
  - 민감정보를 어떻게 관리할 것인가?
    - 민감정보 관리 자동화 방식- 베포시 YAML 템플릿 파일의 빈칸이 보안 저장소에 있는 정보로 채워지는 방식
    - 설정 전담 조직에서 민감정보 주입 방식
- 실습 - 지금까지의 리소스 정리
  - yaml이 있는 디렉토리를 지정하여 디렉토리 내 모든 yaml로 정의 된 리소스를 한번에 삭제 가능하다.
    ```bash
    kubectl delete -f sleep/
    :'
    deployment.apps "sleep" deleted
    Error from server (NotFound): error when deleting "sleep/sleep-with-configMap-env-file_v2.yaml": deployments.apps "sleep" not found
    Error from server (NotFound): error when deleting "sleep/sleep-with-configMap-env-file_v3.yaml": deployments.apps "sleep" not found
    Error from server (NotFound): error when deleting "sleep/sleep-with-configMap-env.yaml": deployments.apps "sleep" not found
    Error from server (NotFound): error when deleting "sleep/sleep-with-env.yaml": deployments.apps "sleep" not found
    Error from server (NotFound): error when deleting "sleep/sleep-with-secret.yaml": deployments.apps "sleep" not found
    Error from server (NotFound): error when deleting "sleep/sleep.yaml": deployments.apps "sleep" not found
    '
    ```
    ```bash
    kubectl delete -f todo-list/
    :'
    service "todo-db" deleted
    deployment.apps "todo-db" deleted
    deployment.apps "todo-web" deleted
    service "todo-web-test" deleted
    deployment.apps "todo-web-test" deleted
    service "todo-web" deleted
    Error from server (NotFound): error when deleting "todo-list/todo-web-dev-no-logging.yaml": deployments.apps "todo-web" not found
    Error from server (NotFound): error when deleting "todo-list/todo-web-dev.yaml": deployments.apps "todo-web" not found
    Error from server (NotFound): error when deleting "todo-list/todo-web.yaml": deployments.apps "todo-web" not found
    '
    ```
    ```bash
    kubectl delete -f todo-list/configMaps/
    :'
    configmap "todo-web-config-dev" deleted
    configmap "todo-web-config-test" deleted
    Error from server (NotFound): error when deleting "todo-list/configMaps/todo-web-config-dev.yaml": configmaps "todo-web-config-dev" not found
    '
    ```
    ```bash
    kubectl delete -f todo-list/secrets/
    :'
    secret "todo-db-secret-test" deleted
    secret "todo-web-secret-test" deleted
    Error from server (NotFound): error when deleting "todo-list/secrets/todo-db-secret-test.yaml": secrets "todo-db-secret-test" not found
    '
    ```