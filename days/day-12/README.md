# Day 12 — RBAC + 用户证书 (CSR) + ResourceQuota / LimitRange

## 场景
给实习生 xiaoming 开权限：在 `12-day` 里管 Deployment、看 Pod 和日志；不能看 Secret、不能 exec、不能出 namespace。

## 做了什么（每步一个产物）
1. `openssl genrsa` → `xiaoming.key`（私钥，不上传）
2. `openssl req -subj "/CN=xiaoming/O=interns"` → `xiaoming.csr`
3. `CertificateSigningRequest`（signer `kubernetes.io/kube-apiserver-client`，24h，client auth）→ `kubectl certificate approve` → 取回 `xiaoming.crt`
4. 独立 `xiaoming.kubeconfig`（证书内嵌，默认 ns 12-day，不动管理员 kubeconfig；不上传）
5. Role `dev-role` + RoleBinding 绑 **组 `interns`**
6. 配额：ResourceQuota `q12`（pods=4 / cpu 500m / mem 512Mi）+ LimitRange 默认 requests
7. CKA 小题：SA `log-reader` 经 RoleBinding 拿 ClusterRole `pod-reader-12`（只在 12-day 生效）；找出绑 cluster-admin 的 ClusterRoleBinding

## 验收结果
| 操作（以 xiaoming 身份） | 结果 |
|---|---|
| 创建 Deployment web12 | ✅ allowed |
| get secrets -n 12-day | Forbidden |
| get pods -n default | Forbidden |
| exec 进 Pod（pods/exec） | Forbidden |
| 有 Quota 无 requests 建 web12 | READY 0/3，报错在 **ReplicaSet events**：`must specify requests.cpu ... requests.memory` |
| 加 LimitRange（100m/128Mi）后扩到 6 | 只跑 4 个：pods=4 与 requests.memory=512Mi 同时打满 |

## 知识点 / 踩坑
- K8s 没有用户对象：用户名=证书 CN，组=证书 O，只认 CA 签过的证书。
- 范围由 Binding 决定：ClusterRole + RoleBinding = 只在该 ns；ClusterRoleBinding = 全集群（实验时误建 CRB `xiaoming-view` 绑 view，小明立刻能看 default，已删除）。内置 view 不含 secrets。
- 绑组的好处：新实习生证书写 O=interns 即可，binding 不用改。代价：RBAC 没有 deny，无法从组里单独踢一个人。
- 客户端证书无法吊销：删证书文件/CSR 对象没用，只能撤授权（删 binding），所以证书要给短有效期。
- 报错跟着“提交人”走：RS 建 Pod 被配额拒 → 报错在 RS events，Pod 根本没生成；自己 `kubectl run` → 报错直接回终端。
- 层级：Deployment(replicas) → ReplicaSet(只数 Pod) → Pod → 容器(kubelet 拉起)；配额在提交 Pod 对象时拦，按 Pod 内所有容器 requests 求和。
- change-cause 注解键是 `kubernetes.io/change-cause`（本题误写成 `tag`）。
- 考试时：`kubectl create role/rolebinding -h` 看例子，`kubectl explain csr.spec` 查 CSR 字段。
