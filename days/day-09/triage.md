# 第 9 题：Service 链路排障记录

1. Endpoints 为 <none> → 使用 `kubectl get endpoints api -n 9-day`、`kubectl describe svc api -n 9-day` 和 `kubectl get pods -n 9-day --show-labels` 检查 → Service selector 中的 `tier: backnd` 与 Pod 标签 `tier: backend` 不一致 → 将 Service selector 修正为 `tier: backend`。

2. 修正标签后 Endpoints 仍为空 → 使用 `kubectl get pod api-58b99d9b9c-fzlkn -n 9-day -o yaml` 检查容器配置，并查看 Service 的 `targetPort` → 后端最初没有声明对应的命名端口，Service 的 `targetPort: web` 也与后来声明的端口名 `http` 不一致 → 在 api Deployment 中声明命名端口 `http: 80`，将 Service 的 `targetPort` 改为 `http`。

3. client 持续输出 FAIL → 使用 `kubectl logs deployment/client -n 9-day --tail=10` 检查请求结果，并结合 `kubectl describe svc api -n 9-day` 检查 Service 端口 → client 访问 `api:8080`，但 Service 原先只开放 `port: 80` → 将 Service 的 `port` 改为 `8080`，保留 `targetPort: http`。


