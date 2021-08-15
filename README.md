# kubernetes-setting-doc

> 이 문서는 Oracle Arm Ampere A1, Ubuntu 20.04를 기준으로 작성되었습니다.
>
> OCI에서 제공되는 K8S가 아닌, 직접 클러스터를 구축하며 경험한 내용을 다루고 있습니다.

## ※ 주의 사항
 - Oracle의 Amere A1 Free tier로 생성가능한 인스턴스는 linux/arm/v8 아키텍쳐를 사용합니다
 - pod log에서 아래와 같은 에러는 높은확률로 다른 아키텍쳐의 이미지를 사용해 발생한 문제입니다
    ```
    standard_init_linux.go:228: exec user process caused: exec format error
    ```

## Microk8s
- Microk8s를 이용한 클러스터 구축 [(Link)](https://github.com/inerplat/kubernetes-setting-doc/blob/main/microk8s/readme.md)

## Kubeadm
- Kubeadm을 이용한 클러스터 구축 [(Link)](https://github.com/inerplat/kubernetes-setting-doc/blob/main/kubeadm/readme.md)
  - VPN을 이용한 멀티노드 클러스터 구축 [(Link)](https://github.com/inerplat/kubernetes-setting-doc/blob/main/kubeadm/multi-node-vpn.md)
- Kubeadm join에 필요한 토큰 발급 [(Link)](https://github.com/inerplat/kubernetes-setting-doc/blob/main/kubeadm/kubeadm-join.md)

## 공통
- Ingress를 이용한 SSL인증서 발급/등록 [(Link)](https://github.com/inerplat/kubernetes-setting-doc/blob/main/ssl_certify_with_ingress.md)
- 로컬에서 접속하는 ServiceAccount 만들기 [(Link)](https://github.com/inerplat/kubernetes-setting-doc/blob/main/create_serviceaccount.md)
- 대시보드 적용 [(Link)](https://github.com/inerplat/kubernetes-setting-doc/blob/main/dashboard.md)
- Redis Cluster 만들기 [(Link)](https://github.com/inerplat/kubernetes-setting-doc/blob/main/redis-cluster.md)
