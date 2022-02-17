# 로컬에서 kubectl을 사용하기 위한 ServiceAccount 발급

> kubectl은 로컬 개발환경에서 사용하기 위해서는 .kube/config를 이용하는것 보다는, ServiceAccount에 ClusterRole를 바인딩해서 사용하는것이 권장된다.

1. 계정 생성 (ex. develop이름의 계정 생성)
    ```
    export ACCOUNT=develop
    kubectl create serviceaccount $ACCOUNT
    ```

2. 계정 토큰명 확인
    ```
    # create serviceaccount가 완료된 뒤에 실행
    export ACCOUNT_TOKEN_NAME=$(kubectl get secret | grep $ACCOUNT | cut -d " " -f1)
    echo $ACCOUNT_TOKEN_NAME
    ```

3. 계정에 클러스터 롤 바인딩 (ex. 계정에 cluster-admin롤 바인딩)
    - kubeadm
        ```
         kubectl create clusterrolebinding $ACCOUNT_TOKEN_NAME-binding --clusterrole=cluster-admin --serviceaccount=default:$ACCOUNT
        ```

4. 발급된 토큰 확인
    ```
    export ACCOUNT_TOKEN=$(kubectl get secret $ACCOUNT_TOKEN_NAME -o jsonpath="{.data.token}" | base64 -d)
    echo $ACCOUNT_TOKEN
    ```


## 로컬 kubectl에 cluster 및 context 설정

- 아래의 명령어는 스크립트로 만들면 개발환경에서 편리하게 클러스터를 변경할 수 있다
    - microk8s는 기본 클러스터이름으로 `microk8s-cluster`, kubeadm은 `kubernetes`를 사용한다. 클러스터 이름을 변경했다면 수정이 필요하다
    ```
    # 클러스터에서 발급한 토큰을 사용
    kubectl config set-credentials develop --token=[클러스터에서 가져온 토큰]

    # 클러스터 이름 설정 (ex. 클러스터 이름이 kubernetes인 경우)
    kubectl config set-cluster kubernetes --insecure-skip-tls-verify=true --server=https://[SERVER_IP_OR_DOMAIN]:6443

    # context에 user, cluster 설정 및 적용
    kubectl config set-context develop --cluster=kubernetes --user=develop
    kubectl config use-context develop
    ```

- 클러스터와 연결이 되었는지 확인
    ```
    $ kubectl get no

    NAME                   STATUS   ROLES                  AGE     VERSION
    master-ubuntu-seoul    Ready    control-plane,master   6h49m   v1.21.3
    worker1-ubuntu-seoul   Ready    <none>                 4h58m   v1.21.4
    worker2-ubuntu-seoul   Ready    <none>                 6h46m   v1.21.3
    worker3-ubuntu-seoul   Ready    <none>                 6h46m   v1.21.3
    ```
