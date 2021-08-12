## 대시보드
- 배포
    ```
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.2.0/aio/deploy/recommended.yaml
    ```
- 토큰 발급
  - 위의 yaml을 그대로 배포하면 kubernetes-dashboard 라는 롤이 생성된다
  - 해당 롤을 사용할 serviceaccount에 바인딩하면 된다
    ```
        kubectl create clusterrolebinding <바인딩이름> --clusterrole=kubernetes-dashboard --serviceaccount=<namespace:account>
    ```
  - admin 권한을 가지고 있는 계정을 사용하면 위의 바인딩작업이 필요 없다
    - dashboard를 상시로 노출해야하는 환경이면 권장되지 않는다.

- 프록시
  - 위의 yaml을 그대로 배포하면 ClusterIP로 Service가 생성되기 때문에, 로컬에서 접속하기 위해서는 프록시가 필요하다
    ```
    kubectl port-forward -n kubernetes-dashboard service/kubernetes-dashboard 8080:443
    ```
  - 프록시를 사용하고싶지 않다면, Service를 NodePort나 Ingress로 변경하여 노출하면 된다
    - 강력한 권한이 들어있는 계정이 노출되면 매우 치명적이므로, 필요할 때만 프록시로 노출하는것이 권장된다

