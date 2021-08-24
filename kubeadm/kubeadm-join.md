# kubeadm join에 필요한 토큰 발급(조회)

## kubeadm join 명령어
```
kubeadm join <master-host>:<master-kubelet-port> \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
```
- 마스터 노드(control-plane)에 조인하기 위해서는 토큰과 cert hash값이 필요하다
- kubeadm init을 할때 알려주는 토큰은 24시간 이후에 만료되기때문에, 나중에 join하기 위해서는 토큰과 해시값을 얻어서 join해야한다

## 기존에 발급한 토큰 조회
```
kubeadm token list
```
```
ubuntu@master-ubuntu-seoul:~$ kubeadm token list
TOKEN                     TTL         EXPIRES                USAGES                   DESCRIPTION 
1a2b3c4d.a2f1as23312123   23h         2021-08-15T10:42:50Z   authentication,signing   <none>                                                     
```
- EXPIRES가 지났다면 토큰을 재발급 받아야 한다

## 새로운 토큰 발급
```
kubeadm token create
```

## cert hash 조회
```
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```
