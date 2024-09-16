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
