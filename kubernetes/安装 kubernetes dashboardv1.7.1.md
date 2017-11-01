### 0. 镜像

镜像名 | 版本号
---|---
sugare/kubernetes-dashboard-init-amd64 | v1.0.1
sugare/kubernetes-dashboard-amd64 | v1.7.1




### 1. 下载官方yaml
```
# curl -o kubernetes-dashboard.yaml https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

### 2. 修改源
```
# sed -i 's#gcr.io/google_containers#sugare#g' kubernetes-dashboard.yaml
```

### 3. 如果直接执行会报错
```
Events:
  Type     Reason                 Age   From               Message
  ----     ------                 ----  ----               -------
  Normal   Scheduled              1m    default-scheduler  Successfully assigned kubernetes-dashboard-569bfc6d47-xchtq to node1
  Normal   SuccessfulMountVolume  1m    kubelet, node1     MountVolume.SetUp succeeded for volume "tmp-volume"
  Normal   SuccessfulMountVolume  1m    kubelet, node1     MountVolume.SetUp succeeded for volume "kubernetes-dashboard-certs"
  Normal   SuccessfulMountVolume  1m    kubelet, node1     MountVolume.SetUp succeeded for volume "kubernetes-dashboard-token-f9vf2"
  Normal   Pulled                 1m    kubelet, node1     Container image "sugare/kubernetes-dashboard-init-amd64:v1.0.1" already present on machine
  Normal   Created                1m    kubelet, node1     Created container
  Normal   Started                1m    kubelet, node1     Started container
  Warning  FailedMount            1m    kubelet, node1     MountVolume.SetUp failed for volume "kubernetes-dashboard-certs" : secrets "kubernetes-dashboard-certs" not found
  Normal   Pulled                 1m    kubelet, node1     Container image "sugare/kubernetes-dashboard-amd64:v1.7.1" already present on machine
  Normal   Created                1m    kubelet, node1     Created container
  Normal   Started                1m    kubelet, node1     Started container
 
 ## 个人理解：secret没有创建完成，就开始挂载
```

### 4 将分离开，先创建secret
```
# vim dashboard-certs.yaml 
apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kube-system
type: Opaque
```
```
# kubectl create -f dashboard-certs.yaml 
```

### 5. 暴露端口
```
# vim kubernetes-dashboard.yaml
...
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 31313
  selector:
    k8s-app: kubernetes-dashboard

```


### 6. 创建 dashboard
```
# kubectl create -f kubernetes-dashboard.yaml
```

### 7. 获取token 登录
```
# kubectl describe secrets $(kubectl get secrets -n kube-system | grep kubernetes-dashboard-token | cut -d' ' -f1) -n kube-system
Name:         kubernetes-dashboard-token-ghbbm
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=kubernetes-dashboard
              kubernetes.io/service-account.uid=189f2d5f-bc64-11e7-9acb-000c295cee54

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1090 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC10b2tlbi1naGJibSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjE4OWYyZDVmLWJjNjQtMTFlNy05YWNiLTAwMGMyOTVjZWU1NCIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTprdWJlcm5ldGVzLWRhc2hib2FyZCJ9.PE5cebZWUUqRoHTllnRwKImbLwo-4gaQetMAgCIBXBQZ1V15KyTAvKTuH6cgvtToMtmyXBgn861D1sTL0getI_P4oauoJXoaERY306UA-e7G4kLSkBz-_XJ2CXkYPwnolGCoWQfUdJLk2Tt6tG-c3Qi_mWsErzfARtC2BmZM8DflHvS0oe3yW-8DsglOt2dleMPnEn65bu9BNIPt06TSNkfGtAKlt8uCSo9LOkuYw2EpNnR3_pt5BDXLpOnzztldbRfSu7_xyXYArVi9kIfgM1sRkeDdoJaMFLyhcF9RhkLsIhNZvm2k08QvkTbVcKoagh0rXD7twI9jGZiL_tID0w
```
```
登录 https://<MasterIP>:31313  输入 token
```

#### 附：
如果想直接 Skip 进入 dashboard，则可以将dashboard 的 ServiceAccount 的权限提为 cluster-admin （不建议）
```
# vim dashboard-admin.yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
```
```
# kubectl create -f dashboard-admin.yaml
```
