# Day 10：控制面故障演练（kube-scheduler）+ etcd 快照

## 场景
第一次由别人往集群里埋故障。运营要把 `shop` 从 2 副本扩到 4，新 Pod 一直 Pending。要求：只改坏掉的那一处、改前备份，不碰 kube-system 其他组件、ingress-nginx 和线上网站；再做一道 CKA 风格的 etcd 快照题。

## 文件
| 文件 | 内容 |
| --- | --- |
| `shop.yaml` | 演练用的 Deployment（扩到 4 副本） |
| `scheduler-fix.diff` | 静态 Pod 清单里被改坏的那一行及修复 |
| `commands.md` | 定位、修复、确认影响面、etcd 快照的实际命令 |
| `etcd-status.txt` | `snapshot status -w table` 输出 |

## 故障复盘
- **现象**：新 Pod 一直 Pending，`NODE` 为 `<none>`；原有 2 个 shop Pod 和线上网站（HTTP 200）一切正常。
- **定位**：`describe` Pending Pod → Events 里没有任何 `Scheduled` / `FailedScheduling`，说明根本没人来调度 → `kube-system` 里 `kube-scheduler-k8s-master` 异常 → `logs --previous` 看到它找不到 kubeconfig 文件。
- **根因**：`/etc/kubernetes/manifests/kube-scheduler.yaml` 里 `--kubeconfig` 被改成不存在的 `schedular.conf`，scheduler 启动即退出，CrashLoop。
- **修复**：先把清单备份到 manifests 目录之外，再把那一处改回 `scheduler.conf`，kubelet 发现文件变化自动重建 scheduler，Pending 的 Pod 随即被调度，4/4 Running。主场景用时约 14 分钟。
- **影响面**：scheduler 只在“给新 Pod 分配节点”那一刻出场。旧 Pod 早已绑定节点、由各自节点的 kubelet 守着运行；访问流量走 ingress / Service，也不经过 scheduler，所以只有“新 Pod 上不了岗”。

## 观察与要点
- **Pod 不是 kubelet 创建的**：Deployment → ReplicaSet，由 kube-controller-manager 里的 ReplicaSet 控制器向 API Server 写入 Pod 对象（此时 `nodeName` 为空）→ scheduler 选好节点写入 `nodeName`（绑定）→ 该节点的 kubelet 才把容器真正拉起来。kubelet 只管“写了自己名字”的 Pod。
- **为什么 `kubectl delete` scheduler Pod 修不好**：它是静态 Pod，kubectl 看到的只是镜像 Pod；删掉后 kubelet 会按 manifests 目录里同一份坏文件马上重建，照样挂。病根在文件，不在 Pod。
- **备份不能放进 manifests 目录**：kubelet 会把该目录下的文件都当清单，备份放进去可能被拉起成第二个 scheduler。
- **etcd 快照为什么借容器**：宿主机没有 etcdctl，etcd 容器里有，证书也通过 hostPath 挂了进去；容器与宿主机文件系统隔离，所以快照要写到 hostPath 挂载的 `/var/lib/etcd`，宿主机才能拿到。
- **更早发现**：Pending 的 Pod 看 Events，没有 Scheduled 就直接查 scheduler；平时用定时脚本 / 监控巡检控制面健康（如 `kubectl get --raw='/readyz?verbose'`、kube-system Pod 重启次数和 CrashLoop 告警），不要等用到才发现坏了。
