# Kubernetes API服务安全防护

## 用户和组

1. 创建ServiceAccount
```
   vim prometheus-service-account.yml

   apiVersion: v1
   kind: ServiceAccount
   metadata:
   name: prometheus
```

2. 创建Cluster级别角色
```
   vim prometheus-clusterrole.yml

   apiVersion: rbac.authorization.k8s.io/v1beta1
   kind: ClusterRole
   metadata:
   name: prometheus
   rules:
    - apiGroups: ["","extensions","apps"]
      resources:
        - nodes
        - nodes/proxy
        - services
        - endpoints
        - pods
        - deployments
          verbs: ["get", "list", "watch"]
        - nonResourceURLs: ["/metrics"]
```

3. 绑定角色及用户
```
   kubectl create clusterrolebinding prometheus-clusterrole-binding --clusterrole=prometheus
   --serviceaccount=default:prometheus
```

4. 任意启动一个pod，serviceAccountName指定为如上的prometheus
```
   vim example.yml

   apiVersion: v1
   kind: Pod
   metadata:
   name: example
   spec:
   serviceAccountName: prometheus
   containers:
    - image: luksa/kubia
      name: kubia
      ports:
        - containerPort: 8080
          protocol: TCP
```

5. 拷贝这个Pod的ca.crt/token/namespace
```
   kubectl describe pod example

    ---
   Mounts:
   /var/run/secrets/kubernetes.io/serviceaccount from prometheus-token-7vdmz (ro)
    ---

   kubectl cp default/example:/var/run/secrets/kubernetes.io/serviceaccount ./

   # 可能会报警告，无法成功，通过警告提示的命令执行即可
   kubectl exec -n "default" "example" -- tar cf - "/var/run/secrets/kubernetes.io/serviceaccount" | tar xf -
```