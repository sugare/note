
kind | apiVersion
---|---
Pod | v1
Namespace | v1
Service | v1
Secret | v1
ReplicationController | v1
Service Account | v1
PersistentVolume | v1
PersistentVolumeClaim | v1
ResourceQuota | v1
ConfigMap | v1
Deployment | extensions/v1beta1
DaemonSet | extensions/v1beta1
ReplicaSet | extensions/v1beta1
Ingress | extensions/v1beta1
ThirdPartyResource | extensions/v1beta1
StatefulSet | apps/v1beta1
Job | batch/v1 
CronJob | batch/v2alpha1
HorizontalPodAutoscaler | autoscaling/v2alpha1
NetworkPolicy | networking.k8s.io/v1
PodPreset | settings.k8s.io/v1alpha1
CustomResourceDefinition | apiextensions.k8s.io/v1beta1



### Namespace
Namespace 比较简单，可直接通过kubecl直接创建
```
kubectl create namespace my-namespace
```
```
# vim my-namespace.yaml
kind: Namespace
apiVersion: v1
metadata:
  name: my-namespace
```


### Pod
常用的pod yaml参数(根据需求修改相关参数)
```
kind: Pod
apiVersion: v1
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: [Always,Never,IfNotPresent]
    env:
    - name: REDIS_HOST
      value: redis
    - name: REDIS_PORT
      value: "6379"
    ports:
    - containerPort: 80
    volumeMounts:
    - name: nginx-www
      mountPath: /usr/share/nginx/html
    resources:
      requests:
        cpu: "300m"
        memory: "56Mi"
      limits:
        cpu: "500m"
        memory: "128Mi"
    livenessProbe:
      httpGet:
      path: /
      port: 80
      initialDelaySeconds: 15
      timeoutSeconds: 1
    readinessProbe:
      httpGet:
      path: /ping
      port: 80
      initialDelaySeconds: 5
      timeoutSeconds: 1
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Hello from the postStart handler > /usr/share/message"]
      preStop:
        exec:
          command: ["/usr/sbin/nginx","-s","quit"]
  nodeSelector:
    disktype: ssd
  volumes:
  - name: nginx-www
    hostPath:
      path: /data/www
  restartPolicy: Never
```

不常用的pod参数：

**使用主机的命名空间**
```
kind: Pod
apiVersion: v1
metadata:
  name: busybox1
  labels:
    name: busybox
spec:
  hostIPC: true
  hostPID: true
  hostNetwork: true
  containers:
  - name: busybox
    image: busybox
    command:
    - sleep
    - "3600"
    
```

**限制带宽**
```
kind: Pod
apiVersion: v1
metadata:
  name: qos
  annotations:
    kubernetes.io/ingress-bandwidth: 3M
    kubernetes.io/egress-bandwidth: 4M
spec:
  containers:
  - name: iperf3
    image: networkstatic/iperf3
    command:
    - iperf3
    - -s
```
### volume
**emptyDir**

Pod 被分配到Node上时候，会创建emptyDir，
只要Pod运行在Node上，emptyDir都会存在（容器挂掉不会导致emptyDir丢失数据） ，
但是如果Pod从Node上被删除（Pod被删除，或者Pod发生迁移） ，emptyDir也会被删
除，并且永久丢失。
```
kind: Pod
apiVersion: v1
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: redis
    volumeMounts:
    - name: redis-storage
      mountPath: /data/redis
  volumes:
  - name: redis-storage
    emptyDir: {}
```


**hostPath**

将pod所在的node目录挂载到容器中
```
kind: Pod
apiVersion: v1
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: www
      mountPath: /usr/share/nginx/html
  volumes:
  - name: www
    hostPath: 
      path: /data
```

**NFS**
```
# showmount -e 192.168.0.7
Export list for 192.168.0.7:
/usr/share/www 192.168.0.0/24

# vim mynfs.yaml
kind: Pod
apiVersion: v1
metadata:
  name: mynfs
spec:
  containers:
  - name: mynfs
    image: nginx
    volumeMounts:
    - name: nfs
      mountPath: /usr/share/nginx/html
  volumes:
  - name: nfs
    nfs:
      server: 192.168.0.7
      path: "/usr/share/www"

# 提示：挂载nfs时注意防火墙
```

### gitRepo

将git代码下拉到指定的容器路径中
```
kind: Pod
apiVersion: v1
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: git-volume
      mountPath: /usr/share/nginx/html
  volumes:
  - name: git-volume
    gitRepo:
      repository: "https://github.com/sugare/note.git"

# kubectl exec nginx /bin/ls /usr/share/nginx/html
note

# 提示：建议使用https，如果使用git需要加入revision（相当于拉取GitHub的key）
```

**gcePersistentDisk**
```
volumes:
- name: test-volume
  # This GCE PD must already exist.
  gcePersistentDisk:
    pdName: my-data-disk
    fsType: ext4
```
**awsElasticBlockStore**
```
volumes:
- name: test-volume
  # This AWS EBS volume must already exist.
  awsElasticBlockStore:
  volumeID: <volume-id>
  fsType: ext4
```
**ceph**
```
# cephfs.yaml
apiVersion: v1
kind: Pod
metadata:
  name: cephfs
spec:
  containers:
  - name: cephfs-rw
    image: kubernetes/pause
    volumeMounts:
    - mountPath: "/mnt/cephfs"
      name: cephfs
  volumes:
  - name: cephfs
    cephfs:
      monitors:
      - 10.16.154.78:6789
      - 10.16.154.82:6789
      - 10.16.154.83:6789
      # by default the path is /, but you can override and mount a specific path of the filesystem by using the path attribute
      # path: /some/path/in/side/cephfs 
      user: admin
      secretFile: "/etc/ceph/admin.secret"
      readOnly: true
      
      
# cephfs with secret yaml
apiVersion: v1
kind: Pod
metadata:
  name: cephfs2
spec:
  containers:
  - name: cephfs-rw
    image: kubernetes/pause
    volumeMounts:
    - mountPath: "/mnt/cephfs"
      name: cephfs
  volumes:
  - name: cephfs
    cephfs:
      monitors:
      - 10.16.154.78:6789
      - 10.16.154.82:6789
      - 10.16.154.83:6789
      user: admin
      secretRef:
        name: ceph-secret
      readOnly: true
```

**glusterfs**
```
# vim glusterfs-cluster.yaml 
kind: Service
apiVersion: v1
metadata:
  name: glusterfs-cluster
spec:
  ports:
  - port: 1
--- 
kind: Endpoints
apiVersion: v1
metadata:
  name: glusterfs-cluster
subsets:
- addresses:
  - ip: 10.240.106.152
  ports:
  - port: 1
- addresses:
  - ip: 10.240.79.157
  ports:
  - port: 1

```

**portworx**
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: <vol-id>
spec:
  capacity:
    storage: <size>Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  portworxVolume:
    volumeID: "<vol-id>"
    fsType:   "<fs-type>"
```

### PV and PVC

**pv和pvc是什么？pv和普通的volume有什么不同？**

普通的volume和直接使用它的pod是一种静态绑定的关系，在定义pod的文件里，定义了它使用的volume，我们无法单独创建一个volume，因为它不是一个独立的k8s资源。

而pv是一个独立的k8s资源对象，所以我们可以单独创建一个pv，它不和pod直接发生关系，而是通过PVC来实现绑定。pod只需定义pvc，至于后面与真正的volume对接都交给pvc来处理。
```
# cat test-pv.yaml 
kind: PersistentVolume
apiVersion: v1
metadata:
  name: pv-nfs
spec:
  capacity:
    storage: 500Mi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    server: 192.168.0.7
    path: /data

# kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
pv-nfs    500Mi      RWO            Recycle          Available                                      47s
```
```
# cat test-pvc.yaml 
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myclaim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi

```
```
# cat test-pvc.yaml 
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: myclaim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi

# cat test-pod.yaml 
kind: Pod
apiVersion: v1
metadata:
  name: mypod
spec:
  containers:
  - name: mynginx
    image: nginx
    volumeMounts:
    - name: mypd
      mountPath: /usr/share/nginx/html
  volumes:
  - name: mypd
    persistentVolumeClaim:
      claimName: myclaim
```
```
# kubectl get pvc
NAME      STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
myclaim   Bound     pv-nfs    500Mi      RWO                           5m
# kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM             STORAGECLASS   REASON    AGE
pv-nfs    500Mi      RWO            Recycle          Bound     default/myclaim                            2m
```

### StorageClass
StorageClass 可以动态配置存储，什么意思？在创建 pv 时，加上 storageClassName 参数，这样 pvc 在使用 pv 时，只需指明 storageClassName 即可，这样将相同的 storageClass 整合到一起，无需关注底层 pv 类型，进一步减少了用户用于关注底层存储提供者所需的精力。
```
[root@node1 ~]# cat task-pv/task-pv-volume.yaml 
kind: PersistentVolume
apiVersion: v1
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/tmp/data"

[root@node1 ~]# cat task-pv/task-pv-
task-pv-claim.yaml   task-pv-pod.yaml     task-pv-volume.yaml  
[root@node1 ~]# cat task-pv/task-pv-claim.yaml 
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
      
[root@node1 ~]# cat task-pv/task-pv-pod.yaml 
kind: Pod
apiVersion: v1
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
       claimName: task-pv-claim
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage

```


### Service
**对接内部**

Service对接内部的应用必须加上selector来指定要对接的应用pod，也可使用externalName来DNS内部的应用，不过externalName多用于对接集群外部的服务。

Service有四种类型，分别是 NodePort、LoadBalancer、ExternalName、ClusterIP

1. NodePort: 暴露一个主机端口，将该端口映射到Service的port，可直接通过 <NodeIP>:<NodePort> 来访问该Service
```
kind: Service
apiVersion: v1
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
    nodePort: 31313
  selector:
    app: nginx
```

**解释下 port targetPort nodePort的区别：**

- port是cluster IP上的端口，可以理解为该Service对Cluster内部开放的端口
- targetPort字面意思是目的端口，我们定义的Service仅仅提供一个访问入口，至于入口后的内容并不知道，必须和其他真正提供服务的应用进行对接，当Service和Pod对接的时候，必须指向要对接的端口，才能完成对接，上面yaml的意思是：==将该Service的80端口对接到标签 app=nginx 的Pod的80 端口上==。 
- nodePort只是在Service所在主机暴露的端口，供外部访问。



2. LoadBalancer: 在NodePort的基础上对接cloud provider的负载均衡
```
略
```

3. ExternalName: 将服务通过DNS CNAME记录方式转发到指定的域名
```
kind: Service
apiVersion: v1
metadata:
  name: nginx
  namespace: default
spec:
  type: ExternalName
  externalName: www.baidu.com
```
此时DNS服务会创建一个CNAME记录<service-name>.<namespace>.svc.cluster.local指向设置的 spec:externalName。

nginx.default.svc.cluster.local ==> www.baidu.com

4. ClusterIP: 默认类型，自动分配一个仅cluster内部可以访问的虚拟IP，仅内部可以访问有什么作用？举个例子：比如要搭建一个LNMP环境，只需将Nginx Service的NodePort暴露出来即可，后端的NMP可通过ClusterIP仅内部访问，无需暴露，安全。
```
kind: Service
apiVersion: v1
metadata:
  name: nginx
  labels:
    app: nginx
  namespace: default
spec:
  type: ClusterIP
  ports:
  - port: 80
    protocol: TCP
    target: 80
  selector:
    app: nginx
  
```

**对接集群外部的服务**
1. 通过自定义一个与Service同名的endpoint（IP:Port）在endpoint中设置外部服务的IP:Port
```
kind: Service
apiVersion: v1
metadata:
  name: nginx
spec:
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
---
kind: Endpoints
apiVersion: v1
metadata:
  name: nginx
subsets:
- address:
  - ip: 1.2.3.4
  ports:
  - port: 80
```


2. 通过DNS转发，指定externalName为外部的域名即可。
```
kind: Service
apiVersion: v1
metadata:
  name: nginx
  namespace: default
spec:
  type: ExternalName
  externalName: www.baidu.com
```

### Ingress
Ingress yaml模板
```
kind: Ingress
apiVersion: extensions/v1beta1
metadata:
  name: test-ingress
spec:
  rules:
  - http:
    paths:
    - path: /testpath
      backend:
        serviceName: test
        servicePort: 80
```
根据不同路径对应不同服务
```
kind: Ingress
apiVersion: extensions/v1beta1
metadata:
  name: test
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: s1
          servicePort: 80
      - path: /bar
        backend:
          serviceName: s2
          servicePort: 80
```

基于域名的虚拟主机
```
kind: Ingress
apiVersion: extensions/v1beta1
metadata:
  name: test
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - backend:
          serviceName: s1
          servicePort: 80
  - host: bar.foo.com
    http:
      paths:
      - backend:
          serviceName: s2
          servicePort: 80
```
### Deployment
简单的yaml模板
```
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```
Deployment 的yaml文件中只是定义了目标状态，当kubectl create -f 时，Deployment controller就会将Pod和Replica Set的实际状态改变到所定义的目标状态，根据yaml文件中所定义的 template来创建Pod，既然是Pod template，则Deployment yaml文件中 template后内容和定义Pod的yaml一样，具体可参考Pod yaml模板


扩容
```
kubectl scale deployment nginx-deployment --replicas 10
```
更新
```
kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1
```
回滚
```
kubectl rollout undo deployment/nginx-deployment
```



### ConfigMap
**创建ConfigMap**

通过yaml创建
```
kind: ConfigMap
apiVersion: v1
metadata:
  name: test-config
  namespace: default
data:
  special.how: very
  special.type: charm
```

通过命令创建
```
# kubectl create configmap special-config --from-literal=special.how=
very --from-literal=special.type=charm
# kubectl create configmap env-config --from-literal=log_level=INFO
```
**ConfigMap使用**

ConfigMap可以通过三种方式在Pod中使用，三种分别方式为：设置环境变量、设置容器命令行参数以及在Volume中直接挂载文件或目录




用作环境变量
```
kind: Pod
apiVersion: v1
metadata:
  name: test-pod
spec:
  containers:
  - name: test-container
    image: busybox
    command: [ "/bin/sh", "-c", "env" ]
    env:
    - name: SPECIAL_LEVEL_KEY
      valueFrom:
        configMapKeyRef:
          name: special-config
          key: special.how
    - name: SPECIAL_TYPE_KEY
      valueFrom:
        configMapKeyRef:
          name: special-config
          key: special.type
    envFrom:
      - configMapRef:
        name: env-config
  restartPolicy: Never
```
用做命令行参数
```
kind: Pod
apiVersion: v1
metadata:
  name: dapi-test-pod
spec:
  containers:
  - name: test-container
    image: busybox
    command: [ "/bin/sh", "-c", "echo $(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY)" ]
    env:
    - name: SPECIAL_LEVEL_KEY
      valueFrom:
        configMapKeyRef:
          name: special-config
          key: special.how
    - name: SPECIAL_TYPE_KEY
      valueFrom:
        configMapKeyRef:
          name: special-config
          key: special.type
  restartPolicy: Never
```
挂载使用
```
apiVersion: v1
kind: Pod
metadata:
  name: vol-test-pod
spec:
  containers:
  - name: test-container
    image: gcr.io/google_containers/busybox
    command: [ "/bin/sh", "-c", "cat /etc/config/special.how" ]
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: special-config
  restartPolicy: Never
```

### Secret
Secret有三种类型：
- Service Account：用来访问Kubernetes API，由Kubernetes自动创建，并且会自动挂载到Pod的 /run/secrets/kubernetes.io/serviceaccount 目录中；
- Opaque：base64编码格式的Secret，用来存储密码、密钥等；
- kubernetes.io/dockerconfigjson ：用来存储私有docker registry的认证信息。

**Opaque Secret**
Opaque类型的数据是一个map类型，要求value是base64编码
```
$ echo -n "admin" | base64
YWRtaW4=
$ echo -n "1f2d1e2e67df" | base64
MWYyZDFlMmU2N2Rm
```
```
# secret.yaml
kind: Secret
apiVersion: v1
metadata:
  name: mysecret
type: Opaque
data:
  password: MWYyZDFlMmU2N2Rm
  username: YWRtaW4=
```

Secret引用方式有两种：
- 以Volume方式
- 以环境变量方式

**将secret挂载到volume中**
```
kind: Pod
apiVersion: v1
metadata:
  labels:
    name: db
  name: db
spec:
  volumes:
  - name: secrets
    secret:
      secretName: mysecret
  containers:
  - image: gcr.io/my_project_id/pg:v1
    name: db
    volumeMounts:
    - name: secrets
      mountPath: "/etc/secrets"
      readOnly: true
    ports:
    - name: cp
      containerPort: 5432
      hostPort: 5432
```

**将secret 导出到环境变量中**
```
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: wordpress-deployment
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: wordpress
        visualize: "true"
    spec:
      containers:
      - name: "wordpress"
        image: "wordpress"
        ports:
        - containerPort: 80
        env: 
        - name: WORDPRESS_DB_USER
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: username
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: password
```










### Job
处理短暂的一次性任务，仅执行一次的任务,Job Controller负责根据Job Spec创建Pod，并持续监控Pod的状态，直至其成功结束。如果失败，则根据restartPolicy（只支持OnFailure和Never，不支持Always）决定是否创建新的Pod再次重试任务。
```
kind: Job
apiVersion: batch/v1
metadata：
  name: test-job
spec:
  completions: 3
  template:
    metadata:
      name: test-job
    spec:
      containers:
      - name: nginx
        image: nginx
        command: ["cat", "/usr/share/nginx/html/index"]
      restartPolicy: Never
# 文件中 completions表示执行3次job，如果执行一次可不写该参数。
```


### CronJob
CronJob即定时任务，类似于Linux系统的crontab，在指定的时间周期运行指定的任务。
```
kind: CronJob
apiVersion: batch/v2alpha1
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```






















## 附录：
https://github.com/kubernetes/examples/blob/master/staging/volumes/cephfs/README.md
https://github.com/kubernetes/examples/blob/master/staging/volumes/glusterfs/README.md
http://blog.csdn.net/liukuan73/article/details/60089305








