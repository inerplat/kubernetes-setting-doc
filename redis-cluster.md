# Redis cluster 설정
Goal: redis 네임스페이스에 master3-slave3 구조의 레디스 세팅, Volume은 로컬디렉터리를 마운트한 PV를 사용


> 검증된 helm중에 arm/v8이 호환되는 차트가 없다... 직접 yaml을 작성하여 클러스터링하는 내용을 다룬다
>
> 참고자료: https://rancher.com/blog/2019/deploying-redis-cluster/


### 0. namespace 생성
```
kubectl create namespace redis
```

### 1. redis pod에 연결할 PV생성
-  총 6개의 볼륨이 필요하다, 5Gi의 PV를 생성하는 yaml
-  실제 데이터의 저장은 PV가 생성된 워커노드의 /mnt/redis-cluster-pv-*에서 확인 가능하다
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis-cluster-pv-0
spec:
  storageClassName: ""
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/redis-cluster-pv-0
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis-cluster-pv-1
spec:
  storageClassName: ""
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/redis-cluster-pv-1
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis-cluster-pv-2
spec:
  storageClassName: ""
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/redis-cluster-pv-2
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis-cluster-pv-3
spec:
  storageClassName: ""
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/redis-cluster-pv-3
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis-cluster-pv-4
spec:
  storageClassName: ""
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/redis-cluster-pv-4
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: redis-cluster-pv-5
spec:
  storageClassName: ""
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt/redis-cluster-pv-5
```

### 2. ConfigMap 작성
> 그대로 사용하면 password가 1234로 설정된다
>
> appendonly옵션으로 pv에 백업이 될수있도록 설정함, dump.rdb를 사용하고 싶으면 제거가 필요
```

apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-cluster
  namespace: redis
data:
  update-node.sh: |
    #!/bin/sh
    REDIS_NODES="/data/nodes.conf"
    sed -i -e "/myself/ s/[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}/${POD_IP}/" ${REDIS_NODES}
    exec "$@"
  redis.conf: |+
    cluster-enabled yes
    cluster-require-full-coverage no
    cluster-node-timeout 15000
    cluster-config-file /data/nodes.conf
    cluster-migration-barrier 1
    appendonly yes
    protected-mode no
    requirepass 1234
```

### 3. StatefulSet 작성
```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
  namespace: redis
spec:
  serviceName: redis-cluster
  replicas: 6
  selector:
    matchLabels:
      app: redis-cluster
  template:
    metadata:
      labels:
        app: redis-cluster
    spec:
      containers:
      - name: redis
        image: redis:6.2.5
        ports:
        - containerPort: 6379
          name: client
        - containerPort: 16379
          name: gossip
        command: ["/conf/update-node.sh", "redis-server", "/conf/redis.conf"]
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        volumeMounts:
        - name: conf
          mountPath: /conf
          readOnly: false
        - name: data
          mountPath: /data
          readOnly: false
      volumes:
      - name: conf
        configMap:
          name: redis-cluster
          defaultMode: 0755
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 5Gi
```

### 4. svc 작성
> 클러스터 외부에서 redis 사용이 필요한 경우 아래처럼 NodePort를 적용
>
> 내부에서만 사용하면 ClusterIP로 작성해도 무방하다
```
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster
  namespace: redis
spec:
  type: NodePort
  ports:
  - port: 6379
    targetPort: 6379
    name: client
    nodePort: 30379
  - port: 16379
    targetPort: 16379
    name: gossip
    nodePort: 31379
  selector:
    app: redis-cluster
```

### 5. pvc 바인드 확인
```
$ kubectl get pvc -n redis
NAME                   STATUS   VOLUME               CAPACITY   ACCESS MODES   STORAGECLASS   AGE
data-redis-cluster-0   Bound    redis-cluster-pv-0   5Gi        RWO                           15s
data-redis-cluster-1   Bound    redis-cluster-pv-1   5Gi        RWO                           13s
data-redis-cluster-2   Bound    redis-cluster-pv-2   5Gi        RWO                           10s
data-redis-cluster-3   Bound    redis-cluster-pv-3   5Gi        RWO                           7s
data-redis-cluster-4   Bound    redis-cluster-pv-5   5Gi        RWO                           3s
data-redis-cluster-5   Bound    redis-cluster-pv-4   5Gi        RWO                           1s
```
> {volumeClaimTemplates로 배포한 pvc}-{pv}랑 이름 이쁘게 바인딩하는법 아시는분 알려주세요...

### 6. pod의 CNI IP 확인
```
$ kubectl get pod -n redis -o wide
NAME              READY   STATUS    RESTARTS   AGE     IP            NODE                 
redis-cluster-0   1/1     Running   0          3m28s   10.244.5.45   worker4-ubuntu-seoul 
redis-cluster-1   1/1     Running   0          3m26s   10.244.5.46   worker4-ubuntu-seoul
redis-cluster-2   1/1     Running   0          3m23s   10.244.5.47   worker4-ubuntu-seoul
redis-cluster-3   1/1     Running   0          3m20s   10.244.1.26   worker1-ubuntu-seoul
redis-cluster-4   1/1     Running   0          3m16s   10.244.5.48   worker4-ubuntu-seoul
redis-cluster-5   1/1     Running   0          3m14s   10.244.3.42   worker3-ubuntu-seoul
```


### 7. redis cluster 생성
```
# 6에서 확인한 IP:PORT가 5개 등장하는지 확인
echo $(echo $(kubectl get pods -n redis -l app=redis-cluster -o jsonpath='{range.items[*]}{.status.podIP}:6379 ') |  sed s/':\w*$'//)
```
```
# cluster 설정
kubectl exec -n redis -it redis-cluster-0 -- redis-cli -a '1234' --cluster create --cluster-replicas 1 $(echo $(kubectl get pods -n redis -l app=redis-cluster -o jsonpath='{range.items[*]}{.status.podIP}:6379 ') |  sed s/':\w*$'//)
```
> redis-cli -a 옵션에 본인의 패스워드를 입력하면 된다(위의 명령어에서는 1234)
>
> yes를 입력하면 join이 완료된다


### 8. 클러스터 정보 확인
```
kubectl exec -n redis -it redis-cluster-0 -- redis-cli cluster info
```
```
for x in $(seq 0 5); do echo "redis-cluster-$x"; kubectl exec -n redis redis-cluster-$x -- redis-cli role; echo; done
```

### 9. redis-cli로 외부 접속 확인
```
redis-cli -h 152.70.251.40 -p <NodePort> -a '1234'
```

### extra. 생성한 redis를 삭제하고 싶을때
> yaml로 delete를 실행해도 pvc와 워커노드에 생성된 로컬 디렉터리는 여전히 남아있다
```
# StatefulSet, Svc, ConfigMap 삭제
kubectl -f deploy.yaml
```
```
# redis 네임스페이스의 pvc전체 삭제
kubectl delete pvc -n redis --all
```
```
# pv 삭제
kubectl delete -f pv.yaml
```
- 이후에 각 워커노드의 /mnt/*에 생성되었던 레디스 디렉터리를 삭제하면 된다
