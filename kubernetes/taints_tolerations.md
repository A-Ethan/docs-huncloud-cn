# 污点、容忍污点
https://mp.weixin.qq.com/s?__biz=Mzg3ODAzMTMyNQ==&mid=2247488004&idx=1&sn=f796ed20678c162d278d069565ea5788&chksm=cf18aab6f86f23a0a6b65aa20161e997b71e48a6761d0e4bf5af4b7e08bd603093074ee74b1b&mpshare=1&scene=1&srcid=1202TEQf5zNR1UhmxjHq1SFC&sharer_sharetime=1606904795703&sharer_shareid=2e51f02d22f752038d1b66400ead27fa&key=df73985cdb2a2cbc74d3feea06046153e9efc19ebbf67f3b501fc8c506b77971b367b8ac4d8743ab97d451da60a1cb5d197b4a4bc5ce67b0c200255a0f82b26be5cac058efb8a71cf57ddbc06cc7bd02d356150ea004ade35052dae0499bb0ed42e809dbe66ed5a6e03564a83f393ded616a60c5d8c5b441ae82af425ae2c731&ascene=1&uin=MjkyMjQ1NDA2MA%3D%3D&devicetype=Windows+10+x64&version=6300002f&lang=zh_CN&exportkey=AZoJ0f3GKvExEN8DN00%2F2CI%3D&pass_ticket=QWLKYbIesPzLAxLPHxyQjRwed%2F%2F4%2FsK5POWropMLu8W8l%2B5%2BWpK9Xu0WO%2FtQIu%2F%2B&wx_header=0

https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/

> 可以配合selector和affinity使用，取交集可用节点

## 污点

给节点增加污点，必须容忍污点才能调度到该节点

```
kubectl taint nodes node1 key1=value1:NoSchedule
```
## 容忍污点

```yaml
apiVersion: apps/v1
kind: Deployment 
metadata: 
  name: hello-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-nginx
  template:
    metadata:
      labels:
        app: hello-nginx
    spec:
      containers:
      - name:  nginx
        image: uhub.service.ucloud.cn/hello123/nginx:1.17.10-alpine
        env:
        - name: NGINX_VERSION
          value: 1.17.10
        workingDir: /
        command: ["nginx"]
        args: ["-g", "daemon off;"]
        ports:
        - name: http
          containerPort: 80
      tolerations:
      - key: "example-key"
        operator: "Exists"  # 存在，不用指定value
        effect: "NoSchedule"
```

```yaml
      tolerations:
      - key: "example-key"
        operator: "Equal"  # 相等，需要指定value 
        value: "value1"
        effect: "NoSchedule"
```
