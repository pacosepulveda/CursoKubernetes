# Exam Questions 2

## Question 6 | Schedule Pod on Controlplane Node

**Task weight:** 3%

Create a single Pod of image `httpd:2.4.41-alpine` in Namespace `default`. The Pod should be named `pod1` and the container should be named `pod1-container`. This Pod should only be scheduled on a controlplane node, do not add new labels any nodes.

### Answer

First we find the controlplane node(s) and their taints:

```bash
k get node # find controlplane node
k describe node cluster1-controlplane1 | grep Taint -A1 # get controlplane node taints
k get node cluster1-controlplane1 --show-labels # get controlplane node labels
```

Next we create the Pod template:

```bash
k run pod1 --image=httpd:2.4.41-alpine $do > 6.yaml
vim 6.yaml
```

Perform the necessary changes manually. Use the Kubernetes docs and search for example for tolerations and `nodeSelector` to find examples:

```yaml
# 6.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: pod1
  name: pod1
spec:
  containers:
    - image: httpd:2.4.41-alpine
      name: pod1-container # change
      resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  tolerations: # add
    - effect: NoSchedule # add
      key: node-role.kubernetes.io/control-plane # add
  nodeSelector: # add
    node-role.kubernetes.io/control-plane: "" # add
status: {}
```

Important here to add the toleration for running on controlplane nodes, but also the `nodeSelector` to make sure it only runs on controlplane nodes. If we only specify a toleration the Pod can be scheduled on controlplane or worker nodes.

Now we create it:

```bash
k -f 6.yaml create
```

---

## Question 7 | Storage, PV, PVC, Pod volume

**Task weight:** 8%

Create a new PersistentVolume named `safari-pv`. It should have a capacity of `2Gi`, accessMode `ReadWriteOnce`, hostPath `/Volumes/Data` and no `storageClassName` defined.

Next create a new PersistentVolumeClaim in Namespace `project-tiger` named `safari-pvc`. It should request `2Gi` storage, accessMode `ReadWriteOnce` and should not define a `storageClassName`. The PVC should bound to the PV correctly.

Finally create a new Deployment `safari` in Namespace `project-tiger` which mounts that volume at `/tmp/safari-data`. The Pods of that Deployment should be of image `httpd:2.4.41-alpine`.

### Answer

```bash
vim 7_pv.yaml
```

Find an example from <https://kubernetes.io/docs> and alter it:

```yaml
# 7_pv.yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: safari-pv
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/Volumes/Data"
```

Then create it:

```bash
k -f 7_pv.yaml create
```

Next the PersistentVolumeClaim:

```bash
vim 7_pvc.yaml
```

Find an example from <https://kubernetes.io/docs> and alter it:

```yaml
# 7_pvc.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: safari-pvc
  namespace: project-tiger
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

Then create:

```bash
k -f 7_pvc.yaml create
```

And check that both have the status `Bound`:

```bash
k -n project-tiger get pv,pvc
```

Next we create a Deployment and mount that volume:

```bash
k -n project-tiger create deploy safari --image=httpd:2.4.41-alpine $do > 7_dep.yaml
vim 7_dep.yaml
```

Alter the YAML to mount the volume:

```yaml
# 7_dep.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: safari
  name: safari
  namespace: project-tiger
spec:
  replicas: 1
  selector:
    matchLabels:
      app: safari
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: safari
    spec:
      volumes: # add
        - name: data # add
          persistentVolumeClaim: # add
            claimName: safari-pvc # add
      containers:
        - image: httpd:2.4.41-alpine
          name: container
          volumeMounts: # add
            - name: data # add
              mountPath: /tmp/safari-data # add
```

```bash
k -f 7_dep.yaml create
```

---

## Question 7 | Multi Containers and Pod shared Volume

**Task weight:** 4%

Create a Pod named `multi-container-playground` in Namespace `default` with three containers, named `c1`, `c2` and `c3`. There should be a volume attached to that Pod and mounted into every container, but the volume shouldn't be persisted or shared with other Pods.

Container `c1` should be of image `nginx:1.17.6-alpine` and have the name of the node where its Pod is running available as environment variable `MY_NODE_NAME`.

Container `c2` should be of image `busybox:1.31.1` and write the output of the `date` command every second in the shared volume into file `date.log`. You can use `while true; do date >> /your/vol/path/date.log; sleep 1; done` for this.

Container `c3` should be of image `busybox:1.31.1` and constantly send the content of file `date.log` from the shared volume to stdout. You can use `tail -f /your/vol/path/date.log` for this.

Check the logs of container `c3` to confirm correct setup.

### Answer

First we create the Pod template:

```bash
k run multi-container-playground --image=nginx:1.17.6-alpine $do > 8.yaml
vim 8.yaml
```

And add the other containers and the commands they should execute:

```yaml
# 8.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: multi-container-playground
  name: multi-container-playground
spec:
  containers:
    - image: nginx:1.17.6-alpine
      name: c1 # change
      resources: {}
      env: # add
        - name: MY_NODE_NAME # add
          valueFrom: # add
            fieldRef: # add
              fieldPath: spec.nodeName # add
      volumeMounts: # add
        - name: vol # add
          mountPath: /vol # add
    - image: busybox:1.31.1 # add
      name: c2 # add
      command: ["sh", "-c", "while true; do date >> /vol/date.log; sleep 1; done"] # add
      volumeMounts: # add
        - name: vol # add
          mountPath: /vol # add
    - image: busybox:1.31.1 # add
      name: c3 # add
      command: ["sh", "-c", "tail -f /vol/date.log"] # add
      volumeMounts: # add
        - name: vol # add
          mountPath: /vol # add
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes: # add
    - name: vol # add
      emptyDir: {} # add
status: {}
```

```bash
k -f 8.yaml create
```

We check if everything is good with the Pod:

```bash
k get pod multi-container-playground
```

Good, then we check if container `c1` has the requested node name as env variable:

```bash
k exec multi-container-playground -c c1 -- env | grep MY
```

And finally we check the logging:

```bash
k logs multi-container-playground -c c3
```

---

## Question 9 | Cluster Event Logging

**Task weight:** 3%

Write a command into `/opt/course/15/cluster_events.sh` which shows the latest events in the whole cluster, ordered by time (`metadata.creationTimestamp`). Use `kubectl` for it.

Now kill the `kube-proxy` Pod running on node `cluster2-node1` and write the events this caused into `/opt/course/15/pod_kill.log`.

Finally kill the containerd container of the `kube-proxy` Pod on node `cluster2-node1` and write the events into `/opt/course/15/container_kill.log`.

Do you notice differences in the events both actions caused?

### Answer

```bash
# /opt/course/9/cluster_events.sh
kubectl get events -A --sort-by=.metadata.creationTimestamp
```

Now we kill the `kube-proxy` Pod:

```bash
k -n kube-system get pod -o wide | grep proxy # find pod running on cluster2-node1
k -n kube-system delete pod kube-proxy-z64cg
```

Now check the events:

```bash
sh /opt/course/9/cluster_events.sh
```

Write the events the killing caused into `/opt/course/15/pod_kill.log`:

```text
# /opt/course/9/pod_kill.log
kube-system 9s Normal Killing pod/kube-proxy-jsv7t ...
kube-system 3s Normal SuccessfulCreate daemonset/kube-proxy ...
kube-system <unknown> Normal Scheduled pod/kube-proxy-m52sx ...
default 2s Normal Starting node/cluster2-node1 ...
kube-system 2s Normal Created pod/kube-proxy-m52sx ...
kube-system 2s Normal Pulled pod/kube-proxy-m52sx ...
kube-system 2s Normal Started pod/kube-proxy-m52sx ...
```

Finally we will try to provoke events by killing the container belonging to the container of the `kube-proxy` Pod:

```bash
ssh cluster2-node1
crictl ps | grep kube-proxy
crictl rm 1e020b43c4423
crictl ps | grep kube-proxy
```

Example output:

```text
1e020b43c4423 36c4ebbc9d979 About an hour ago Running kube-proxy ...
1e020b43c4423
0ae4245707910 36c4ebbc9d979 17 seconds ago Running kube-proxy ...
```

We killed the main container (`1e020b43c4423`), but also noticed that a new container (`0ae4245707910`) was directly created.

Now we see if this caused events again and we write those into the second file:

```bash
sh /opt/course/9/cluster_events.sh
```

```text
# /opt/course/9/container_kill.log
kube-system 13s Normal Created pod/kube-proxy-m52sx ...
kube-system 13s Normal Pulled pod/kube-proxy-m52sx ...
kube-system 13s Normal Started pod/kube-proxy-m52sx ...
```

Comparing the events we see that when we deleted the whole Pod there were more things to be done, hence more events. For example was the DaemonSet in the game to re-create the missing Pod. Where when we manually killed the main container of the Pod, the Pod would still exist but only its container needed to be re-created, hence less events.

---

## Question 10 | Create Secret and mount into Pod

**Task weight:** 3%

Do the following in a new Namespace `secret`. Create a Pod named `secret-pod` of image `busybox:1.31.1` which should keep running for some time.

There is an existing Secret located at `/opt/course/10/secret1.yaml`, create it in the Namespace `secret` and mount it readonly into the Pod at `/tmp/secret1`.

Create a new Secret in Namespace `secret` called `secret2` which should contain `user=user1` and `pass=1234`. These entries should be available inside the Pod's container as environment variables `APP_USER` and `APP_PASS`.

Confirm everything is working.

### Answer

First we create the Namespace and the requested Secrets in it:

```bash
k create ns secret
cp /opt/course/10/secret1.yaml 10_secret1.yaml
vim 10_secret1.yaml
```

We need to adjust the Namespace for that Secret:

```yaml
# 10_secret1.yaml
apiVersion: v1
data:
  halt: IyEgL2Jpbi9zaAo...
kind: Secret
metadata:
  creationTimestamp: null
  name: secret1
  namespace: secret # change
```

```bash
k -f 10_secret1.yaml create
```

Next we create the second Secret:

```bash
k -n secret create secret generic secret2 --from-literal=user=user1 --from-literal=pass=1234
```

Now we create the Pod template:

```bash
k -n secret run secret-pod --image=busybox:1.31.1 $do -- sh -c "sleep 5d" > 10.yaml
vim 10.yaml
```

Then make the necessary changes:

```yaml
# 10.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: secret-pod
  name: secret-pod
  namespace: secret # add
spec:
  containers:
    - args:
        - sh
        - -c
        - sleep 1d
      image: busybox:1.31.1
      name: secret-pod
      resources: {}
      env: # add
        - name: APP_USER # add
          valueFrom: # add
            secretKeyRef: # add
              name: secret2 # add
              key: user # add
        - name: APP_PASS # add
          valueFrom: # add
            secretKeyRef: # add
              name: secret2 # add
              key: pass # add
      volumeMounts: # add
        - name: secret1 # add
          mountPath: /tmp/secret1 # add
          readOnly: true # add
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes: # add
    - name: secret1 # add
      secret: # add
        secretName: secret1 # add
status: {}
```

It might not be necessary in current K8s versions to specify the `readOnly: true` because it's the default setting anyways.

And execute:

```bash
k -f 10.yaml create
```

Finally we check if all is correct:

```bash
k -n secret exec secret-pod -- env | grep APP
```

```text
APP_PASS=1234
APP_USER=user1
```

```bash
k -n secret exec secret-pod -- find /tmp/secret1
```

```text
/tmp/secret1
/tmp/secret1/..data
/tmp/secret1/halt
/tmp/secret1/..2019_12_08_12_15_39.463036797
/tmp/secret1/..2019_12_08_12_15_39.463036797/halt
```

```bash
k -n secret exec secret-pod -- cat /tmp/secret1/halt
```

```text
#! /bin/sh
### BEGIN INIT INFO
# Provides: halt
# Required-Start:
# Required-Stop:
# Default-Start:
# Default-Stop: 0
# Short-Description: Execute the halt command.
# Description:
...
```
