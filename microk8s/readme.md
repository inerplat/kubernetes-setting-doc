# MicroK8s로 단일 노드 클러스터 구축하기

* 이 문서는 Oracle Arm Ampere A1, Ubuntu 20.04를 기준으로 작성되었습니다.
* 작성일 기준(2021-08-01) 최신버전에서 작동하는 플로우입니다. 패키지 업데이트에 따라 변동이 생길 수 있습니다.

1. Docker 설치 
    ```
    sudo apt-get update
    
    sudo apt-get install \
        apt-transport-https \
        ca-certificates \
        curl \
        gnupg \
        lsb-release -y
        
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
    
    echo \
    "deb [arch=arm64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
    $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    
    sudo apt-get update
    sudo apt-get install docker-ce docker-ce-cli containerd.io -y
    ```

2. MicroK8s 설치
    ```
    sudo snap install microk8s --classic
    sudo usermod -a -G microk8s ubuntu
    sudo chown -f -R ubuntu ~/.kube
    newgrp microk8s
    ```

3. "microk8s kubectl" alias 등록 (선택)
    microk8s kubectl을 kubectl로 등록, 필수는 아니지만 이후의 명령어은 kubectl로 진행됨
    ```
    echo "alias kubectl='microk8s kubectl'" >> .bashrc
    source .bashrc
    ```

4. MicroK8s 상태 확인
    ```
    microk8s status --wait-ready
    kubectl get node
    ```
    - output
        ```
        microk8s is running
        high-availability: no
        datastore master nodes: 127.0.0.1:19001
        datastore standby nodes: none
        ```
        ```
        NAME    STATUS   ROLES    AGE     VERSION
        test2   Ready    <none>   2m43s   v1.21.3-3+6343a564e351b0
        ```

5. 사용할 서비스 enable
    ex)
    ```
    microk8s enable dashboard 
    microk8s enable dns 
    microk8s enable registry 
    microk8s enable istio 
    microk8s enable ingress
    microk8s kubectl get all --all-namespaces
    ```

6. 방화벽 오픈
    ```
    sudo iptables -I INPUT -p tcp -m tcp --dport 80 -j ACCEPT
    sudo iptables -I INPUT -p tcp -m tcp --dport 443 -j ACCEPT
    sudo iptables -I INPUT -p tcp -m tcp --dport 10443 -j ACCEPT
    sudo iptables -I INPUT -p tcp -m tcp --dport 16443 -j ACCEPT
    sudo netfilter-persistent save
    sudo netfilter-persistent reload
    ```

7. 로컬에서 접속하기위한 서비스계정 만들기
    ```
    export ACCOUNT=develop
    kubectl create serviceaccount $ACCOUNT
    sleep 7 && export ACCOUNT_TOKEN_NAME=$(kubectl get secret | grep $ACCOUNT | cut -d " " -f1)
    kubectl create rolebinding $ACCOUNT_TOKEN_NAME-binding --clusterrole=admin --serviceaccount=default:$ACCOUNT
    export ACCOUNT_TOKEN=$(kubectl get secret $ACCOUNT_TOKEN_NAME -o jsonpath="{.data.token}" | base64 -d)
    ```

8. 로컬 kubectl 설정
- 클러스터 환경
    ```
    echo $ACCOUNT_TOKEN
    ```

- 로컬
    ```
    kubectl config set-credentials develop --token=[클러스터에서 가져온 토큰]
    kubectl config set-cluster microk8s-cluster --insecure-skip-tls-verify=true --server=https://[SERVER_IP_OR_DOMAIN]:16443
    kubectl config set-context develop --cluster=microk8s-cluster --user=develop
    kubectl config use-context develop

    kubectl get node
    ```

