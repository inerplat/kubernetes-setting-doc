# VPN을 이용한 멀티노드 클러스터 세팅

> 서로 다른 네트워크(계정)에 있는 인스턴스를 연결하는 법

> 이 문서는 Oracle Arm Ampere A1, Ubuntu 20.04를 기준으로 작성되었습니다.
>
> ingress-nginx는 작성일 기준으로 정식버전에서 `networking.k8s.io/v1beta1`를 사용하고 있으므로 kubelet v1.22 미만 버전을 사용해야합니다. (v1.22부터 deprecated)
>
> 작성일 이후 패키지 업데이트에 따라 변동이 생길 수 있습니다.

- 방화벽 해제
  - 방화벽을 VCN에서도 관리하므로, iptables를 전부 개방하고 대시보드에서 관리하는걸 선택
      ```
    sudo iptables -X
    sudo iptables -F
    sudo iptables -P INPUT ACCEPT
    sudo iptables -P FORWARD ACCEPT
    sudo iptables -P OUTPUT ACCEPT
    sudo netfilter-persistent save
    sudo netfilter-persistent reload
    ```
  - VCN 방화벽
    - `0.0.0.0/0`: `80`, `443`, `6443`(master), `30000-32768`(worker) 개방
    - master, worker간 [필수포트](https://kubernetes.io/ko/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)과 cni포트(종류마다 다르므로 잘 찾아봐야 한다) 개방
    - 보안을 희생하고 master-worker, worker-worker간 전부 허용하면 편하긴 하다

- VPN 설치
  - NAT으로 포워딩 하면 kubeadm join은 잘 되지만, CNI(Flannel, Callico, Cillium, weave로 시도해봄) 통신에서 문제가 생긴다.
  - VPN을 설치해서 같은 네트워크에 있는것처럼 인식을 시키는게 이후의 Ingress Control를 설치할때도 편하게 작업할 수 있다.
  - client-to-client가 서로 통신이 가능하도록 설정하면 된다

> 이후의 설명은 CNI를 Flannel로 사용하는 설정 방법이다
- kubeadm join할때 Public IP 알려주기
    ```
    # master
    sudo kubeadm init --pod-network-cidr=10.244.0.0/16 \
        --apiserver-advertise-address=<Master VPN IP> \
        --control-plane-endpoint=<Master Public IP> \
        --apiserver-cert-extra-sans=<Master Public IP>,<Master Private IP>,<VPN IP>
    ```
- Flannel 설치 후 네트워크 인터페이스를 vpn으로 사용하도록 설정
    ```
    # master
    kubectl edit daemonset kube-flannel-ds -n kube-system
    ```
    spec.template.spec.containers.args에 `--iface=tun0` 추가
    ```
    #...
      containers:
      - args:
        - --ip-masq
        - --kube-subnet-mgr
        - --iface=tun0
        command:
        - /opt/bin/flanneld
    #...
    ```
    저장하면 daemonset이 다시 시작된다(master, worker 자동 반영)
- kubelet이 Internal IP를 VPN IP로 사용하도록 설정 (master, worker 전체에 필요한 작업)
  
  1. kubelet 설정 편집
        ```
        sudo vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
        ```
  2. KUBELET_CONFIG_ARGS에 `--node-ip=<자신의 VPN IP>` 추가
        ```
        Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml --node-ip=10.8.0.6"
        ```
  3. kubelet 재시작
        ```
        sudo systemctl daemon-reload
        sudo systemctl restart kubelet
        ```
  4. 모든 노드에서 작업이 완료되면 InternalIP가 vpn으로 연결된 것을 확인할 수 있다.
        ```
        ubuntu@master-ubuntu-seoul:~$ kubectl get no -o wide
        
        NAME                   STATUS   ROLES                  AGE    VERSION   INTERNAL-IP  
        master-ubuntu-seoul    Ready    control-plane,master   3h5m   v1.21.3   10.8.0.1  
        worker1-ubuntu-seoul   Ready    <none>                 75m    v1.21.4   10.8.0.18   
        worker2-ubuntu-seoul   Ready    <none>                 3h2m   v1.21.3   10.8.0.6    
        worker3-ubuntu-seoul   Ready    <none>                 3h2m   v1.21.3   10.8.0.10  
        ```

- ingress-nginx 컨트롤러 설치
  > 무료계층계정 A1인스턴스에는 LB를 연결할수 없다...
  >
  > 편법으로 모든 노드에 ingress-nginx-controller pod을 DaemonSet으로 띄우고, host모드로 Public IP를 연결하도록 하면 80, 443포트를 NodePort처럼 이용할 수 있다.

  1. ingress-nginx를 NodePort로 설정하는 yaml파일 다운로드
      ```
      wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.48.1/deploy/static/provider/baremetal/deploy.yaml
      ```
  2. Deployment → DaemonSet 변경
      ```
      # ...
      apiVersion: apps/v1
      kind: DaemonSet
      metadata:
      labels:
          helm.sh/chart: ingress-nginx-3.34.0
          app.kubernetes.io/name: ingress-nginx
          app.kubernetes.io/instance: ingress-nginx
          app.kubernetes.io/version: 0.48.1
          app.kubernetes.io/managed-by: Helm
          app.kubernetes.io/component: controller
      name: ingress-nginx-controller
      namespace: ingress-nginx
      # ...
      ```
  3. 변경한 DaemonSet의 `spec.template.spec`에 `hostNetwork: true` 추가
      > Worker노드마다 ingress-nginx-controller가 생성되고, 각 컨트롤러는 host(Public IP)의 트래픽을 받아서 Ingress의 설정에 따라 Service로 전달할 수 있다.
      >
      > Worker노드의 IP들을 도메인의 A 레코드로 등록해서 로드밸런싱을 적용할 수 있다.

      ```
      # ...
      template:
        metadata:
            labels:
            app.kubernetes.io/name: ingress-nginx
            app.kubernetes.io/instance: ingress-nginx
            app.kubernetes.io/component: controller
        spec:
            hostNetwork: true
            dnsPolicy: ClusterFirst
            containers:
            - name: controller
      # ...
      ```
  4. 수정한 ingress-nginx 컨트롤러 배포
      ```
      kubectl apply -f deploy.yaml
      ```
      ```
      ubuntu@master-ubuntu-seoul:~$ kubectl get all -n ingress-nginx
      NAME                                       READY   STATUS      RESTARTS   AGE
      pod/ingress-nginx-admission-create-cbgvw   0/1     Completed   0          3h20m
      pod/ingress-nginx-admission-patch-kwrqv    0/1     Completed   1          3h20m
      pod/ingress-nginx-controller-gzczg         1/1     Running     0          94m
      pod/ingress-nginx-controller-nx4kl         1/1     Running     0          3h20m
      pod/ingress-nginx-controller-slv85         1/1     Running     0          3h20m

      NAME                                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
      service/ingress-nginx-controller             NodePort    10.102.50.123   <none>        80:30080/TCP,443:30443/TCP   3h20m
      service/ingress-nginx-controller-admission   ClusterIP   10.102.57.251   <none>        443/TCP                      3h20m

      NAME                                      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
      daemonset.apps/ingress-nginx-controller   3         3         3       3            3           kubernetes.io/os=linux   3h20m

      NAME                                       COMPLETIONS   DURATION   AGE
      job.batch/ingress-nginx-admission-create   1/1           2s         3h20m
      job.batch/ingress-nginx-admission-patch    1/1           3s         3h20m
     ```
- systemd nameserver 추가 (Oracle Linux 7.x)
  - `/run/systemd/resolve/resolv.conf`에 master node의 설정을 복사
