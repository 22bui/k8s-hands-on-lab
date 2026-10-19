# Day 7：有状态的 MySQL —— StorageClass 动态供给 + StatefulSet + PV 回收救援

## 场景
在两节点集群里跑一个 MySQL：Pod 重建、StatefulSet 被误删、甚至 PVC 被误删，数据都不能丢。

## 文件
| 文件 | 内容 |
| --- | --- |
| `mysql-service.yaml` | headless Service `mysql`（`clusterIP: None`，3306） |
| `mysql-sts.yaml` | StatefulSet `mysql`：`volumeClaimTemplates`（`data`，1Gi，`local-path`）挂 `/var/lib/mysql`，`mysqladmin ping` readinessProbe，root 密码从 Secret 注入 |
| `rescue.log` | PVC 误删救援全过程：改 Retain → 删 sts 和 PVC → 改 claimRef → 重建 sts → 查回数据 |
| `dns.txt` | busybox `nslookup` headless Service 和单个 Pod 的结果 |

Secret 用命令创建（不入库）：
```bash
kubectl create secret generic mysql-root -n 7-day --from-literal=MYSQL_ROOT_PASSWORD=<你的密码>
```

## 操作步骤
```bash
kubectl apply -f local-path-storage.yaml          # rancher local-path-provisioner
kubectl apply -f mysql-service.yaml -f mysql-sts.yaml
# 建库 hark、表 t，插入 day7
kubectl delete pod mysql-0                        # 名字、PVC、节点都不变，数据还在
kubectl delete sts mysql && kubectl apply -f mysql-sts.yaml   # PVC/PV 都还在

# 救援：删 PVC 之前先改回收策略
kubectl patch pv <pv> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
kubectl delete sts mysql; kubectl delete pvc data-mysql-0     # PV → Released
kubectl edit pv <pv>                              # claimRef 去掉 uid / resourceVersion，保留 name+namespace
kubectl apply -f mysql-sts.yaml                   # 新 data-mysql-0 自动认领老 PV，查回 day7
```

## 观察结果
- `local-path` 的 SC：`reclaimPolicy: Delete`、`volumeBindingMode: WaitForFirstConsumer`。PVC 一开始 Pending，等第一个使用它的 Pod 被调度到某个节点，才在那台节点上建目录。本地盘必须先知道 Pod 去哪。
- PV 落在 server01：`/opt/local-path-provisioner/pvc-<uid>_7-day_data-mysql-0`，并自带 `nodeAffinity: kubernetes.io/hostname in [server01]`，所以 Pod 重建总是回到 server01（Pod 跟着数据走；server01 挂了 mysql-0 只能 Pending）。
- StatefulSet 被删，PVC 不会跟着删；重建后按名字 `data-mysql-0` 接回原盘。
- 回收策略是 Delete 时，删 PVC = 删 PV = 删数据。必须**先**改 Retain 再删。
- Released 的 PV 不会被新 PVC 绑定：`spec.claimRef.uid` 还记着旧 PVC 的身份证。去掉 uid、保留 name/namespace，等于“给叫 data-mysql-0 的 PVC 预留”；整个 claimRef 删掉则 PV 变 Available，任何条件匹配（SC/容量/访问模式）的 PVC 都可能抢走。
- headless Service 没有 ClusterIP，kube-proxy 不为它写 iptables 规则；DNS 直接返回 Pod IP（`mysql` 和 `mysql-0.mysql.7-day.svc.cluster.local` 都解析到 10.244.1.33）。

## 踩坑
- `volumeClaimTemplates` 和 `spec.template` 同级，不放进 Pod 的 `spec`；`volumeMounts[].name` 要和模板的 `metadata.name` 一致，也不用再写 `volumes`。
- 误以为 PV/PVC 按名字绑定：其实按条件匹配，名字只在 claimRef 预留时才起作用。
