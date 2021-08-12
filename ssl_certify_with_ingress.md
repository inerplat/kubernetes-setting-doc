# Ingress를 이용한 인증서 발급 및 적용

## certbot을 이용한 인증서 발급
일반적인 환경에서 certbot의 manual로 인증을 하려면 80/443포트에서 운영중인 서비스의 변경이 필요하다.
Kubernetes에서는 Ingress를 사용하면 path이름을 기준으로 서비스 라우팅이 가능하다. (트래픽을 먼저 받기 때문에)

### certbot 설치
```
sudo snap install certbot --classic
```

### certbot 인증 명령어
```
sudo certbot certonly --manual -d '도메인'
```

이메일 및 정책동의를 입력하고나면 아래와 같은 내용이 출력되며 멈추게된다.
사용하던쉘은 유지한채, 새로운 쉘을 열어서 아래 과정을 수행하면 된다.
```
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Create a file containing just this data:

_HkidijU118L1Uh9DhddEBtEAZBhf3LhhByJXL23l3Bk.TTqZfag9Zr3fywBPckZRledcJe6C9BSvMvXEx79Amic

And make it available on your web server at this URL:

http://도메인/.well-known/acme-challenge/_Hk44dijU118L1Uh9DhddEBtEAZBhf3LhhByJXLjl3Bk

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue
```

### nginx deployment, service, ingress 설정
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  labels:
    app: nginx
spec:
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
        image: nginx:1.14.2
        ports:
        - containerPort: 80
        lifecycle:
          postStart:
            exec:
              command: ["/bin/sh", "-c", "echo '_HkidijU118L1Uh9DhddEBtEAZBhf3LhhByJXL23l3Bk.TTqZfag9Zr3fywBPckZRledcJe6C9BSvMvXEx79Amic' > /usr/share/nginx/html/index.html"]
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  type: ClusterIP
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    # kubernetes.io/ingress.class: public  # microk8s는 ingress.class를 public으로 명시해야 한다
spec:
  rules:
  - host: 도메인
    http:
      paths:
      - path: /.well-known/acme-challenge/_Hk44dijU118L1Uh9DhddEBtEAZBhf3LhhByJXLjl3Bk
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
```
- deployment가 배포된 이후에, index.html이 인증정보로 덮어씌워진다.
- ingress의 spec.rules.http.paths.path에 certbot에서 인증을 요청할 path을 적어주면 된다.
- 주의할점
  - microk8s에서는 디폴트로 ingress.class를 public를 사용한다. 따로 변경한게 없으면 ingress의 metadata.annotations에 명시가 필요하다.
  - rewrite-target옵션을 사용하면 `http://도메인/.well-known/acme-challenge/_Hk44dijU118L1Uh9DhddEBtEAZBhf3LhhByJXLjl3Bk`가 `http://nginx-svc/`로 path가 설정된다.

```
kubectl apply -f nginx.yaml
```
배포가 완료되면 정지된 인증을 이어서 진행하면 된다.

## tls secret을 생성하고, ingress에 적용하는법
만든 인증서를 ingress에 적용해, 서비스는 그대로 유지하며 ssl인증을 적용하는 방법이다.

### 인증서를 작업 디렉터리로 이동
```
sudo cp /etc/letsencrypt/live/193.122.106.23.nip.io/cert.pem .
sudo cp /etc/letsencrypt/live/193.122.106.23.nip.io/privkey.pem .
sudo chmod 644 cert.pem privkey.epm
```

### kubernetes secret에 인증서를 등록
```
kubectl create secret ssl nip --cert cert.pem --key privkey.pem
kubectl describe secret ssl
```

### 인그레스에 secret를 등록
ex) 위의 nginx에 ssl을 적용하는 예제
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    # kubernetes.io/ingress.class: public # microk8s는 ingress.class를 public으로 명시해야 한다
spec:
  tls:
  - hosts:
    - 본인의도메인
    secretName: ssl
  rules:
  - host: 본인의도메인
    http:
      paths:
      - path: /.well-known/acme-challenge/_HkidijU118L1Uh9DhddEBtEAZBhf3LhhByJXLjl3Bk
        pathType: Prefix
        backend:
          service:
            name: nginx-svc
            port:
              number: 80
```
ingress의 spec.tls에 본인의 도메인 정보와 secret으로 등록한 인증서의 이름을 적고 배포하면 반영이 된다.
