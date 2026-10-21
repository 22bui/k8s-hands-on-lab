# Day 9：静态 Pod + Service 链路排障 + 节点失联演练

## 场景
往下挖一层：看不归 API Server 管的静态 Pod；client 访问 `api:8080` 一直 FAIL，只准改 api 那边，一层层查到 iptables；在 worker 上停 kubelet，观察 NotReady、污点和 `tolerationSeconds` 驱逐；最后一道 CKA 风格 NodePort 小题。

## 文件
| 文件 | 内容 |
| --- | --- |
| `static-web.yaml` | 放进 server01 静态 Pod 目录（kubelet 配置 `staticPodPath: /etc/kubernetes/manifests`）的 nginx |
| `svc-broken.yaml` | api + client（修复后） |
| `triage.md` | 每处毛病：现象 → 命令 → 根因 → 修法 |
| `chain.txt` | `KUBE-SERVICES` → `KUBE-SVC` → `KUBE-SEP` → DNAT 的 iptables 规则，附 `--probability 0.5` 注释 |
| `evict-fast.yaml` / `evict-default.yaml` | 钉在 server01 的两个 busybox：一个 not-ready/unreachable 各容忍 20s，一个不写容忍 |
| `node.log` | 停 kubelet 的时间线与观察 |
| `api-np.yaml` / `np.txt` | NodePort 30909，curl 结果 `200` |

## Service 排障结果
| 现象 | 根因 | 修法 |
| --- | --- | --- |
| Endpoints `<none>` | selector `tier: backnd`，Pod 是 `tier: backend` | 改 selector |
| 改完仍为空 | `targetPort: web` 是命名端口，容器没声明同名端口 | 容器声明 `name: http, containerPort: 80`，targetPort 改 `http` |
| client 仍 FAIL | client 连 8080，Service `port: 80` | `port: 8080`，targetPort 不变 |

## 节点失联时间线
- 10:33:05 停 kubelet → 10:33:48 NotReady（约 40s 宽限期）
- evict-fast 约 20s 后被 TaintManagerEviction；替身因 `node.kubernetes.io/unreachable` 污点 + nodeSelector 卡 Pending，节点恢复后回到 server01
- evict-default 没被赶：系统自动补了 300s 容忍

## 观察与要点
- 静态 Pod 的真身是节点上的文件，kubectl 看到的是镜像（mirror）Pod，`ownerReferences` 指向 Node；`kubectl delete` 后 kubelet 会重新上报，真删要删文件。kube-apiserver 自己也是静态 Pod。
- 不写容忍的 Pod 由 API Server 准入插件 `DefaultTolerationSeconds` 自动加 not-ready / unreachable `NoExecute` 各 300s。
- 判 NotReady 的是 kube-controller-manager 的节点生命周期控制器：kubelet 每 10s 续一次 Lease，40s（`node-monitor-grace-period`）没续就标 Unknown 并加 unreachable 污点；API Server 只负责写进 etcd。
- 删除 Pod 只是打上 `deletionTimestamp`，要 kubelet 停掉容器并确认才真正删除；断联期间容器其实还在跑。`--force` 只删记录不杀容器，StatefulSet 可能出现两个同名实例。
- kube-proxy 在每个节点都开 NodePort，所以 Pod 全在 server01 时访问 master 的 30909 也能通（`externalTrafficPolicy: Cluster` 会跨节点转发）；改成 `Local` 就只有有 Pod 的节点能通。
- `--probability 0.5` 是随机挑后端，不是严格轮询。
