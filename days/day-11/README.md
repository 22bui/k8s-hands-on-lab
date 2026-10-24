# Day 11：CoreDNS 故障演练 + 证书到期检查 + 自定义 Pod DNS

## 场景
第二次由别人往集群里埋故障。`client` 每 3 秒按服务名 `wget http://api`，突然开始刷 FAIL。要求：只改坏掉的那一处、改前备份到 manifests 目录之外，不碰 kube-system 其他组件、ingress-nginx 和线上网站；再做两道 CKA 风格小题。

## 文件
| 文件 | 内容 |
| --- | --- |
| `dns-lab.yaml` | 演练用的 api（nginx）+ Service + client |
| `dns.log` | 排查记录：resolv.conf、三种名字的解析结果、kube-dns Service/Endpoints、故障与修复后的 Corefile、reload 日志 |
| `coredns-fix.diff` | Corefile 里被改坏的那一行及修复 |
| `certs.txt` / `apiserver-exp.txt` | `kubeadm certs check-expiration` 输出；从证书文件本身读出的 apiserver 到期时间 |
| `dns-pin.yaml` / `dnspin.txt` | `dnsPolicy: None` + 自定义 `dnsConfig` 的 Pod 及其 resolv.conf |

## 故障复盘
- **现象**：client 刷 FAIL；CoreDNS 两个 Pod 一直 Running、0 重启。
- **定位**：client 的 resolv.conf 正常（nameserver 10.96.0.10）→ 三种名字对比：短名 `api` 和 FQDN `api.11-day.svc.cluster.local` 都 **NXDOMAIN**，外网域名却能正常解析 → DNS 服务本身活着、网络也通，是“活着但答错了” → 看 CoreDNS 的 ConfigMap。
- **根因**：Corefile 中 `kubernetes` 插件负责的区被改成了 `cluster.lokal`。
- **修复**：先导出 ConfigMap 备份，再把那一处改回 `cluster.local`。没有重启任何东西，约半分钟后 client 恢复 OK。

## 观察与要点
- **为什么外网没坏**：CoreDNS 的插件按顺序过。`kubernetes` 插件只回答自己负责的区，`cluster.lokal` 不认 `cluster.local`，请求就交给兜底的 `forward .`，转到阿里云上游 DNS；上游不认识集群里的 `api`，回 NXDOMAIN。外网域名本来就走 forward，所以毫发无伤。
- **为什么改完不用重启**：两步——kubelet 定期把新的 ConfigMap 同步进 Pod 挂载的文件（最多约一分钟）；CoreDNS 的 `reload` 插件发现 Corefile 变了就热加载（日志 `Reloading complete`）。
- **ndots:5**：名字里的点少于 5 个，就先依次拼 search 后缀查。短名 `api` 因此先查到 `api.11-day.svc.cluster.local`；代价是外网域名（如 2 个点）也要先白查 3 个后缀（A + AAAA 各一次）才查原名。解法：末尾加点写成绝对域名，或调低 ndots。
- **`dnsPolicy: None` vs 默认 + `dnsConfig`**：None 是从白纸写起，只用 dnsConfig；默认 ClusterFirst 加 dnsConfig 是**合并**，search 里仍会保留 `cluster.local` 等默认项，只有 ndots 这种 option 会覆盖。题目要求只有两个 search，所以必须 None。
- **证书到期两种看法**：`kubeadm certs check-expiration` 看全部；`openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -enddate` 直接读证书文件本身。
