# Exam Questions 3

## Question 11 | Update Kubernetes Version and join cluster

Task weight: 10%

Your coworker said node cluster3-node2 is running an older Kubernetes version and is not even part of the cluster. Update Kubernetes on that node to the exact version that's running on cluster3-controlplane1. Then add this node to the cluster. Use kubeadm for this.

**Answer:**

Upgrade Kubernetes to cluster3-controlplane1 version

Search in the docs for kubeadm upgrade: https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade

```bash
➜ k get node
NAME                     STATUS   ROLES           AGE   VERSION
cluster3-controlplane1   Ready    control-plane   22h   v1.27.1
cluster3-node1           Ready    <none>          22h   v1.27.1
```

Controlplane node seems to be running Kubernetes 1.27.1. Node cluster3-node2 might not yet be part of the cluster depending on previous tasks.

```bash
➜ ssh cluster3-node2
➜ root@cluster3-node2:~# kubeadm version
kubeadm version: &version.Info{Major:"1", Minor:"27", GitVersion:"v1.27.1", GitCommit:"4c9411232e10168d7b050c49a1b59f6df9d7ea4b", GitTreeState:"clean", BuildDate:"2023-04-14T13:20:04Z", GoVersion:"go1.20.3", Compiler:"gc", Platform:"linux/amd64"}
➜ root@cluster3-node2:~# kubectl version --short
Flag --short has been deprecated, and will be removed in the future. The --short output will become the default.
Client Version: v1.26.4
Kustomize Version: v4.5.7
The connection to the server localhost:8080 was refused - did you specify the right host or port?
➜ root@cluster3-node2:~# kubelet --version
Kubernetes v1.26.4
```

Here kubeadm is already installed in the wanted version, so we don't need to install it. Hence we can run:

```bash
➜ root@cluster3-node2:~# kubeadm upgrade node
couldn't create a Kubernetes client from file "/etc/kubernetes/kubelet.conf": failed to load admin kubeconfig: open /etc/kubernetes/kubelet.conf: no such file or directory
```

To see the stack trace of this error execute with --v=5 or higher

This is usually the proper command to upgrade a node. But this error means that this node was never even initialised, so nothing to update here. This will be done later using kubeadm join. For now we can continue with kubelet and kubectl:

```bash
➜ root@cluster3-node2:~# apt update
...
Fetched 5,775 kB in 2s (2,313 kB/s)
Reading package lists... Done
Building dependency tree
Reading state information... Done
90 packages can be upgraded. Run 'apt list --upgradable' to see them.
➜ root@cluster3-node2:~# apt show kubectl -a | grep 1.27
Version: 1.27.1-00
Version: 1.27.0-00
➜ root@cluster3-node2:~# apt install kubectl=1.27.1-00 kubelet=1.27.1-00
Reading package lists... Done
Building dependency tree
Reading state information... Done
The following packages will be upgraded:
  kubectl kubelet
2 upgraded, 0 newly installed, 0 to remove and 165 not upgraded.
Need to get 29.0 MB of archives.
After this operation, 13.9 MB disk space will be freed.
Get:1 https://packages.cloud.google.com/apt kubernetes-xenial/main amd64 kubectl amd64 1.27.1-00 [10.2 MB]
Get:2 https://packages.cloud.google.com/apt kubernetes-xenial/main amd64 kubelet amd64 1.27.1-00 [18.7 MB]
Fetched 29.0 MB in 1s (30.7 MB/s)
(Reading database ... 112510 files and directories currently installed.)
Preparing to unpack .../kubectl_1.27.1-00_amd64.deb ...
Unpacking kubectl (1.27.1-00) over (1.26.4-00) ...
Preparing to unpack .../kubelet_1.27.1-00_amd64.deb ...
Unpacking kubelet (1.27.1-00) over (1.26.4-00) ...
Setting up kubectl (1.27.1-00) ...
Setting up kubelet (1.27.1-00) ...
➜ root@cluster3-node2:~# kubelet --version
Kubernetes v1.27.1
```

Now we're up to date with kubeadm, kubectl and kubelet. Restart the kubelet:

```bash
➜ root@cluster3-node2:~# service kubelet restart
➜ root@cluster3-node2:~# service kubelet status
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
    Drop-In: /etc/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: activating (auto-restart) (Result: exit-code) since Sun 2023-04-23 14:38:13 UTC; 6s ago
       Docs: https://kubernetes.io/docs/home/
    Process: 32429 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBE>
   Main PID: 32429 (code=exited, status=1/FAILURE)
Apr 23 14:38:13 cluster3-node2 systemd[1]: kubelet.service: Main process exited, code=exited, status=1/FA>
Apr 23 14:38:13 cluster3-node2 systemd[1]: kubelet.service: Failed with result 'exit-code'.
```

These errors occur because we still need to run kubeadm join to join the node into the cluster. Let's do this in the next step.

Add cluster3-node2 to cluster

First we log into the controlplane1 and generate a new TLS bootstrap token, also printing out the join command:

```bash
➜ ssh cluster3-controlplane1
➜ root@cluster3-controlplane1:~# kubeadm token create --print-join-command
kubeadm join 192.168.100.31:6443 --token an7gxi.a1d29qy772xcxdfu --discovery-token-ca-cert-hash sha256:68fd1ebb3b96aeee04df3d5d0bb488ba4896010f2724fcff6b90d0c97be20a6b
➜ root@cluster3-controlplane1:~# kubeadm token list
TOKEN                     TTL         EXPIRES                ...
an7gxi.a1d29qy772xcxdfu   23h         2023-04-24T14:39:14Z   ...
feh5e4.h8qloqkkyq0r1cz1   <forever>   <never>                ...
z10zgm.e2vgkyv9ca9zjh0l   22h         2023-04-24T12:59:14Z   ...
```

We see the expiration of 23h for our token, we could adjust this by passing the ttl argument.

Next we connect again to cluster3-node2 and simply execute the join command:

```bash
➜ ssh cluster3-node2
➜ root@cluster3-node2:~# kubeadm join 192.168.100.31:6443 --token an7gxi.a1d29qy772xcxdfu --discovery-token-ca-cert-hash sha256:68fd1ebb3b96aeee04df3d5d0bb488ba4896010f2724fcff6b90d0c97be20a6b
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.
Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
➜ root@cluster3-node2:~# service kubelet status
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/lib/systemd/system/kubelet.service; enabled; vendor preset: enabled)
    Drop-In: /etc/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Wed 2022-12-21 16:32:19 UTC; 1min 4s ago
       Docs: https://kubernetes.io/docs/home/
   Main PID: 32510 (kubelet)
      Tasks: 11 (limit: 462)
     Memory: 55.2M
     CGroup: /system.slice/kubelet.service
             └─32510 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --container-runti>
```

If you have troubles with kubeadm join you might need to run kubeadm reset.

This looks great though for us. Finally we head back to the main terminal and check the node status:

```bash
➜ k get node
NAME                     STATUS     ROLES           AGE    VERSION
cluster3-controlplane1   Ready      control-plane   102m   v1.27.1
cluster3-node1           Ready      <none>          97m    v1.27.1
cluster3-node2           NotReady   <none>          108s   v1.27.1
```

Give it a bit of time till the node is ready.

```bash
➜ k get node
NAME                     STATUS     ROLES           AGE    VERSION
cluster3-controlplane1   Ready      control-plane   102m   v1.27.1
cluster3-node1           Ready      <none>          97m    v1.27.1
cluster3-node2           Ready      <none>          108s   v1.27.1
```

## Question 12 | Etcd Snapshot Save and Restore

Task weight: 8%

Make a backup of etcd running on cluster3-controlplane1 and save it on the controlplane node at /tmp/etcd-backup.db.

Then create a Pod of your kind in the cluster.

Finally restore the backup, confirm the cluster is still working and that the created Pod is no longer with us.

**Answer:**

Etcd Backup

First we log into the controlplane and try to create a snapshop of etcd:

```bash
➜ ssh cluster3-controlplane1
➜ root@cluster3-controlplane1:~# ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db
Error:  rpc error: code = Unavailable desc = transport is closing
```

But it fails because we need to authenticate ourselves. For the necessary information we can check the etc manifest:

```bash
➜ root@cluster3-controlplane1:~# vim /etc/kubernetes/manifests/etcd.yaml
```

We only check the etcd.yaml for necessary information we don't change it.

```yaml
# /etc/kubernetes/manifests/etcd.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: etcd
    tier: control-plane
  name: etcd
  namespace: kube-system
spec:
  containers:
  - command:
    - etcd
    - --advertise-client-urls=https://192.168.100.31:2379
    - --cert-file=/etc/kubernetes/pki/etcd/server.crt                           # use
    - --client-cert-auth=true
    - --data-dir=/var/lib/etcd
    - --initial-advertise-peer-urls=https://192.168.100.31:2380
    - --initial-cluster=cluster3-controlplane1=https://192.168.100.31:2380
    - --key-file=/etc/kubernetes/pki/etcd/server.key                            # use
    - --listen-client-urls=https://127.0.0.1:2379,https://192.168.100.31:2379   # use
    - --listen-metrics-urls=http://127.0.0.1:2381
    - --listen-peer-urls=https://192.168.100.31:2380
    - --name=cluster3-controlplane1
    - --peer-cert-file=/etc/kubernetes/pki/etcd/peer.crt
    - --peer-client-cert-auth=true
    - --peer-key-file=/etc/kubernetes/pki/etcd/peer.key
    - --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt                    # use
    - --snapshot-count=10000
    - --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.crt
    image: k8s.gcr.io/etcd:3.3.15-0
    imagePullPolicy: IfNotPresent
    livenessProbe:
      failureThreshold: 8
      httpGet:
        host: 127.0.0.1
        path: /health
        port: 2381
        scheme: HTTP
      initialDelaySeconds: 15
      timeoutSeconds: 15
    name: etcd
    resources: {}
    volumeMounts:
    - mountPath: /var/lib/etcd
      name: etcd-data
    - mountPath: /etc/kubernetes/pki/etcd
      name: etcd-certs
  hostNetwork: true
  priorityClassName: system-cluster-critical
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd                                                     # important
      type: DirectoryOrCreate
    name: etcd-data
status: {}
```

But we also know that the api-server is connecting to etcd, so we can check how its manifest is configured:

```bash
➜ root@cluster3-controlplane1:~# cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep etcd
    - --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt
    - --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt
    - --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key
    - --etcd-servers=https://127.0.0.1:2379
```

We use the authentication information and pass it to etcdctl:

```bash
➜ root@cluster3-controlplane1:~# ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db \
--cacert /etc/kubernetes/pki/etcd/ca.crt \
--cert /etc/kubernetes/pki/etcd/server.crt \
--key /etc/kubernetes/pki/etcd/server.key
Snapshot saved at /tmp/etcd-backup.db
```

NOTE: Dont use snapshot status because it can alter the snapshot file and render it invalid

Etcd restore

Now create a Pod in the cluster and wait for it to be running:

```bash
➜ root@cluster3-controlplane1:~# kubectl run test --image=nginx
pod/test created
➜ root@cluster3-controlplane1:~# kubectl get pod -l run=test -w
NAME   READY   STATUS    RESTARTS   AGE
test   1/1     Running   0          60s
```

Next we stop all controlplane components:

```bash
root@cluster3-controlplane1:~# cd /etc/kubernetes/manifests/
root@cluster3-controlplane1:/etc/kubernetes/manifests# mv * ..
root@cluster3-controlplane1:/etc/kubernetes/manifests# watch crictl ps
```

Now we restore the snapshot into a specific directory:

```bash
➜ root@cluster3-controlplane1:~# ETCDCTL_API=3 etcdctl snapshot restore /tmp/etcd-backup.db \
--data-dir /var/lib/etcd-backup \
--cacert /etc/kubernetes/pki/etcd/ca.crt \
--cert /etc/kubernetes/pki/etcd/server.crt \
--key /etc/kubernetes/pki/etcd/server.key
2020-09-04 16:50:19.650804 I | mvcc: restore compact to 9935
2020-09-04 16:50:19.659095 I | etcdserver/membership: added member 8e9e05c52164694d [http://localhost:2380] to cluster cdf818194e3a8c32
```

We could specify another host to make the backup from by using etcdctl --endpoints http://IP, but here we just use the default value which is: http://127.0.0.1:2379,http://127.0.0.1:4001.

The restored files are located at the new folder /var/lib/etcd-backup, now we have to tell etcd to use that directory:

```yaml
➜ root@cluster3-controlplane1:~# vim /etc/kubernetes/etcd.yaml
# /etc/kubernetes/etcd.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: etcd
    tier: control-plane
  name: etcd
  namespace: kube-system
spec:
...
    - mountPath: /etc/kubernetes/pki/etcd
      name: etcd-certs
  hostNetwork: true
  priorityClassName: system-cluster-critical
  volumes:
  - hostPath:
      path: /etc/kubernetes/pki/etcd
      type: DirectoryOrCreate
    name: etcd-certs
  - hostPath:
      path: /var/lib/etcd-backup                # change
      type: DirectoryOrCreate
    name: etcd-data
status: {}
```

Now we move all controlplane yaml again into the manifest directory. Give it some time (up to several minutes) for etcd to restart and for the api-server to be reachable again:

```bash
root@cluster3-controlplane1:/etc/kubernetes/manifests# mv ../*.yaml .
root@cluster3-controlplane1:/etc/kubernetes/manifests# watch crictl ps
```

Then we check again for the Pod:

```bash
➜ root@cluster3-controlplane1:~# kubectl get pod -l run=test
No resources found in default namespace.
```

Awesome, backup and restore worked as our pod is gone.
