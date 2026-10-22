# 实际排障与操作命令（摘自 bash 历史）

## 故障定位与修复
```bash
kubectl apply -f shop.yaml                       # replicas 2 → 4
kubectl get pods -n 10-day -o wide               # 新 Pod Pending，NODE 为 <none>
kubectl describe pod shop-66dbfd9695-d58ts -n 10-day   # Events 里没有 Scheduled
kubectl get pods -n kube-system -o wide          # kube-scheduler-k8s-master 异常
kubectl logs -n kube-system kube-scheduler-k8s-master --previous   # 看挂掉那次的日志：找不到 schedular.conf
grep -n -C 3 -E 'schedular\.conf|scheduler\.conf' /etc/kubernetes/manifests/kube-scheduler.yaml
cp -a /etc/kubernetes/manifests/kube-scheduler.yaml /root/hark-test/10-day/kube-scheduler.yaml.bak
sed -i 's#/etc/kubernetes/schedular.conf#/etc/kubernetes/scheduler.conf#' /etc/kubernetes/manifests/kube-scheduler.yaml
kubectl get pods -n 10-day                       # 4/4 Running
```

## 故障期间确认影响面
```bash
kubectl get ingress,svc,pods -n test-demo -o wide
curl -k -sS -o /dev/null -w 'HTTP %{http_code}\n' --resolve niuzhe-huanuc.com:443:<节点内网IP> https://niuzhe-huanuc.com   # HTTP 200
```

## CKA 小题 a：etcd 快照
宿主机没有 etcdctl，借 etcd 静态 Pod 里的；快照写到它 hostPath 挂载的 /var/lib/etcd，宿主机才能拿到。
```bash
kubectl exec -n kube-system etcd-k8s-master -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  snapshot save /var/lib/etcd/etcd-snap.db
cp /var/lib/etcd/etcd-snap.db /root/hark-test/10-day/etcd-snap.db
kubectl exec -n kube-system etcd-k8s-master -- etcdctl snapshot status /var/lib/etcd/etcd-snap.db --write-out=table \
  > /root/hark-test/10-day/etcd-status.txt
```
（快照文件本身含集群全部数据和 Secret，不上传仓库。）
