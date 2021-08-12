# kubeadm-setting-doc
- 이 문서는 Oracle Arm Ampere A1, Ubuntu 20.04를 기준으로 작성되었습니다.
- 작성일 기준(2021-08-09) 최신버전에서 작동하는 플로우입니다. 패키지 업데이트에 따라 변동이 생길 수 있습니다.


## 공통

1. 도커 설치
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
2. 도커 드라이버를 systemd를 사용하도록 설정
    ```
    su
    # 패스워드 입력 (비밀번호가 설정되어 있지 않다면 sudo passwd로 설정)

    cat > /etc/docker/daemon.json <<EOF
    {
      "exec-opts": ["native.cgroupdriver=systemd"],
      "log-driver": "json-file",
      "log-opts": {
        "max-size": "100m"
      },
      "storage-driver": "overlay2"
    }
    EOF

    sudo mkdir -p /etc/systemd/system/docker.service.d
    sudo systemctl daemon-reload
    sudo systemctl restart docker

    exit
    ```

3. swap 메모리 제거
    ```
    sudo swapoff -a
    sudo sed -i '2s/^/#/' /etc/fstab
    ```

4. kubeadm, kubelet, kubectl 설치
    ```
    sudo apt update
    curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

    cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
    deb https://apt.kubernetes.io/ kubernetes-xenial main
    EOF

    sudo apt update

    sudo apt install -y kubelet kubeadm kubectl

    sudo apt-mark hold kubelet kubeadm kubectl

    kubeadm version && kubelet --version && kubectl version

    ```
## Master Node
1. 네트워크 설정
- 포트 오픈
    > Flannel로 pod간 통신을 하기 위해서는 UDP 8285, 8472 포트 오픈이 필요하다
    
    ```
    sudo iptables -I INPUT -p tcp -m tcp --dport 80 -j ACCEPT
    sudo iptables -I INPUT -p tcp -m tcp --dport 443 -j ACCEPT
    sudo iptables -I INPUT -p udp -m udp --dport 8285 -j ACCEPT
    sudo iptables -I INPUT -p udp -m udp --dport 8472 -j ACCEPT
    sudo iptables -I INPUT -p tcp -m multiport --dports 6443,2379:2380,10250,10252 -j ACCEPT
    ```
- 네트워크 nat설정 (인스턴스에서 Public IP를 인식할 수 없는경우만)
    > 오라클 클라우드는 [인스턴스]-[subnet ingress]-[Public IP]-[인터넷] path로 네트워크를 만들어준다.
    > 
    > 인스턴스 내부에서는 Public IP를 인식할 수 없으므로, `apiserver-advertise-addres`에 Public IP를 넣으면 kubeadm이 설치되지 않는다.
    >
    > 따라서 Private IP로 advertise를 하고, 다른 인스턴스들이 Private IP를 입력해도 Master Node를 찾을수 있도록 설정이 필요하다.
    ```
    sudo iptables -t nat -A OUTPUT -d <worker private ip> -j DNAT --to-destination <worker public ip>
    ```

2. iptables 저장 및 리로드
    ```
    sudo netfilter-persistent save
    sudo netfilter-persistent reload
    ```

3. kubeadm init (Flannel로 10.244.0.0/16 대역으로 IP할당을 원하는 경우)
    > init이 완료되고 나온 join 정보를 복사해서 Worker Node설정할때 붙여놓기를 해야한다.
    > 
    > Worker노드에 작업하기 전에 (4), (5)과정은 완료하고 넘어가야 한다
- ifconfig로 Public IP를 조회할 수 있는 경우 
    ```
    sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<Master Public IP>
    ```
- Public IP를 인식할 수 없는 경우
    ```
    sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<Master Private IP> --apiserver-cert-extra-sans=<Master Public IP>,<Master Private IP>
    ```


4. admin.conf 복사 및 sudo 없이 kubectl을 실행할수 있도록 변경
    ```
    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    sudo chmod 600 /etc/kubernetes/admin.conf
    echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> .bashrc
    source .bashrc
    ```

5. Flannel 배포
    ```
    # master
    sudo sysctl net.bridge.bridge-nf-call-iptables=1
    kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
    ```
    ```
    # worker
    sudo sysctl net.bridge.bridge-nf-call-iptables=1
    ```

## Worker Node

1. 네트워크 설정
- 포트 오픈
    ```
    sudo iptables -I INPUT -p tcp -m tcp --dport 80 -j ACCEPT
    sudo iptables -I INPUT -p tcp -m tcp --dport 443 -j ACCEPT
    sudo iptables -I INPUT -p udp -m udp --dport 8285 -j ACCEPT
    sudo iptables -I INPUT -p udp -m udp --dport 8472 -j ACCEPT
    sudo iptables -I INPUT -p tcp -m multiport --dports 10250,30000:32767 -j ACCEPT
    ```
- 네트워크 nat설정 (인스턴스에서 Public IP를 인식할 수 없는경우만)
    ```
    sudo iptables -t nat -A OUTPUT -d <master private ip> -j DNAT --to-destination <master public ip>
    ```

2. iptables 저장 및 리로드
    ```
    sudo netfilter-persistent save
    sudo netfilter-persistent reload
    ```

3. kubeadm join
    ```
    sudo kubeadm join <master private ip>:6443 --token 토큰토큰토큰토큰 \
            --discovery-token-ca-cert-hash sha256:해시해시해시해시
    ```

