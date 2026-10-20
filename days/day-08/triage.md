web-a：ImagePullBackOff，使用 kubectl describe pod 查看 Events，发现镜像标签写错为 nginx:1.27-alpinee，将镜像改为 nginx:1.27 后恢复正常。
web-b：容器启动后反复报错并重启，使用 kubectl logs 查看日志、kubectl get configmap app-conf -n 8-day -o yaml 检查配置，发现 ConfigMap 的键名 application.conf 与程序要求的 app.conf 不一致，将键名改为 app.conf 后恢复正常。
web-c：Pod 处于 Pending，使用 kubectl describe pod 查看 Events，发现两个节点均 Insufficient cpu；检查 Deployment 后发现 requests.cpu 设置为 3 核，将其调整为 100m 后恢复正常。
web-d：Pod 处于 Running 但未就绪，使用 kubectl describe pod 查看 Events，发现 Readiness Probe 检查 8080 端口时连接被拒绝，而 Nginx 实际监听 80 端口；将 readinessProbe 的端口改为 80 后恢复正常，Service Endpoints 随之出现。
