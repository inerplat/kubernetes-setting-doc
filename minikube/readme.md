# minikube 튜토리얼

## 1. 시작하기

Mac OS ([hyperkit](https://minikube.sigs.k8s.io/docs/drivers/hyperkit/))
```
minikube start --driver hyperkit --nodes 2
```
Windows ([HyperV](https://minikube.sigs.k8s.io/docs/drivers/hyperv/))
```
minikube start --driver hyperv --nodes 2
```

## 2. 노드확인

```
$ minikube node list

minikube        172.17.37.178
minikube-m02    172.17.40.221
```

```
$ kubectl get no
NAME           STATUS   ROLES                  AGE   VERSION
minikube       Ready    control-plane,master   12m   v1.23.3
minikube-m02   Ready    <none>                 10m   v1.23.3
```

## 3. 로컬 kubectl에 권한 부여하기

- 실제 환경에서는 `cluster-admin` 보다 낮은 권한을 권장합니다
- linux [(링크)](https://github.com/inerplat/kubernetes-setting-doc/blob/main/create_serviceaccount.md)
- powershell [(링크)](https://github.com/inerplat/kubernetes-setting-doc/blob/main/create_serviceaccount_powershell.md)

## 4. [대시보드](https://github.com/kubernetes/dashboard) 사용하기
(애드온을 사용하지 않는 방법입니다)
1. 대시보드 설치 (v2.5.0)
   ```
   kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.5.0/aio/deploy/recommended.yaml
   ```
2. 대시보드 접속
  1. svc/kubernetes-dashboard의 443포트 트래픽을 localhost:8080으로 포워딩
     ```
     kubectl port-forward -n kubernetes-dashboard svc/kubernetes-dashboard 8080:443
     ```
  2. 대시보드 접속([https://localhost:8080](https://localhost:8080))
  3. kubeconfig 선택 후 `~/.kube/config` 파일 업로드 (혹은 (3)에서 획득한 토큰을 붙여놓기)

## 5. Nginx 어플리케이션 배포
1. deployment.yaml
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx
      labels:
        app: nginx
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginx:latest
            ports:
            - containerPort: 80
    ```
2. service.yaml
    ```yaml
    apiVersion: v1
      kind: Service
    metadata:
      name: nginx-svc
    spec:
      selector:
        app: nginx
      ports:
      - protocol: TCP
        port: 8080
        targetPort: 80
        nodePort: 30080
    ```

## 6. 트래픽 분산 직접 확인해보기
1. pod 이름 확인
```
$ kubectl get pod
NAME                    READY   STATUS    RESTARTS   AGE
nginx-8d545c96d-j5cnp   1/1     Running   0          39s
nginx-8d545c96d-q4zns   1/1     Running   0          39s
```
2. pod에 접속하여 파일 수정하기
```
$ kubectl exec -it nginx-8d545c96d-j5cnp -- /bin/bash

root@nginx-8d545c96d-j5cnp:/# cd /usr/share/nginx/html
root@nginx-8d545c96d-j5cnp:/# echo "pod1" > index.html
```

```
$ kubectl exec -it nginx-8d545c96d-q4zns -- /bin/bash

root@nginx-8d545c96d-j5cnp:/# cd /usr/share/nginx/html
root@nginx-8d545c96d-j5cnp:/# echo "pod2" > index.html
```