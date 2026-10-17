# K8s Hands-on Lab · 三周 Kubernetes 实战记录

> 在两台阿里云 ECS 上自建 Kubernetes 集群，每天一道实操题，从 Deployment 一路做到调度、存储、RBAC、Helm 和 HPA。
> 每道题都在真实集群上动手完成，并记录现象、踩坑和原理。

## 🧱 环境

| 节点 | 角色 | 配置 | 说明 |
|---|---|---|---|
| k8s-master | control-plane | 阿里云 2C4G | kubeadm 部署，ingress-nginx（hostNetwork 占用 80/443） |
| server01 | worker | 阿里云 2C2G | 2026-10-16 加入集群 |

- 网络插件：Flannel
- 入口：ingress-nginx + 自有域名 `niuzhe-huanuc.com`
- HTTPS：cert-manager + Let's Encrypt（HTTP-01）自动签发与续期
- 镜像：国内无法直连 Docker Hub，通过国内镜像源拉取后用 `ctr` 重新打原名

## 🌐 公网请求链路

```
浏览器
  → DNS 解析 niuzhe-huanuc.com
  → 公网 IP（阿里云 EIP）
  → 节点内网 IP :443
  → ingress-nginx（按 Host / Header / Cookie 路由，TLS 终止）
  → 直接读取 Endpoints，转发到 Pod IP（不经过 ClusterIP / kube-proxy）
  → Pod
```

## 📅 每日实操

| Day | 主题 | 关键点 |
|---|---|---|
| [Day 1](days/day-01) | Deployment | 扩缩容、滚动更新、`rollout undo`、change-cause |
| [Day 2](days/day-02) | ConfigMap + Secret | env vs 卷挂载、subPath 不热更新、Secret 卷权限 0400 |
| [Day 3](days/day-03) | 健康检查 | startup / liveness / readiness 探针、零中断滚动 |
| [Day 4](days/day-04) | 资源管理 | requests/limits、OOMKilled(137)、CPU 节流、QoS |
| [Day 5](days/day-05) | Ingress 进阶 | Header / 权重灰度、cookie 亲和、checksum 注解触发滚动、无 selector Service |
| [Day 6](days/day-06) | 调度 | nodeAffinity、污点与容忍、podAntiAffinity、PDB + drain |

*持续更新中……*

## 🧠 踩坑笔记（精选）

- **ConfigMap 卷热更新 ≠ 滚动更新**：文件内容会变，但 Pod 不会重建；想自动滚动，要把 checksum 写进 `Deployment.spec.template.metadata.annotations`。
- **subPath 挂载不会热更新**：subPath 在启动时绑定具体文件，kubelet 更新的是 `..data` 软链接，绑定的旧文件不变。
- **Ingress 不走 kube-proxy**：ingress-nginx 自己读 Endpoints 直连 Pod IP；集群内访问 Service 名称 / NodePort 才走 kube-proxy。
- **零中断滚动的关键是 `maxUnavailable: 0`**，配合 readiness 探针。

## 📂 目录结构

```
days/
  day-01/   # 每天的 YAML、命令记录、README 笔记
  day-02/
  ...
```
