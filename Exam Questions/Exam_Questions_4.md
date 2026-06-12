# Exam Questions 4

## Question 16 | Contexts

Task weight: 1%

You have access to multiple clusters from your main terminal through kubectl contexts. Write all those context names into /opt/course/1/contexts.

Next write a command to display the current context into /opt/course/16/context_default_kubectl.sh, the command should use kubectl.

Finally write a second command doing the same thing into /opt/course/16/context_default_no_kubectl.sh, but without the use of kubectl

**Answer:**

Maybe the fastest way is just to run:

```bash
k config get-contexts # copy manually
k config get-contexts -o name > /opt/course/16/contexts
```

Or using jsonpath:

```bash
k config view -o yaml # overview
k config view -o jsonpath="{.contexts[*].name}"
k config view -o jsonpath="{.contexts[*].name}" | tr " " "\n" # new lines
k config view -o jsonpath="{.contexts[*].name}" | tr " " "\n" > /opt/course/16/contexts
```

The content should then look like:

```bash
# /opt/course/16/contexts
k8s-c1-H
k8s-c2-AC
k8s-c3-CCC
```

Next create the first command:

```bash
# /opt/course/16/context_default_kubectl.sh
kubectl config current-context
➜ sh /opt/course/16/context_default_kubectl.sh
k8s-c1-H
```

And the second one:

```bash
# /opt/course/16/context_default_no_kubectl.sh
cat ~/.kube/config | grep current
➜ sh /opt/course/16/context_default_no_kubectl.sh
current-context: k8s-c1-H
```

In the real exam you might need to filter and find information from bigger lists of resources, hence knowing a little jsonpath and simple bash filtering will be helpful.

The second command could also be improved to:

```bash
# /opt/course/16/context_default_no_kubectl.sh
cat ~/.kube/config | grep current | sed -e "s/current-context: //"
```

## Question 17 | Pod Ready if Service is reachable

Task weight: 4%

Use context: kubectl config use-context k8s-c1-H

Do the following in Namespace default. Create a single Pod named ready-if-service-ready of image nginx:1.16.1-alpine. Configure a LivenessProbe which simply executes command true. Also configure a ReadinessProbe which does check if the url http://service-am-i-ready:80 is reachable, you can use wget -T2 -O- http://service-am-i-ready:80 for this. Start the Pod and confirm it isn't ready because of the ReadinessProbe.

Create a second Pod named am-i-ready of image nginx:1.16.1-alpine with label id: cross-server-ready. The already existing Service service-am-i-ready should now have that second Pod as endpoint.

Now the first Pod should be in ready state, confirm that.

**Answer:**

It's a bit of an anti-pattern for one Pod to check another Pod for being ready using probes, hence the normally available readinessProbe.httpGet doesn't work for absolute remote urls. Still the workaround requested in this task should show how probes and Pod<->Service communication works.

First we create the first Pod:

```bash
k run ready-if-service-ready --image=nginx:1.16.1-alpine $do > 4_pod1.yaml
vim 4_pod1.yaml
```

Next perform the necessary additions manually:

```yaml
# 4_pod1.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: ready-if-service-ready
  name: ready-if-service-ready
spec:
  containers:
  - image: nginx:1.16.1-alpine
    name: ready-if-service-ready
    resources: {}
    livenessProbe:                                      # add from here
      exec:
        command:
        - 'true'
    readinessProbe:
      exec:
        command:
        - sh
        - -c
        - 'wget -T2 -O- http://service-am-i-ready:80'   # to here
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}
```

Then create the Pod:

```bash
k -f 4_pod1.yaml create
```

And confirm it's in a non-ready state:

```bash
➜ k get pod ready-if-service-ready
NAME                     READY   STATUS    RESTARTS   AGE
ready-if-service-ready   0/1     Running   0          7s
```

We can also check the reason for this using describe:

```bash
➜ k describe pod ready-if-service-ready
 ...
  Warning  Unhealthy  18s   kubelet, cluster1-node1  Readiness probe failed: Connecting to service-am-i-ready:80 (10.109.194.234:80)
wget: download timed out
```

Now we create the second Pod:

```bash
k run am-i-ready --image=nginx:1.16.1-alpine --labels="id=cross-server-ready"
```

The already existing Service service-am-i-ready should now have an Endpoint:

```bash
k describe svc service-am-i-ready
k get ep # also possible
```

Which will result in our first Pod being ready, just give it a minute for the Readiness probe to check again:

```bash
➜ k get pod ready-if-service-ready
NAME                     READY   STATUS    RESTARTS   AGE
ready-if-service-ready   1/1     Running   0          53s
```

Look at these Pods coworking together!

## Question 18 | Kubectl sorting

Task weight: 1%

Use context: kubectl config use-context k8s-c1-H

There are various Pods in all namespaces. Write a command into /opt/course/5/find_pods.sh which lists all Pods sorted by their AGE (metadata.creationTimestamp).

Write a second command into /opt/course/5/find_pods_uid.sh which lists all Pods sorted by field metadata.uid. Use kubectl sorting for both commands.

**Answer:**

A good resources here (and for many other things) is the kubectl-cheat-sheet. You can reach it fast when searching for "cheat sheet" in the Kubernetes docs.

```bash
# /opt/course/5/find_pods.sh
kubectl get pod -A --sort-by=.metadata.creationTimestamp
```

And to execute:

```bash
➜ sh /opt/course/5/find_pods.sh
NAMESPACE         NAME                                             ...          AGE
kube-system       kube-scheduler-cluster1-controlplane1            ...          63m
kube-system       etcd-cluster1-controlplane1                      ...          63m
kube-system       kube-apiserver-cluster1-controlplane1            ...          63m
kube-system       kube-controller-manager-cluster1-controlplane1   ...          63m
...
```

For the second command:

```bash
# /opt/course/5/find_pods_uid.sh
kubectl get pod -A --sort-by=.metadata.uid
```

And to execute:

```bash
➜ sh /opt/course/5/find_pods_uid.sh
NAMESPACE         NAME                                      ...          AGE
kube-system       coredns-5644d7b6d9-vwm7g                  ...          68m
project-c13       c13-3cc-runner-heavy-5486d76dd4-ddvlt     ...          63m
project-hamster   web-hamster-shop-849966f479-278vp         ...          63m
project-c13       c13-3cc-web-646b6c8756-qsg4b              ...          63m
```

## Question 19 | Node and Pod Resource Usage

Task weight: 1%

Use context: kubectl config use-context k8s-c1-H

The metrics-server has been installed in the cluster. Your college would like to know the kubectl commands to:

show Nodes resource usage

show Pods and their containers resource usage

Please write the commands into /opt/course/7/node.sh and /opt/course/7/pod.sh.

**Answer:**

The command we need to use here is top:

```bash
➜ k top -h
```

Display Resource (CPU/Memory/Storage) usage.

The top command allows you to see the resource consumption for nodes or pods.

This command requires Metrics Server to be correctly configured and working on the server.

Available Commands:

  node        Display Resource (CPU/Memory/Storage) usage of nodes

  pod         Display Resource (CPU/Memory/Storage) usage of pods

We see that the metrics server provides information about resource usage:

```bash
➜ k top node
NAME               CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
cluster1-controlplane1   178m         8%     1091Mi          57%
cluster1-node1   66m          6%     834Mi           44%
cluster1-node2   91m          9%     791Mi           41%
```

We create the first file:

```bash
# /opt/course/7/node.sh
kubectl top node
```

For the second file we might need to check the docs again:

```bash
➜ k top pod -h
Display Resource (CPU/Memory/Storage) usage of pods.
...
Namespace in current context is ignored even if specified with --namespace.
      --containers=false: If present, print usage of containers within a pod.
      --no-headers=false: If present, print output without headers.
...
```

With this we can finish this task:

```bash
# /opt/course/7/pod.sh
kubectl top pod --containers=true
```

## Question 20 | Get Controlplane Information

Task weight: 2%

Use context: kubectl config use-context k8s-c1-H

Ssh into the controlplane node with ssh cluster1-controlplane1. Check how the controlplane components kubelet, kube-apiserver, kube-scheduler, kube-controller-manager and etcd are started/installed on the controlplane node. Also find out the name of the DNS application and how it's started/installed on the controlplane node.

Write your findings into file /opt/course/8/controlplane-components.txt. The file should be structured like:

```bash
# /opt/course/8/controlplane-components.txt
kubelet: [TYPE]
kube-apiserver: [TYPE]
kube-scheduler: [TYPE]
kube-controller-manager: [TYPE]
etcd: [TYPE]
dns: [TYPE] [NAME]
```

Choices of [TYPE] are: not-installed, process, static-pod, pod

**Answer:**

We could start by finding processes of the requested components, especially the kubelet at first:

```bash
➜ ssh cluster1-controlplane1
root@cluster1-controlplane1:~# ps aux | grep kubelet # shows kubelet process
```

We can see which components are controlled via systemd looking at /etc/systemd/system directory:

```bash
➜ root@cluster1-controlplane1:~# find /etc/systemd/system/ | grep kube
/etc/systemd/system/kubelet.service.d
/etc/systemd/system/kubelet.service.d/10-kubeadm.conf
/etc/systemd/system/multi-user.target.wants/kubelet.service
➜ root@cluster1-controlplane1:~# find /etc/systemd/system/ | grep etcd
```

This shows kubelet is controlled via systemd, but no other service named kube nor etcd. It seems that this cluster has been setup using kubeadm, so we check in the default manifests directory:

```bash
➜ root@cluster1-controlplane1:~# find /etc/kubernetes/manifests/
/etc/kubernetes/manifests/
/etc/kubernetes/manifests/kube-controller-manager.yaml
/etc/kubernetes/manifests/etcd.yaml
/etc/kubernetes/manifests/kube-apiserver.yaml
/etc/kubernetes/manifests/kube-scheduler.yaml
```

(The kubelet could also have a different manifests directory specified via parameter --pod-manifest-path in it's systemd startup config)

This means the main 4 controlplane services are setup as static Pods. Actually, let's check all Pods running on in the kube-system Namespace on the controlplane node:

```bash
➜ root@cluster1-controlplane1:~# kubectl -n kube-system get pod -o wide | grep controlplane1
coredns-5644d7b6d9-c4f68                            1/1     Running            ...   cluster1-controlplane1
coredns-5644d7b6d9-t84sc                            1/1     Running            ...   cluster1-controlplane1
etcd-cluster1-controlplane1                         1/1     Running            ...   cluster1-controlplane1
kube-apiserver-cluster1-controlplane1               1/1     Running            ...   cluster1-controlplane1
kube-controller-manager-cluster1-controlplane1      1/1     Running            ...   cluster1-controlplane1
kube-proxy-q955p                                    1/1     Running            ...   cluster1-controlplane1
kube-scheduler-cluster1-controlplane1               1/1     Running            ...   cluster1-controlplane1
weave-net-mwj47                                     2/2     Running            ...   cluster1-controlplane1
```

There we see the 5 static pods, with -cluster1-controlplane1 as suffix.

We also see that the dns application seems to be coredns, but how is it controlled?

```bash
➜ root@cluster1-controlplane1$ kubectl -n kube-system get ds
NAME         DESIRED   CURRENT   ...   NODE SELECTOR            AGE
kube-proxy   3         3         ...   kubernetes.io/os=linux   155m
weave-net    3         3         ...   <none>                   155m
➜ root@cluster1-controlplane1$ kubectl -n kube-system get deploy
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   2/2     2            2           155m
```

Seems like coredns is controlled via a Deployment. We combine our findings in the requested file:

```bash
# /opt/course/8/controlplane-components.txt
kubelet: process
kube-apiserver: static-pod
kube-scheduler: static-pod
kube-controller-manager: static-pod
etcd: static-pod
dns: pod coredns
```

You should be comfortable investigating a running cluster, know different methods on how a cluster and its services can be setup and be able to troubleshoot and find error sources.
