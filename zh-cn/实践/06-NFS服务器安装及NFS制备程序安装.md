# NFS服务器安装及NFS制备程序安装

## NFS服务器安装
NFS是一种分布式文件系统协议，由sun公司开发，旨在允许客户端主机可以像访问本地存储一样通过网络访问服务端文件

### 安装NFS服务器

1. 登录需要安装NFS服务器的主机执行以下命令完成NFS安装
```
yum -y install nfs-utils rpcbind
```
2. 创建NFS目录
```
mkdir -p /nfs/data
```
3. 添加需要共享目录
```
# 编辑export文件    
vim /etc/exports

# 往export文件添加共享目录配置：NFS共享路径 客户机IP段(参数1,参数2,参数3,...,参数n)
# 如果指定*则所有能连接到NFS服务所在主机的所有客户端都能访问。如果指定某个IP端或者IP；则只有在IP段内的或者指定IP的客户端能访问NFS服务
/nfs/data *(rw,no_root_squash,sync)

# 使配置生效
exportfs -r

# 查看生效
exportfs
```

共享目录参数说明

- ro：只读访问
- rw：读写访问
- sync：所有数据在请求时写入共享
- async：nfs在写入数据前可以响应请求
- secure：nfs通过1024以下的安全TCP/IP端口发送
- insecure：nfs通过1024以上的端口发送
- wdelay：如果多个用户要写入nfs目录，则归组写入（默认）
- no_wdelay：如果多个用户要写入nfs目录，则立即写入，当使用async时，无需此设置
- hide：在nfs共享目录中不共享其子目录
- no_hide：共享nfs目录的子目录
- subtree_check：如果共享/usr/bin之类的子目录时，强制nfs检查父目录的权限（默认）
- no_subtree_check：不检查父目录权限
- all_squash：共享文件的UID和GID映射匿名用户anonymous，适合公用目录
- no_all_squash：保留共享文件的UID和GID（默认）
- root_squash：root用户的所有请求映射成如anonymous用户一样的权限（默认）
- no_root_squash：root用户具有根目录的完全管理访问权限
- anonuid=xxx：指定nfs服务器/etc/passwd文件中匿名用户的UID
- anongid=xxx：指定nfs服务器/etc/passwd文件中匿名用户的GID

4. 启动rpcbind、nfs服务
```
systemctl start rpcbind && systemctl enable rpcbind
sudo systemctl enable nfs-server && sudo systemctl start nfs-server
```

5. 查看rpc服务的注册情况
```
rpcinfo -p localhost
```

6. 查看nfs共享目录
```
showmount -e nfs服务器ip
```

### 客户机安装NFS
1. 安装NFS工具包
```
yum -y install nfs-utils

sudo systemctl enable nfs-server && sudo systemctl start nfs-server
```

2. 查看nfs共享目录
```
showmount -e nfs服务器ip
```

## 安装NFS制备程序

### 创建RBAC资源
通过rbac资源对StorageClass、PV、PVC等资源进行权限控制
```
cat <<EOF> rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  namespace: default
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update","patch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["create", "delete", "get", "list","watch", "patch", "update"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
  namespace: default
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: Role
  name: leader-locking-nfs-client-provisioner
  apiGroup: rbac.authorization.k8s.io
EOF

# 执行如下指令创建RBAC资源
kubectl apply -f rbac.yaml
```
- 由于nfs-client-provisioner是命名空间下资源，因此如果要操作集群系统级别资源，例如StorageClass，则需要分配权限才行

### 创建NFS Provisioner资源
```
# 创建nfs provisioner资源描述文件
cat <<EOF> nfs-client-provisioner.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-client-provisioner
  labels:
    app: nfs-client-provisioner
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-client-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: registry.cn-hangzhou.aliyuncs.com/open-ali/nfs-client-provisioner
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: hskp.io/nfs-client-provisioner
            - name: NFS_SERVER
              value: nfs文件服务器的Ip地址，不需要带端口
            - name: NFS_PATH
              value: nfs文件服务器共享的目录
      volumes:
        - name: nfs-client-root
          nfs:
            server: nfs文件服务器的Ip地址，不需要带端口
            path: nfs文件服务器共享的目录
EOF

# 创建nfs provisioner资源
kubectl create -f nfs-client-provisioner.yaml
```

### 测试nfs provisioner是否可用

#### 创建StorageClass
```
cat <<EOF> managed-nfs-storage.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-nfs-storage
provisioner: hskp.io/nfs-client-provisioner #对应nfs-client-provisioner.yaml文件中spec.template.spec.containers[0].env下的PROVISIONER_NAME配置的内容
EOF

# 使用如下指令创建StorageClass
kubectl create -f managed-nfs-storage.yaml
```
- 创建StorageClass并指定nfs provisioner。StorageClass是系统层级的资源

#### 创建pvc使用storageClass动态创建pv
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
spec:
  storageClassName: managed-nfs-storage
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Mi
```

#### 创建pod使用pvc
```
kind: Pod
apiVersion: v1
metadata:
  name: test-pod
spec:
  containers:
    - name: test-pod
      image: nginx:1.20.0
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - name: nfs-pvc
          mountPath: "/usr/share/nginx/html"
  restartPolicy: "Never"
  volumes:
    - name: nfs-pvc
      persistentVolumeClaim:
        claimName: test-claim
```
- 使用pvc卷

#### 查看pvc资源状态
执行如下指令查看pvc资源列表
```
kubectl get pvc # 如果需要指定命名空间可以添加 -n 命名空间
```
- 查看STATUS列状态变为Bound表示已经成功绑定pv

#### 查看动态创建的pv资源
```
kubectl get pv
```
- 执行上述命令可以查看集群下的pv资源列表
- pvs属于系统资源而非命名空间下资源
- CLAIM：可以知道那个命名空间下的那个pvc于当前pv绑定
- STORAGECLASS：pv是由那个storageClass动态创建
- RECLAIM POLICY：告诉pv的保留策略

#### nfs提供者无法动态分配资源
如果pvc一直没有到Bound状态，可以查看nfs-client-provisioner pod日志，如果报以下类似错误，可以通过如下方式解决
```
E0209 04:58:34.682881       1 controller.go:1004] provision "mysql/www-nginx-0" class "managed-nfs-storage": unexpected error getting claim reference: selfLink was empty, can't make reference
```
SelfLink在Kubernetes v1.16引入，v1.20之前默认使用，在v1.20之后默认禁用，需要在/etc/kubernetes/manifests/kube-apiserver.yaml中添加如下指令参数启用SelfLink
```
spec:
containers:
- command:
    - kube-apiserver
    - --feature-gates=RemoveSelfLink=false
```
执行如下指令使参数生效
```
kubectl apply -f /etc/kubernetes/manifests/kube-apiserver.yaml
```

