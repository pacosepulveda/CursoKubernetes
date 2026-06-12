# Exam Questions 5

## Question 21 | RBAC ServiceAccount Role RoleBinding

Task weight: 6%

Use context: kubectl config use-context k8s-c1-H

Create a new ServiceAccount processor in Namespace project-hamster. Create a Role and RoleBinding, both named processor as well. These should allow the new SA to only create Secrets and ConfigMaps in that Namespace.

**Answer:**

Let's talk a little about RBAC resources

A ClusterRole|Role defines a set of permissions and where it is available, in the whole cluster or just a single Namespace.

A ClusterRoleBinding|RoleBinding connects a set of permissions with an account and defines where it is applied, in the whole cluster or just a single Namespace.

Because of this there are 4 different RBAC combinations and 3 valid ones:

Role + RoleBinding (available in single Namespace, applied in single Namespace)

ClusterRole + ClusterRoleBinding (available cluster-wide, applied cluster-wide)

ClusterRole + RoleBinding (available cluster-wide, applied in single Namespace)

Role + ClusterRoleBinding (NOT POSSIBLE: available in single Namespace, applied cluster-wide)

To the solution

We first create the ServiceAccount:

```bash
➜ k -n project-hamster create sa processor
serviceaccount/processor created
```

Then for the Role:

```bash
k -n project-hamster create role -h # examples
```

So we execute:

```bash
k -n project-hamster create role processor \
  --verb=create \
  --resource=secret \
  --resource=configmap
```

Which will create a Role like:

```yaml
# kubectl -n project-hamster create role processor --verb=create --resource=secret --resource=configmap
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: processor
  namespace: project-hamster
rules:
- apiGroups:
  - ""
  resources:
  - secrets
  - configmaps
  verbs:
  - create
```

Now we bind the Role to the ServiceAccount:

```bash
k -n project-hamster create rolebinding -h # examples
```

So we create it:

```bash
k -n project-hamster create rolebinding processor \
  --role processor \
  --serviceaccount project-hamster:processor
```

This will create a RoleBinding like:

```yaml
# kubectl -n project-hamster create rolebinding processor --role processor --serviceaccount project-hamster:processor
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: processor
  namespace: project-hamster
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: processor
subjects:
- kind: ServiceAccount
  name: processor
  namespace: project-hamster
```

To test our RBAC setup we can use kubectl auth can-i:

```bash
k auth can-i -h # examples
```

Like this:

```bash
➜ k -n project-hamster auth can-i create secret \
  --as system:serviceaccount:project-hamster:processor
yes
➜ k -n project-hamster auth can-i create configmap \
  --as system:serviceaccount:project-hamster:processor
yes
➜ k -n project-hamster auth can-i create pod \
  --as system:serviceaccount:project-hamster:processor
no
➜ k -n project-hamster auth can-i delete secret \
  --as system:serviceaccount:project-hamster:processor
no
➜ k -n project-hamster auth can-i get configmap \
  --as system:serviceaccount:project-hamster:processor
no
```

Done.

## Question 14 | Find out Cluster Information

Task weight: 2%

Use context: kubectl config use-context k8s-c1-H

You're ask to find out following information about the cluster k8s-c1-H:

How many controlplane nodes are available?

How many worker nodes are available?

What is the Service CIDR?

Which Networking (or CNI Plugin) is configured and where is its config file?

Which suffix will static pods have that run on cluster1-node1?

Write your answers into file /opt/course/14/cluster-info, structured like this:

```bash
# /opt/course/14/cluster-info
1: [ANSWER]
2: [ANSWER]
3: [ANSWER]
4: [ANSWER]
5: [ANSWER]
```

 Answer:

How many controlplane and worker nodes are available?

```bash
➜ k get node
NAME                    STATUS   ROLES          AGE   VERSION
cluster1-controlplane1  Ready    control-plane  27h   v1.27.1
cluster1-node1          Ready    <none>         27h   v1.27.1
cluster1-node2          Ready    <none>         27h   v1.27.1
```

We see one controlplane and two workers.

What is the Service CIDR?

```bash
➜ ssh cluster1-controlplane1
➜ root@cluster1-controlplane1:~# cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep range
    - --service-cluster-ip-range=10.96.0.0/12
```

Which Networking (or CNI Plugin) is configured and where is its config file?

```bash
➜ root@cluster1-controlplane1:~# find /etc/cni/net.d/
/etc/cni/net.d/
/etc/cni/net.d/10-weave.conflist
➜ root@cluster1-controlplane1:~# cat /etc/cni/net.d/10-weave.conflist
{
    "cniVersion": "0.3.0",
    "name": "weave",
...
```

By default the kubelet looks into /etc/cni/net.d to discover the CNI plugins. This will be the same on every controlplane and worker nodes.

Which suffix will static pods have that run on cluster1-node1?

The suffix is the node hostname with a leading hyphen. It used to be -static in earlier Kubernetes versions.

Result

The resulting /opt/course/14/cluster-info could look like:

```bash
# /opt/course/14/cluster-info
# How many controlplane nodes are available?
1: 1
# How many worker nodes are available?
2: 2
# What is the Service CIDR?
3: 10.96.0.0/12
# Which Networking (or CNI Plugin) is configured and where is its config file?
4: Weave, /etc/cni/net.d/10-weave.conflist
# Which suffix will static pods have that run on cluster1-node1?
5: -cluster1-node1
```

## Question 22 | Namespaces and Api Resources

Task weight: 2%

Use context: kubectl config use-context k8s-c1-H

Write the names of all namespaced Kubernetes resources (like Pod, Secret, ConfigMap...) into /opt/course/16/resources.txt.

Find the project-* Namespace with the highest number of Roles defined in it and write its name and amount of Roles into /opt/course/16/crowded-namespace.txt.

**Answer:**

Namespace and Namespaces Resources

Now we can get a list of all resources like:

```bash
k api-resources    # shows all
k api-resources -h # help always good
k api-resources --namespaced -o name > /opt/course/16/resources.txt
```

Which results in the file:

```bash
# /opt/course/16/resources.txt
bindings
configmaps
endpoints
events
limitranges
persistentvolumeclaims
pods
podtemplates
replicationcontrollers
resourcequotas
secrets
serviceaccounts
services
controllerrevisions.apps
daemonsets.apps
deployments.apps
replicasets.apps
statefulsets.apps
localsubjectaccessreviews.authorization.k8s.io
horizontalpodautoscalers.autoscaling
cronjobs.batch
jobs.batch
leases.coordination.k8s.io
events.events.k8s.io
ingresses.extensions
ingresses.networking.k8s.io
networkpolicies.networking.k8s.io
poddisruptionbudgets.policy
rolebindings.rbac.authorization.k8s.io
roles.rbac.authorization.k8s.io
```

Namespace with most Roles

```bash
➜ k -n project-c13 get role --no-headers | wc -l
No resources found in project-c13 namespace.
0
➜ k -n project-c14 get role --no-headers | wc -l
300
➜ k -n project-hamster get role --no-headers | wc -l
No resources found in project-hamster namespace.
```

0

```bash
➜ k -n project-snake get role --no-headers | wc -l
No resources found in project-snake namespace.
```

0

```bash
➜ k -n project-tiger get role --no-headers | wc -l
No resources found in project-tiger namespace.
```

0

Finally we write the name and amount into the file:

```bash
# /opt/course/16/crowded-namespace.txt
project-c14 with 300 resources
```

## Question 23 | Find Container of Pod and check info

Task weight: 3%

Use context: kubectl config use-context k8s-c1-H

In Namespace project-tiger create a Pod named tigers-reunite of image httpd:2.4.41-alpine with labels pod=container and container=pod. Find out on which node the Pod is scheduled. Ssh into that node and find the containerd container belonging to that Pod.

Using command crictl:

Write the ID of the container and the info.runtimeType into /opt/course/17/pod-container.txt

Write the logs of the container into /opt/course/17/pod-container.log

**Answer:**

First we create the Pod:

```bash
k -n project-tiger run tigers-reunite \
  --image=httpd:2.4.41-alpine \
  --labels "pod=container,container=pod"
```

Next we find out the node it's scheduled on:

```bash
k -n project-tiger get pod -o wide
# or fancy:
k -n project-tiger get pod tigers-reunite -o jsonpath="{.spec.nodeName}"
```

Then we ssh into that node and and check the container info:

```bash
➜ ssh cluster1-node2
➜ root@cluster1-node2:~# crictl ps | grep tigers-reunite
b01edbe6f89ed    54b0995a63052    5 seconds ago    Running        tigers-reunite ...
➜ root@cluster1-node2:~# crictl inspect b01edbe6f89ed | grep runtimeType
    "runtimeType": "io.containerd.runc.v2",
```

Then we fill the requested file (on the main terminal):

```bash
# /opt/course/17/pod-container.txt
b01edbe6f89ed io.containerd.runc.v2
```

Finally we write the container logs in the second file:

```bash
ssh cluster1-node2 'crictl logs b01edbe6f89ed' &> /opt/course/17/pod-container.log
```

The &> in above's command redirects both the standard output and standard error.

You could also simply run crictl logs on the node and copy the content manually, if it's not a lot. The file should look like:

```bash
# /opt/course/17/pod-container.log
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.44.0.37. Set the 'ServerName' directive globally to suppress this message
AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using 10.44.0.37. Set the 'ServerName' directive globally to suppress this message
[Mon Sep 13 13:32:18.555280 2021] [mpm_event:notice] [pid 1:tid 139929534545224] AH00489: Apache/2.4.41 (Unix) configured -- resuming normal operations
[Mon Sep 13 13:32:18.555610 2021] [core:notice] [pid 1:tid 139929534545224] AH00094: Command line: 'httpd -D FOREGROUND'
```

## Question 24 | Create Secret and mount into Pod

Task weight: 3%

NOTE: This task can only be solved if questions 18 or 20 have been successfully implemented and the k8s-c3-CCC cluster has a functioning worker node

Use context: kubectl config use-context k8s-c3-CCC

Do the following in a new Namespace secret. Create a Pod named secret-pod of image busybox:1.31.1 which should keep running for some time.

There is an existing Secret located at /opt/course/19/secret1.yaml, create it in the Namespace secret and mount it readonly into the Pod at /tmp/secret1.

Create a new Secret in Namespace secret called secret2 which should contain user=user1 and pass=1234. These entries should be available inside the Pod's container as environment variables APP_USER and APP_PASS.

Confirm everything is working.

**Answer:**

First we create the Namespace and the requested Secrets in it:

```bash
k create ns secret
cp /opt/course/19/secret1.yaml 19_secret1.yaml
vim 19_secret1.yaml
```

We need to adjust the Namespace for that Secret:

```yaml
# 19_secret1.yaml
apiVersion: v1
data:
  halt: IyEgL2Jpbi9zaAo...
kind: Secret
metadata:
  creationTimestamp: null
  name: secret1
  namespace: secret           # change
k -f 19_secret1.yaml create
```

Next we create the second Secret:

```bash
k -n secret create secret generic secret2 --from-literal=user=user1 --from-literal=pass=1234
```

Now we create the Pod template:

```bash
k -n secret run secret-pod --image=busybox:1.31.1 $do -- sh -c "sleep 5d" > 19.yaml
vim 19.yaml
```

Then make the necessary changes:

```yaml
# 19.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: secret-pod
  name: secret-pod
  namespace: secret                       # add
spec:
  containers:
  - args:
    - sh
    - -c
    - sleep 1d
    image: busybox:1.31.1
    name: secret-pod
    resources: {}
    env:                                  # add
    - name: APP_USER                      # add
      valueFrom:                          # add
        secretKeyRef:                     # add
          name: secret2                   # add
          key: user                       # add
    - name: APP_PASS                      # add
      valueFrom:                          # add
        secretKeyRef:                     # add
          name: secret2                   # add
          key: pass                       # add
    volumeMounts:                         # add
    - name: secret1                       # add
      mountPath: /tmp/secret1             # add
      readOnly: true                      # add
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes:                                # add
  - name: secret1                         # add
    secret:                               # add
      secretName: secret1                 # add
status: {}
```

It might not be necessary in current K8s versions to specify the readOnly: true because it's the default setting anyways.

And execute:

```bash
k -f 19.yaml create
```

Finally we check if all is correct:

```bash
➜ k -n secret exec secret-pod -- env | grep APP
APP_PASS=1234
APP_USER=user1
➜ k -n secret exec secret-pod -- find /tmp/secret1
/tmp/secret1
/tmp/secret1/..data
/tmp/secret1/halt
/tmp/secret1/..2019_12_08_12_15_39.463036797
/tmp/secret1/..2019_12_08_12_15_39.463036797/halt
➜ k -n secret exec secret-pod -- cat /tmp/secret1/halt
#! /bin/sh
### BEGIN INIT INFO
# Provides:          halt
# Required-Start:
# Required-Stop:
# Default-Start:
# Default-Stop:      0
# Short-Description: Execute the halt command.
# Description:
...
```

All is good.

## Question 25 | NetworkPolicy

Task weight: 9%

Use context: kubectl config use-context k8s-c1-H

There was a security incident where an intruder was able to access the whole cluster from a single hacked backend Pod.

To prevent this create a NetworkPolicy called np-backend in Namespace project-snake. It should allow the backend-* Pods only to:

connect to db1-* Pods on port 1111

connect to db2-* Pods on port 2222

Use the app label of Pods in your policy.

After implementation, connections from backend-* Pods to vault-* Pods on port 3333 should for example no longer work.

**Answer:**

First we look at the existing Pods and their labels:

```bash
➜ k -n project-snake get pod
NAME        READY   STATUS    RESTARTS   AGE
backend-0   1/1     Running   0          8s
db1-0       1/1     Running   0          8s
db2-0       1/1     Running   0          10s
vault-0     1/1     Running   0          10s
➜ k -n project-snake get pod -L app
NAME        READY   STATUS    RESTARTS   AGE     APP
backend-0   1/1     Running   0          3m15s   backend
db1-0       1/1     Running   0          3m15s   db1
db2-0       1/1     Running   0          3m17s   db2
vault-0     1/1     Running   0          3m17s   vault
```

We test the current connection situation and see nothing is restricted:

```bash
➜ k -n project-snake get pod -o wide
NAME        READY   STATUS    RESTARTS   AGE     IP          ...
backend-0   1/1     Running   0          4m14s   10.44.0.24  ...
db1-0       1/1     Running   0          4m14s   10.44.0.25  ...
db2-0       1/1     Running   0          4m16s   10.44.0.23  ...
vault-0     1/1     Running   0          4m16s   10.44.0.22  ...
➜ k -n project-snake exec backend-0 -- curl -s 10.44.0.25:1111
```

database one

```bash
➜ k -n project-snake exec backend-0 -- curl -s 10.44.0.23:2222
```

database two

```bash
➜ k -n project-snake exec backend-0 -- curl -s 10.44.0.22:3333
```

vault secret storage

Now we create the NP by copying and chaning an example from the k8s docs:

```yaml
vim 24_np.yaml
# 24_np.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: np-backend
  namespace: project-snake
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress                    # policy is only about Egress
  egress:
    -                           # first rule
      to:                           # first condition "to"
      - podSelector:
          matchLabels:
            app: db1
      ports:                        # second condition "port"
      - protocol: TCP
        port: 1111
    -                           # second rule
      to:                           # first condition "to"
      - podSelector:
          matchLabels:
            app: db2
      ports:                        # second condition "port"
      - protocol: TCP
        port: 2222
```

The NP above has two rules with two conditions each, it can be read as:

allow outgoing traffic if:

  (destination pod has label app=db1 AND port is 1111)

  OR

  (destination pod has label app=db2 AND port is 2222)

Wrong example

Now let's shortly look at a wrong example:

```yaml
# WRONG
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: np-backend
  namespace: project-snake
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress
  egress:
    -                           # first rule
      to:                           # first condition "to"
      - podSelector:                    # first "to" possibility
          matchLabels:
            app: db1
      - podSelector:                    # second "to" possibility
          matchLabels:
            app: db2
      ports:                        # second condition "ports"
      - protocol: TCP                   # first "ports" possibility
        port: 1111
      - protocol: TCP                   # second "ports" possibility
        port: 2222
```

The NP above has one rule with two conditions and two condition-entries each, it can be read as:

allow outgoing traffic if:

  (destination pod has label app=db1 OR destination pod has label app=db2)

  AND

  (destination port is 1111 OR destination port is 2222)

Using this NP it would still be possible for backend-* Pods to connect to db2-* Pods on port 1111 for example which should be forbidden.

Create NetworkPolicy

We create the correct NP:

```bash
k -f 24_np.yaml create
```

And test again:

```bash
➜ k -n project-snake exec backend-0 -- curl -s 10.44.0.25:1111
```

database one

```bash
➜ k -n project-snake exec backend-0 -- curl -s 10.44.0.23:2222
```

database two

```bash
➜ k -n project-snake exec backend-0 -- curl -s 10.44.0.22:3333
```

^C

Also helpful to use kubectl describe on the NP to see how k8s has interpreted the policy.

Great, looking more secure. Task done.
