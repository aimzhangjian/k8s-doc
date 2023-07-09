# Nodeport暴露服务

## 通过service NodePort类型资源暴露服务
```
apiVersion: v1
kind: Service
metadata:
  name: hskp-devops-node-port
spec:
  type: NodePort
  ports:
  - name: devops-node-port
    port: 18300
    targetPort: 18300
    nodePort: 30123
  selector:
    choerodon.io/release: hskp-devops-service
```