# Day 6：调度 —— nodeAffinity、污点容忍、Pod 反亲和、PDB + drain

## 场景
两节点集群（k8s-master + worker server01）。边缘服务只能跑在 server01，server01 设为"专用"节点，节点维护时服务不能全挂。
红线：只对 server01 做 taint / cordon / drain，控制面节点不动（ingress-nginx 未固定节点）。

## 文件
| 文件 | 内容 |
| --- | --- |
| `edge-web.yaml` | Deployment，required nodeAffinity 钉在 `zone=edge`，带 `dedicated=edge:NoSchedule` 容忍 |
| `spread-web.yaml` | Deployment，required podAntiAffinity（topologyKey `kubernetes.io/hostname`），3 副本观察第 3 个 Pending |
| `edge-pdb.yaml` | PDB `minAvailable: 1` |
| `drain.log` | `kubectl drain server01` 被 PDB 拦截直到超时的完整输出 |

## 操作步骤
```bash
kubectl label node server01 zone=edge
kubectl apply -f edge-web.yaml                       # 2 个 Pod 都落在 server01
kubectl taint node server01 dedicated=edge:NoSchedule  # 已有 Pod 不受影响
# 改 env 触发滚动 → 新 Pod Pending（untolerated taint）→ 加 toleration 修复
kubectl apply -f spread-web.yaml                     # 每节点 1 个，第 3 个 Pending
kubectl apply -f edge-pdb.yaml
kubectl drain server01 --ignore-daemonsets --delete-emptydir-data --timeout=90s
kubectl uncordon server01
kubectl taint node server01 dedicated=edge:NoSchedule-
kubectl label node server01 zone-
```

## 观察结果
- **NoSchedule** 只在调度时生效，已经运行的 Pod 不会被赶走；滚动时新 Pod 报 `untolerated taint {dedicated: edge}`。
- **反亲和**：第 3 个 Pod 报 `0/2 nodes are available: 2 node(s) didn't match pod anti-affinity rules`。
- **drain 卡住**：第一个 edge-web Pod 被驱逐后，替身只能去 `zone=edge` 的 server01，而它已被 cordon → 替身 Pending → 健康副本只剩 1 → PDB 允许中断数为 0 → 第二个 Pod 一直 `Cannot evict pod as it would violate the pod's disruption budget`，直到超时。

## 知识点
- **Taint effect**
  - `NoSchedule`：没容忍的新 Pod 不调度过来，已有的不管。
  - `PreferNoSchedule`：软规则，没容忍的尽量避开，实在没地方才来；有容忍则不受影响。
  - `NoExecute`：连已在运行、没容忍的 Pod 也驱逐。
- **tolerationSeconds**：只配合 NoExecute，表示"能容忍，但只容忍 N 秒"。K8s 默认给 Pod 加上 `node.kubernetes.io/not-ready` 和 `unreachable` 各 300 秒，所以节点挂掉约 5 分钟后 Pod 才被迁走。
- **required vs preferred**：required 是硬条件，不满足就 Pending；preferred 带 `weight`（1–100），只影响打分，没得选时仍会调度。
- **IgnoredDuringExecution**：Pod 调度完成后节点标签变了也不会被赶走；但 Pod 一旦重建（删除、滚动更新、驱逐、节点重启）就要重新满足条件，否则 Pending。容器原地重启不算重建。
- **PDB + drain**：drain 走 Eviction API，每驱逐一个都先问 PDB。要顺利维护，被驱逐 Pod 的替身必须有别的节点可落脚；如果业务只能跑在一台节点上，维护就必然中断，应安排维护窗口，而不是删掉 PDB 硬来。
- **readiness 与滚动**：没有 readinessProbe 时，容器一启动就算 Ready，进入 Endpoints 并被当作可用，Deployment 会接着删旧 Pod，`maxUnavailable: 0` 的保护就失效了。
