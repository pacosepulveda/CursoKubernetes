# Exam Questions 1

## Pre-setup

Once you've gained access to your terminal it might be wise to spend ~1 minute to setup your environment. You could set these:

```
alias k=kubectl    # will already be pre-configured
export do="--dry-run=client -o yaml"    # k create deploy nginx --image=nginx $do
export now="--force --grace-period 0"   # k delete pod x $now
```

## Question 1 | Scale down StatefulSet

**Task weight: 1%**

Start creating two Pods named o3db-* in Namespace project-c13. Management asked you to scale the Pods down to one replica to save resources.

If we check the Pods we see two replicas:

```
➜ k -n project-c13 get pod | grep o3db
o3db-0                                  1/1     Running   0          52s
o3db-1                                  1/1     Running   0          42s
```

From their name it looks like these are managed by a StatefulSet. But if we're not sure we could also check for the most common resources which manage Pods:

```
➜ k -n project-c13 get deploy,ds,sts | grep o3db
statefulset.apps/o3db   2/2     2m56s
```

Confirmed, we have to work with a StatefulSet. To find this out we could also look at the Pod labels:

```
➜ k -n project-c13 get pod --show-labels | grep o3db
o3db-0                                  1/1     Running   0          3m29s   app=nginx,controller-revision-hash=o3db-5fbd4bb9cc,statefulset.kubernetes.io/pod-name=o3db-0
o3db-1                                  1/1     Running   0          3m19s   app=nginx,controller-revision-hash=o3db-5fbd4bb9cc,statefulset.kubernetes.io/pod-name=o3db-1
```

To fulfil the task we simply run:

```
➜ k -n project-c13 scale sts o3db --replicas 1
statefulset.apps/o3db scaled
➜ k -n project-c13 get sts o3db
NAME   READY   AGE
o3db   1/1     4m39s
```

## Question 2 | Kubectl sorting

There are various Pods in all namespaces. Write a command into /opt/course/5/find_pods.sh which lists all Pods sorted by their AGE (metadata.creationTimestamp).

Write a second command into /opt/course/5/find_pods_uid.sh which lists all Pods sorted by field metadata.uid. Use kubectl sorting for both commands.

### Answer

A good resources here (and for many other things) is the kubectl-cheat-sheet. You can reach it fast when searching for "cheat sheet" in the Kubernetes docs.

<https://kubernetes.io/docs/reference/kubectl/cheatsheet/>

```
# /opt/course/5/find_pods.sh
kubectl get pod -A --sort-by=.metadata.creationTimestamp
```

And to execute:

```
➜ sh /opt/course/5/find_pods.sh
NAMESPACE         NAME                                             ...          AGE
kube-system       kube-scheduler-cluster1-controlplane1            ...          63m
kube-system       etcd-cluster1-controlplane1                      ...          63m
kube-system       kube-apiserver-cluster1-controlplane1            ...          63m
kube-system       kube-controller-manager-cluster1-controlplane1   ...          63m
...
```

For the second command:

```
# /opt/course/5/find_pods_uid.sh
kubectl get pod -A --sort-by=.metadata.uid
```

And to execute:

```
➜ sh /opt/course/5/find_pods_uid.sh
NAMESPACE         NAME                                      ...          AGE
kube-system       coredns-5644d7b6d9-vwm7g                  ...          68m
project-c13       c13-3cc-runner-heavy-5486d76dd4-ddvlt     ...          63m
project-hamster   web-hamster-shop-849966f479-278vp         ...          63m
project-c13       c13-3cc-web-646b6c8756-qsg4b              ...          63m
```

## Question 3 | Get Controlplane Information

Ssh into the controlplane node. Check how the controlplane components kubelet, kube-apiserver, kube-scheduler, kube-controller-manager and etcd are started/installed on the controlplane node. Also find out the name of the DNS application and how it's started/installed on the controlplane node.

Write your findings into file /opt/course/8/controlplane-components.txt. The file should be structured like:

```
# /opt/course/8/controlplane-components.txt
kubelet: [TYPE]
kube-apiserver: [TYPE]
kube-scheduler: [TYPE]
kube-controller-manager: [TYPE]
etcd: [TYPE]
dns: [TYPE] [NAME]
```

Choices of [TYPE] are: not-installed, process, static-pod, pod.

### Answer

We could start by finding processes of the requested components, especially the kubelet at first:

```
root@cluster1-controlplane1:~# ps aux | grep kubelet # shows kubelet process
```

We can see which components are controlled via systemd looking at /etc/systemd/system directory:

```
➜ root@cluster1-controlplane1:~# find /etc/systemd/system/ | grep kube
/etc/systemd/system/kubelet.service.d
/etc/systemd/system/kubelet.service.d/10-kubeadm.conf
/etc/systemd/system/multi-user.target.wants/kubelet.service
➜ root@cluster1-controlplane1:~# find /etc/systemd/system/ | grep etcd
```

This shows kubelet is controlled via systemd, but no other service named kube nor etcd. It seems that this cluster has been setup using kubeadm, so we check in the default manifests directory:

```
➜ root@cluster1-controlplane1:~# find /etc/kubernetes/manifests/
/etc/kubernetes/manifests/
/etc/kubernetes/manifests/kube-controller-manager.yaml
/etc/kubernetes/manifests/etcd.yaml
/etc/kubernetes/manifests/kube-apiserver.yaml
/etc/kubernetes/manifests/kube-scheduler.yaml
```

(The kubelet could also have a different manifests directory specified via parameter --pod-manifest-path in it's systemd startup config)

This means the main 4 controlplane services are setup as static Pods. Actually, let's check all Pods running on in the kube-system Namespace on the controlplane node:

```
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

```
➜ root@cluster1-controlplane1$ kubectl -n kube-system get ds
NAME         DESIRED   CURRENT   ...   NODE SELECTOR            AGE
kube-proxy   3         3         ...   kubernetes.io/os=linux   155m
weave-net    3         3         ...   <none>                   155m
➜ root@cluster1-controlplane1$ kubectl -n kube-system get deploy
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   2/2     2            2           155m
```

Seems like coredns is controlled via a Deployment. We combine our findings in the requested file:

```
# /opt/course/8/controlplane-components.txt
kubelet: process
kube-apiserver: static-pod
kube-scheduler: static-pod
kube-controller-manager: static-pod
etcd: static-pod
dns: pod coredns
```

You should be comfortable investigating a running cluster, know different methods on how a cluster and its services can be setup and be able to troubleshoot and find error sources.

## Question 4 | Kill Scheduler, Manual Scheduling

Ssh into the controlplane node. Temporarily stop the kube-scheduler, this means in a way that you can start it again afterwards.

Create a single Pod named manual-schedule of image httpd:2.4-alpine, confirm it's created but not scheduled on any node.

Now you're the scheduler and have all its power, manually schedule that Pod on node Worker1. Make sure it's running.

Start the kube-scheduler again and confirm it's running correctly by creating a second Pod named manual-schedule2 of image httpd:2.4-alpine and check if it's running.

### Answer

Check if the scheduler is running:

```
➜ root@master:~# kubectl -n kube-system get pod | grep schedule
kube-scheduler-master            1/1     Running   0          6s
```

Kill the Scheduler (temporarily):

```
➜ root@master:~# cd /etc/kubernetes/manifests/
➜ root@master:~# mv kube-scheduler.yaml ..
```

And it should be stopped:

```
➜ root@master:~# kubectl -n kube-system get pod | grep schedule
➜ root@master:~#
```

#### Create a Pod

Now we create the Pod:

```
k run manual-schedule --image=httpd:2.4-alpine
```

And confirm it has no node assigned:

```
➜ k get pod manual-schedule -o wide
NAME              READY   STATUS    ...   NODE     NOMINATED NODE
manual-schedule   0/1     Pending   ...   <none>   <none>
```

#### Manually schedule the Pod

Let's play the scheduler now:

```
k get pod manual-schedule -o yaml > 9.yaml
# 9.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2020-09-04T15:51:02Z"
  labels:
    run: manual-schedule
  managedFields:
...
    manager: kubectl-run
    operation: Update
    time: "2020-09-04T15:51:02Z"
  name: manual-schedule
  namespace: default
  resourceVersion: "3515"
  selfLink: /api/v1/namespaces/default/pods/manual-schedule
  uid: 8e9d2532-4779-4e63-b5af-feb82c74a935
spec:
  nodeName: Worker1        # add the Worker1 node name
  containers:
  - image: httpd:2.4-alpine
    imagePullPolicy: IfNotPresent
    name: manual-schedule
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-nxnc7
      readOnly: true
  dnsPolicy: ClusterFirst
...
```

The only thing a scheduler does, is that it sets the nodeName for a Pod declaration. How it finds the correct node to schedule on, that's a very much complicated matter and takes many variables into account.

As we cannot kubectl apply or kubectl edit , in this case we need to delete and create or replace:

```
k -f 9.yaml replace --force
```

How does it look?

```
➜ k get pod manual-schedule -o wide
NAME              READY   STATUS    ...   NODE
manual-schedule   1/1     Running   ...   Worker1
```

It looks like our Pod is running on the controlplane now as requested, although no tolerations were specified. Only the scheduler takes tains/tolerations/affinity into account when finding the correct node name. That's why it's still possible to assign Pods manually directly to a controlplane node and skip the scheduler.

#### Start the scheduler again

```
➜ root@master:~# cd /etc/kubernetes/manifests/
➜ root@master:~# mv ../kube-scheduler.yaml .
```

Checks it's running:

```
➜ root@master:~# kubectl -n kube-system get pod | grep schedule
kube-scheduler-cluster2-controlplane1            1/1     Running   0          16s
```

Schedule a second test Pod:

```
k run manual-schedule2 --image=httpd:2.4-alpine
➜ k get pod -o wide | grep schedule
manual-schedule    1/1     Running   ...   Worker1
manual-schedule2   1/1     Running   ...   Worker2
```

Back to normal.

## Question 5 | DaemonSet on all Nodes

Use Namespace project-tiger for the following. Create a DaemonSet named ds-important with image httpd:2.4-alpine and labels id=ds-important and uuid=18426a0b-5f59-4e10-923f-c0e078e82462. The Pods it creates should request 10 millicore cpu and 10 mebibyte memory. The Pods of that DaemonSet should run on all nodes, also controlplanes.

### Answer

As of now we aren't able to create a DaemonSet directly using kubectl, so we create a Deployment and just change it up:

```
k -n project-tiger create deployment --image=httpd:2.4-alpine ds-important $do > 11.yaml
vim 11.yaml
```

(Sure you could also search for a DaemonSet example yaml in the Kubernetes docs and alter it.)

Then we adjust the yaml to:

```yaml
# 11.yaml
apiVersion: apps/v1
kind: DaemonSet                                     # change from Deployment to Daemonset
metadata:
  creationTimestamp: null
  labels:                                           # add
    id: ds-important                                # add
    uuid: 18426a0b-5f59-4e10-923f-c0e078e82462      # add
  name: ds-important
  namespace: project-tiger                          # important
spec:
  #replicas: 1                                      # remove
  selector:
    matchLabels:
      id: ds-important                              # add
      uuid: 18426a0b-5f59-4e10-923f-c0e078e82462    # add
  #strategy: {}                                     # remove
  template:
    metadata:
      creationTimestamp: null
      labels:
        id: ds-important                            # add
        uuid: 18426a0b-5f59-4e10-923f-c0e078e82462  # add
    spec:
      containers:
      - image: httpd:2.4-alpine
        name: ds-important
        resources:
          requests:                                 # add
            cpu: 10m                                # add
            memory: 10Mi                            # add
      tolerations:                                  # add
      - effect: NoSchedule                          # add
        key: node-role.kubernetes.io/control-plane  # add
#status: {}                                         # remove
```

It was requested that the DaemonSet runs on all nodes, so we need to specify the toleration for this.

Let's confirm:

```
k -f 11.yaml create
➜ k -n project-tiger get ds
NAME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
ds-important   3         3         3       3            3           <none>          8s
➜ k -n project-tiger get pod -l id=ds-important -o wide
NAME                      READY   STATUS          NODE
ds-important-6pvgm        1/1     Running   ...   worker1
ds-important-lh5ts        1/1     Running   ...   master
ds-important-qhjcq        1/1     Running   ...   worker2
```
