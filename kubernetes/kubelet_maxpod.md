# Node节点Pod控制

每个节点kubelet控制的参数

## 登录节点执行

```bash
# 修改kubelet参数
vim /etc/kubernetes/kubelet.conf
```

```yaml
maxpods: 110 # 单节点最大的pod数量
podsPerCore: 8 # 单核最大的pod数量
```

```bash
# 重启kubelet
systemctl restart kubelet
```