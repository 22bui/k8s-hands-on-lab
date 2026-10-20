# Day 8：DaemonSet + Job/CronJob + 四个病号排障

## 场景
给每台节点放一个日志小弟（DaemonSet），跑并行批任务和必失败任务（Job），再用 `get / describe / logs / events` 修好 4 个故意弄坏的 Deployment。最后一道 CKA 风格 CronJob 小题。

## 文件
| 文件 | 内容 |
| --- | --- |
| `node-agent.yaml` | DaemonSet：Downward API `spec.nodeName` → `NODE_NAME`，每 30s 写 hostPath `/var/log/hark-agent/agent.log`；带 `control-plane:NoSchedule` 容忍 |
| `batch.yaml` | Job `completions: 6`、`parallelism: 2` |
| `flaky.yaml` | Job 直接 `exit 1`，`backoffLimit: 2` |
| `job.log` | flaky 最终状态：`BackoffLimitExceeded`，失败 3 个 Pod |
| `broken.yaml` | 4 个带病 Deployment（web-a ~ web-d，修复前原样） |
| `triage.md` | 每个病号：现象 → 命令 → 根因 → 修法 |
| `cka-date-cronjob.yaml` / `cka-date.txt` | CronJob `*/5`，`successfulJobsHistoryLimit: 1`；手动 Job 的输出 |

## 排障结果
| Pod | 现象 | 根因 | 修法 |
| --- | --- | --- | --- |
| web-a | ImagePullBackOff | 镜像标签 `nginx:1.27-alpinee` 拼错 | 改 `nginx:1.27` |
| web-b | CrashLoopBackOff | ConfigMap 键名 `application.conf`，程序要 `app.conf` | 改键名，kubelet 同步卷后下次重启即恢复 |
| web-c | Pending | `requests.cpu: 3`，两节点都 Insufficient cpu | 改 `100m` |
| web-d | Running 0/1 | readinessProbe 打 8080，nginx 听 80 | 改 80，Endpoints 出现 |

## CKA 小题命令
```bash
kubectl create cronjob cka-date -n 8-day --image=busybox:1.36 --schedule="*/5 * * * *" \
  --dry-run=client -o yaml -- date > cj.yaml   # 补 successfulJobsHistoryLimit: 1
kubectl apply -f cj.yaml
kubectl create job cka-date-now -n 8-day --from=cronjob/cka-date
kubectl logs -n 8-day job/cka-date-now > /root/hark-test/8-day/cka-date.txt
```

## 观察与要点
- DaemonSet 两节点各一个，日志里节点名分别是 `k8s-master`、`server01`。
- `backoffLimit` 是重试次数：首次 1 个 + 重试 2 个 = 失败 3 个后判 `Failed`。
- `--from=cronjob` 建的 Job 带指向 CronJob 的 ownerReference，会被算进 `successfulJobsHistoryLimit`，下一次成功运行后被清掉（连带 Pod）。
- `NoSchedule` 是硬性的：没容忍就绝不调度，其他节点满了也只会 Pending；软性的是 `PreferNoSchedule`。DaemonSet 没容忍时 DESIRED 直接少一个。
- 本集群 master 没有污点，普通 Pod 都去 server01 是调度打分：master 已分配 CPU requests 约 57%，server01 约 10%，调度器偏向空闲节点。
- ConfigMap 以卷挂载（非 subPath）时，由 kubelet 定期同步到节点（约一分钟内），Deployment 不监听 ConfigMap 内容；env 和 subPath 不会热更新。
