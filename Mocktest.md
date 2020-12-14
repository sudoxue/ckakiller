# Praparation

1. Use Chrome
2. Got to Cheat sheet
3. Auto Complete. alias 
4. export as="--dry-run=client -o yaml"
5. set vim
```
vim  ~/.vimrc
set tabset=4
set shiftwidth=4
set expandtab
```

# Question 1 | Contexts

You have access to multiple clusters from your main terminal through kubectl contexts. Write all those context names into /opt/course/1/contexts.

Next write a command to display the current context into /opt/course/1/context_default_kubectl.sh, the command should use kubectl.

Finally write a second command doing the same thing into /opt/course/1/context_default_no_kubectl.sh, but without the use of kubectl.

```
k conifg view -o jsonpath='{.contexts[*].name}'
k config view -o jsonpath='{.contexts[*].name}' | tr " " "\n"
k config view -o jsonpath='{.contexts[*].name}' | tr " " "\n" >/opt/course/1/contexts  
```
Additional questions 

Get me all the cluster server name: 
```
k config view -o jsonpath='{.clusters[*].cluster.server}'| tr " " "\n"
```

# Question 2 | Schedule Pod on Master Node
Use context: kubectl config use-context k8s-c1-H

 

Create a single Pod of image httpd:2.4.41-alpine in Namespace default. The Pod should be named pod1 and the container should be named pod1-container. This Pod should only be scheduled on a master node, do not add new labels any nodes.

Shortly write the reason on why Pods are by default not scheduled on master nodes into /opt/course/2/master_schedule_reason .

```
k get node # find master node
​
k describe node cluster1-master1 | grep Taint # get master node taints
​
k describe node cluster1-master1 | grep Labels -A 10 # get master node labels, this means give me the labels 10 lines from the very begining. 


```
You need to add nodeSelector and tolerations to the manifest

```
# 2.yaml
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
    name: pod1-container                  # change
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  tolerations:                            # add
  - effect: NoSchedule                    # add
    key: node-role.kubernetes.io/master   # add
  nodeSelector:                           # add
    node-role.kubernetes.io/master: ""    # add
status: {}
```
Finally the short reason why Pods are not scheduled on master nodes by default:

```￼
# /opt/course/2/master_schedule_reason
master nodes usually have a taint defined
```
#  Question 3 | Scale down StatefulSet
Use context: kubectl config use-context k8s-c1-H
There are two Pods named o3db-* in Namespace project-c13. C13 management asked you to scale the Pods down to one replica to save resources. Record the action.

### Answers:
If we check the pods, we see two replicas: 
```
➜ k -n project-c13 get pod | grep o3db
o3db-0                                  1/1     Running   0          52s
o3db-1                                  1/1     Running   0          42s
```
From their name it looks like these are managed by a StatefulSet. But if we're not sure we could also check for the most common resources which mange Pods:

```
k get deploy,sts,ds | Grep o3db

```

```
➜ k -n project-c13 scale sts o3db --replicas 1 --record
statefulset.apps/o3db scaled
​
➜ k -n project-c13 get sts o3db
NAME   READY   AGE
o3db   1/1     4m39s
```
The --record created an annotation:

```
➜ k -n project-c13 describe sts o3db
Name:               o3db
Namespace:          project-c13
CreationTimestamp:  Sun, 20 Sep 2020 14:47:57 +0000
Selector:           app=nginx
Labels:             <none>
Annotations:        kubernetes.io/change-cause: kubectl scale sts o3db --namespace=project-c13 --replicas=1 --record=true
Replicas:           1 desired | 1 total
```
# Question 4 | Pod Ready if Service is reachable
Use context: kubectl config use-context k8s-c1-H

Do the following in Namespace default. Create a single Pod named ready-if-service-ready of image nginx:1.16.1-alpine. Configure a StartupProbe which runs sleep 3 && true. Also configure a ReadinessProbe which does check if the url http://service-am-i-ready:80 is reachable, you can use wget -T2 -O- http://service-am-i-ready:80 for this. Start the Pod and confirm it isn't ready because of the ReadinessProbe.

Create a second Pod named am-i-ready of image nginx:1.16.1-alpine with label id: cross-server-ready. The already existing Service service-am-i-ready should now have that second Pod as endpoint.

Now the first Pod should be in ready state, confirm that.

### Answers:
It's a bit of an anti-pattern for one Pod to check another Pod for being ready using probes, hence the normally available readinessProbe.httpGet doesn't work for absolute remote urls. Still the workaround requested in this task should show how probes and Pod<->Service communication works.

First we create the first Pod:

clucluster1-master1ster1-master1# 4_pod1.yaml
```
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
    startupProbe:                               # add from here
      exec:
        command:
        - sh
        - -c
        - 'sleep 3 && true'
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
Create pod And confirm its in a non-ready state:

```
➜ k describe pod ready-if-service-ready
 ...
  Warning  Unhealthy  18s   kubelet, cluster1-worker1  Readiness probe failed: Connecting to service-am-i-ready:80 (10.109.194.234:80)
wget: download timed out
create the second pod

k run am-i-ready --image=nginx:1.16.1-alpine --labels="id=cross-server-ready"
```

# Question 5 | kubectl sorting

Use context: kubectl config use-context k8s-c1-H

There are various Pods in all namespaces. Write a command into /opt/course/5/find_pods.sh which lists all Pods sorted by their AGE.

Write a second command into /opt/course/5/find_pods_uid.sh which lists all Pods sorted by field metadata.uid. Use kubectl sorting for both commands.

### Answer

 A good resources here (and for many other things) is the kubectl-cheat-sheet. You can reach it fast when searching for "cheat sheet" in the Kubernetes docs.
 ```
 # /opt/course/5/find_pods.sh
kubectl get pod -A --sort-by=.status.startTime
```

For the second command:

```
# /opt/course/5/find_pods_uid.sh
kubectl get pod -A --sort-by=.metadata.uid
```
and to execute:
```
➜ sh /opt/course/5/find_pods_uid.sh
NAMESPACE         NAME                                      ...          AGE
kube-system       coredns-5644d7b6d9-vwm7g                  ...          68m
project-c13       c13-3cc-runner-heavy-5486d76dd4-ddvlt     ...          63m
project-hamster   web-hamster-shop-849966f479-278vp         ...          63m
project-c13       c13-3cc-web-646b6c8756-qsg4b              ...          63m
```
# Question 6 | Storage, PV, PVC, Pod volume

Use context: kubectl config use-context k8s-c1-H

Create a new PersistentVolume named safari-pv. It should have a capacity of 2Gi, accessMode ReadWriteOnce, hostPath /Volumes/Data and no storageClassName defined.

Next create a new PersistentVolumeClaim in Namespace project-tiger named safari-pvc . It should request 2Gi storage, accessMode ReadWriteOnce and should not define a storageClassName. The PVC should bound to the PV correctly.

Finally create a new Deployment safari in Namespace project-tiger which mounts that volume at /tmp/safari-data. The Pods of that Deployment should be of image httpd:2.4.41-alpine.

### Answers:
ignore

# Question 7 | Node and Pod Resource Usage

Use context: kubectl config use-context k8s-c1-H

 

The metrics-server hasn't been installed yet in the cluster, but it's something that should be done soon. Your college would already like to know the kubectl commands to:

show node resource usage
show Pod and their containers resource usage
Please write the commands into /opt/course/7/node.sh and /opt/course/7/pod.sh.

 # Question 8 | Get Master Information

 Ssh into the master node with ssh root@cluster1-master1. Check how the master components kubelet, kube-apiserver, kube-scheduler, kube-controller-manager and etcd are started/installed on the master node. Also find out the name of the DNS application and how its started/installed on the master node.

Write your findings into file /opt/course/8/master-components.txt. The file could look like:

```
# /opt/course/8/master-components.txt
kubelet: not installed
kube-apiserver: systemd
kube-scheduler: not installed
kube-controller-manager: ...
etcd: ...
dns: super-dns, systemd

```
### Answers:

We could start by finding processes of the requested components, especially the kubelet at first

```
➜ ssh root@cluster1-master1
​
root@cluster1-master1:~# ps aux | grep kubelet # shows kubelet process

```
We can see which components are controlled via systemd looking at /etc/systemd/system directory:

```
➜ root@cluster1-master1:~# find /etc/systemd/system/ | grep kube
/etc/systemd/system/kubelet.service.d
/etc/systemd/system/kubelet.service.d/10-kubeadm.conf
/etc/systemd/system/multi-user.target.wants/kubelet.service
​
➜ root@cluster1-master1:~# find /etc/systemd/system/ | grep etcd
```
This shows kubelet is controlled via systemd, but no other service named kube nor etcd. It seems that this cluster has been setup using kubeadm, so we check in the default manifests directory:
```
➜ root@cluster1-master1:~# find /etc/kubernetes/manifests/
/etc/kubernetes/manifests/
/etc/kubernetes/manifests/kube-controller-manager.yaml
/etc/kubernetes/manifests/etcd.yaml
/etc/kubernetes/manifests/kube-scheduler-special.yaml
/etc/kubernetes/manifests/kube-apiserver.yaml
/etc/kubernetes/manifests/kube-scheduler.yaml
```

(The kubelet could also have a different manifests directory specified via parameter --pod-manifest-path in it's systemd startup config)

This means the main 4 master services are setup as static Pods. There also seems to be a second scheduler kube-scheduler-special existing.

Actually, let's check all Pods running on in the kube-system Namespace on the master node:

```
➜ root@cluster1-master1:~# kubectl -n kube-system get pod -o wide | grep master1
coredns-5644d7b6d9-c4f68                   1/1     Running            ...   cluster1-master1
coredns-5644d7b6d9-t84sc                   1/1     Running            ...   cluster1-master1
etcd-cluster1-master1                      1/1     Running            ...   cluster1-master1
kube-apiserver-cluster1-master1            1/1     Running            ...   cluster1-master1
kube-controller-manager-cluster1-master1   1/1     Running            ...   cluster1-master1
kube-proxy-q955p                           1/1     Running            ...   cluster1-master1
kube-scheduler-cluster1-master1            1/1     Running            ...   cluster1-master1
kube-scheduler-special-cluster1-master1    0/1     CrashLoopBackOff   ...   cluster1-master1
weave-net-mwj47                            2/2     Running            ...   cluster1-master1
```
There we see the 5 static pods, with -cluster1-master1 as suffix.

We also see that the dns application seems to be coredns, but how is it controlled?

```
➜ root@cluster1-master1$ kubectl -n kube-system get ds
NAME         DESIRED   CURRENT   ...   NODE SELECTOR            AGE
kube-proxy   3         3         ...   kubernetes.io/os=linux   155m
weave-net    3         3         ...   <none>                   155m
​
➜ root@cluster1-master1$ kubectl -n kube-system get deploy
NAME      READY   UP-TO-DATE   AVAILABLE   AGE
coredns   2/2     2            2           155m
```
Seems like coredns is controlled via a Deployment. We combine our findings in the requested file:
```
# /opt/course/8/master-components.txt
kubelet: systemd
kube-apiserver: static pod
kube-scheduler: static pod
kube-scheduler-special: static pod (status CrashLoopBackOff)
kube-controller-manager: static pod
etcd: static pod
dns: coredns, deployment with toleration and nodeSelector to only run on master
```
You should be comfortable investigating a running cluster, know different methods on how a cluster and its services can be setup and be able to troubleshoot and find error sources.

# Question 9 | Kill Scheduler, Manual Scheduling


Use context: kubectl config use-context k8s-c2-AC

 

Ssh into the master node with ssh root@cluster2-master1. Temporarily stop the kube-scheduler, this means in a way that you can start it again afterwards.

Create a single Pod named manual-schedule of image httpd:2.4-alpine, confirm its started but not scheduled on any node.

Now you're the scheduler and have all its power, manually schedule that Pod on node cluster2-master1. Make sure it's running.

Start the kube-scheduler again and confirm its running correctly by creating a second Pod named manual-schedule2 of image httpd:2.4-alpine and check if it's running on cluster2-worker1.


### Answer 

Stop the Scheduler 

First, we find the master node:

```
➜ k get node
NAME               STATUS   ROLES    AGE   VERSION
cluster2-master1   Ready    master   26h   v1.19.1
cluster2-worker1   Ready    <none>   26h   v1.19.1

```

Then we connect and check if the scheduler is running:

```
➜ ssh root@cluster2-master1
​
➜ root@cluster2-master1:~# kubectl -n kube-system get pod | grep schedule
kube-scheduler-cluster2-master1            1/1     Running   0          6s

```

Kill the Scheduler (temporarily):
```
➜ root@cluster2-master1:~# cd /etc/kubernetes/manifests/
​
➜ root@cluster2-master1:~# mv kube-scheduler.yaml ..

```

And it should be stopped:

```
➜ root@cluster2-master1:~# kubectl -n kube-system get pod | grep schedule
​
➜ root@cluster2-master1:~# 
```
### Create a Pod
Now we create the Pod:

```
k run manual-schedule --image=httpd:2.4-alpine

```

And confirm it has no node assigned

```
➜ k get pod manual-schedule -o wide
NAME              READY   STATUS    ...   NODE     NOMINATED NODE
manual-schedule   0/1     Pending   ...   <none>   <none>       
```
### Manually schedule the Pod
Let's play the scheduler now:

```
k get pod manual-schedule -o yaml > 9.yaml
```

```
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
  nodeName: cluster2-master1        # add the master node name
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
```
The only thing a scheduler does, is that it sets the nodeName for a Pod declaration. How it finds the correct node to schedule on, that's a very much complicated matter and takes many variables into account.

As we cannot kubectl apply or kubectl edit , in this case we need to delete and create or replace:

```
k -f 9.yaml replace --force
```
```
➜ k get pod manual-schedule -o wide
NAME              READY   STATUS    ...   NODE            
manual-schedule   1/1     Running   ...   cluster2-master1

```
It looks like our Pod is running on the master now as requested, although no tolerations were specified. Only the scheduler takes tains/tolerations/affinity into account when finding the correct node name. That's why its still possible to assign Pods manually directly to a master node and skip the scheduler.

Start the scheduler again

```
➜ ssh root@cluster2-master1
​
➜ root@cluster2-master1:~# cd /etc/kubernetes/manifests/
​
➜ root@cluster2-master1:~# mv ../kube-scheduler.yaml .
```

check its running: 
```
➜ root@cluster2-master1:~# kubectl -n kube-system get pod | grep schedule
kube-scheduler-cluster2-master1            1/1     Running   0          16s
```

Schedule a second test Pod: 

```
k run manual-schedule2 --image=httpd:2.4-alpine
```
```
➜ k get pod -o wide | grep schedule
manual-schedule    1/1     Running   ...   cluster2-master1
manual-schedule2   1/1     Running   ...   cluster2-worker1
```
back to normal

# Question 10 | RBAC ServiceAccount Role RoleBinding

Use context: kubectl config use-context k8s-c1-H


Create a new ServiceAccount processor in Namespace project-hamster. Create a Role and RoleBinding, both named processor as well. These should allow the new SA to only create Secrets and ConfigMaps in that Namespace.

### Answer:

Let's talk about a liottle about RBAC resources
Because of this there a 4 different RBAC combinations and 3 valid ones:

Role + RoleBinding (available in single Namespace, applied in single Namespace)

ClusterRole + ClusterRoleBinding (available cluster-wide, applied cluster-wide)

ClusterRole + RoleBinding (available cluster-wide, applied in single Namespace)

Role + ClusterRoleBinding (NOT POSSIBLE: available in single Namespace, applied cluster-wide)

 To the solution
We first create the ServiceAccount:
```
➜ k -n project-hamster create sa processor
serviceaccount/processor created
```


Then for the Role:
```
k -n project-hamster create role processor \
  --verb=create \
  --resource=secret \
  --resource=configmap
```
Which will create a Role like: 
```
# kubectl -n project-hamster create role accessor --verb=create --resource=secret --resource=configmap
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
No we bing the Role to the ServiceAccount

```
k -n project-hamster create rolebinding processor \
  --role processor \
  --serviceaccount project-hamster:processor
```
This will create a RoleBinding like:
```
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
To test our RBAC setup we can use kubectl suth can-i
```
k auth can-i -h example
```
like this 
```
➜ k -n project-hamster auth can-i create secret \
  --as system:serviceaccount:project-hamster:processor
yes
​
➜ k -n project-hamster auth can-i create configmap \
  --as system:serviceaccount:project-hamster:processor
yes
​
➜ k -n project-hamster auth can-i create pod \
  --as system:serviceaccount:project-hamster:processor
no
​
➜ k -n project-hamster auth can-i delete secret \
  --as system:serviceaccount:project-hamster:processor
no
​
➜ k -n project-hamster auth can-i get configmap \
  --as system:serviceaccount:project-hamster:processor
no
```
Done

# Question 11 | DaemonSet on all Nodes

Use context: kubectl config use-context k8s-c1-H

Use Namespace project-tiger for the following. Create a DaemonSet named ds-important with image httpd:2.4-alpine and labels id=ds-important and uuid=18426a0b-5f59-4e10-923f-c0e078e82462. The Pods it creates should request 100 millicore cpu and 10 megabytes memory. The Pods of that DaemonSet should run on all nodes.

### Answers:

As of now we aren't able to create a DaemonSet directly using kubectl, so we create a Deployment and just change it up:

```
k -n project-tiger create deployment --image=httpd:2.4-alpine ds-important $do > 11.yaml
​
vim 11.yaml

```
(Sure you could also search for a DaemonSet example yaml in the Kubernetes docs and alter it.)
```
Then we adjust the yaml to:

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
            cpu: 100m                               # add
            memory: 10Mi                            # add
      tolerations:                                  # add
      - effect: NoSchedule                          # add
        key: node-role.kubernetes.io/master         # add
#status: {}                                         # remove
```
It was requested that the DaemonSet runs on all nodes, so we need to specify the toleration for this.

Let's confirm:
```
k -f 11.yaml create
```

```
➜ k -n project-tiger get ds
NAME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
ds-important   3         3         3       3            3           <none>          8s
￼
```
```➜ k -n project-tiger get pod -l id=ds-important -o wide
NAME                      READY   STATUS          NODE
ds-important-6pvgm        1/1     Running   ...   cluster1-worker1
ds-important-lh5ts        1/1     Running   ...   cluster1-master1
ds-important-qhjcq        1/1     Running   ...   cluster1-worker2
```


# Question 12 | Deployment on all Nodes

Use Namespace project-tiger for the following. Create a Deployment named deploy-important with label id=very-important (the pods should also have this label) and 3 replicas. It should contain two containers, the first named container1 with image nginx:1.17.6-alpine and the second one named container2 with image kubernetes/pause.

There should be only ever one Pod of that Deployment running on one worker node. We have two worker nodes: cluster1-worker1 and cluster1-worker2. Because the Deployment has three replicas the result should be that on both nodes one Pod is running. The third Pod won't be scheduled, unless a new worker node will be added.

In a way we kind of simulate the behaviour of a DaemonSet here, but using a Deployment and a fixed number of replicas.

### Answer:

Check this example:
Node afinity is like a node selector
```
apiVersion: v1
kind: Pod
metadata:
  name: with-node-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - e2e-az1
            - e2e-az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
  containers:
  - name: with-node-affinity
    image: k8s.gcr.io/pause:2.0
```

Node affinity is conceptually similar to nodeSelector but more flexible and more powerful and dynamic. This node affinity rule says the pod can only be placed on a node with a label whose key is kubernetes.io/e2e-az-name and whose value is either e2e-az1 or e2e-az2. In addition, among nodes that meet that criteria, nodes with a label whose key is another-node-label-key and whose value is another-node-label-value should be preferred.


The idea here is that we create a "Inter-pod anti-affinity" which allows us to say a Pod should only be scheduled on a node where another Pod of a specific label (here the same label) is not already running.

Let's begin by creating the Deployment template:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    id: very-important                  # change
  name: deploy-important
  namespace: project-tiger              # important
spec:
  replicas: 3                           # change
  selector:
    matchLabels:
      id: very-important                # change
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        id: very-important              # change
    spec:
      containers:
      - image: nginx:1.17.6-alpine
        name: container1                # change
        resources: {}
      - image: kubernetes/pause         # add
        name: container2                # add
      affinity:                                             # add
        podAntiAffinity:                                    # add
          requiredDuringSchedulingIgnoredDuringExecution:   # add
          - labelSelector:                                  # add
              matchExpressions:                             # add
              - key: id                                     # add
                operator: In                                # add
                values:                                     # add
                - very-important                            # add
            topologyKey: kubernetes.io/hostname             # add
status: {}
```

Specify a topologyKey, which is a pre-populated Kubernetes label, you can find this by describing a node.

Let's run it:

```
k -f 12.yaml create
```
Then we check the Deployment status where it shows 2/3 ready count:

```
￼➜ k -n project-tiger get deploy -l id=very-important
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
deploy-important   2/3     3            2           2m35s
```
And running the following we see one Pod on each worker node and one not scheduled.
```
➜ k -n project-tiger get pod -o wide -l id=very-important
NAME                                READY   STATUS    ...   NODE             
deploy-important-58db9db6fc-9ljpw   2/2     Running   ...   cluster1-worker1
deploy-important-58db9db6fc-lnxdb   0/2     Pending   ...   <none>          
deploy-important-58db9db6fc-p2rz8   2/2     Running   ...   cluster1-worker2
```
￼If we kubectl describe the Pod deploy-important-58db9db6fc-lnxdb it will show us the reason for not scheduling is our implemented pod affinity/anti-affinity ruling:

￼
# Question 13 | Multi Containers and Pod shared Volume

Use context: kubectl config use-context k8s-c1-H

 

Create a Pod named multi-container-playground in Namespace default with three containers, named c1, c2 and c3. There should be a volume attached to that Pod and mounted into every container, but the volume shouldn't be persisted or shared with other Pods.

Container c1 should be of image nginx:1.17.6-alpine and have the name of the node where its Pod is running on value available as environment variable MY_NODE_NAME.

Container c2 should be of image busybox:1.31.1 and write the output of the date command every second in the shared volume into file date.log. You can use while true; do date >> /your/vol/path/date.log; sleep 1; done for this.

Container c3 should be of image busybox:1.31.1 and constantly write the content of file date.log from the shared volume to stdout. You can use tail -f /your/vol/path/date.log for this.

Check the logs of container c3 to confirm correct setup.

###
 
First we create the Pod template:
```
k run multi-container-playground --image=nginx:1.17.6-alpine $do > 13.yaml
​
vim 13.yaml
```
And add the other containers and the commands they should execute:

￼
```
# 13.yaml
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
    name: c1                                                                      # change
    resources: {}
    env:                                                                          # add
    - name: MY_NODE_NAME                                                          # add
      valueFrom:                                                                  # add
        fieldRef:                                                                 # add
          fieldPath: spec.nodeName                                                # add
    volumeMounts:                                                                 # add
    - name: vol                                                                   # add
      mountPath: /vol                                                             # add
  - image: busybox:1.31.1                                                         # add
    name: c2                                                                      # add
    command: ["sh", "-c", "while true; do date >> /vol/date.log; sleep 1; done"]  # add
    volumeMounts:                                                                 # add
    - name: vol                                                                   # add
      mountPath: /vol                                                             # add
  - image: busybox:1.31.1                                                         # add
    name: c3                                                                      # add
    command: ["sh", "-c", "tail -f /vol/date.log"]                                # add
    volumeMounts:                                                                 # add
    - name: vol                                                                   # add
      mountPath: /vol                                                             # add
  dnsPolicy: ClusterFirst
  restartPolicy: Always
  volumes:                                                                        # add
    - name: vol                                                                   # add
      emptyDir: {}                                                                # add
status: {}
￼

```
```
k -f 13.yaml create
```
```
➜ k get pod multi-container-playground
NAME                         READY   STATUS    RESTARTS   AGE
multi-container-playground   3/3     Running   0          95s
```
Good, then we check if container c1 has the requested node name as env variable:

```
➜ k exec multi-container-playground -c c1 -- env | grep MY
MY_NODE_NAME=cluster1-worker2
```
and finally we check the logging

```
➜ k logs multi-container-playground -c c3
Sat Dec  7 16:05:10 UTC 2077
Sat Dec  7 16:05:11 UTC 2077
Sat Dec  7 16:05:12 UTC 2077
Sat Dec  7 16:05:13 UTC 2077
Sat Dec  7 16:05:14 UTC 2077
Sat Dec  7 16:05:15 UTC 2077
Sat Dec  7 16:05:16 UTC 2077
```

# Question 14 | Find out Cluster Information
Use context: kubectl config use-context k8s-c1-H

 

You're ask to find out following information about the cluster k8s-c1-H:

How many master and worker nodes are available?
What are the Pod CIDR of master1, worker1 and worker2?
What is the Service CIDR?
Which Networking (or CNI Plugin) is configured and where is its config file?
Which suffix will static pods have that run on cluster1-worker1?
Write your answers into the existing file /opt/course/14/cluster-info.txt.

 ### Answer:

 How many master and worker nodes are available?
￼
```
➜ k get node
NAME               STATUS   ROLES    AGE   VERSION
cluster1-master1   Ready    master   27h   v1.19.1
cluster1-worker1   Ready    <none>   27h   v1.19.1
cluster1-worker2   Ready    <none>   27h   v1.19.1
```

We see one master and two workers.

 

What are the Pod CIDR of master1, worker1 and worker2?
The fastest way might just be to describe all nodes and look manually for the PodCIDR entry:

```
k describe node
​
k describe node | less -p PodCIDR
```
Or we can use jsonpath for this, but better be fast than tidy:
```
➜ k get node -o jsonpath="{range .items[*]}{.metadata.name} {.spec.podCIDR}{'\n'}"
cluster1-master1 10.244.0.0/24
cluster1-worker1 10.244.1.0/24
cluster1-worker2 10.244.2.0/24
```

What is the Service CIDR?
```
➜ ssh root@cluster1-master1
​
➜ root@cluster1-master1:~# cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep range
    - --service-cluster-ip-range=10.96.0.0/12

```

Which Networking (or CNI Plugin) is configured and where is its config file?
￼
```
➜ root@cluster1-master1:~# find /etc/cni/net.d/
/etc/cni/net.d/
/etc/cni/net.d/10-weave.conflist
​
➜ root@cluster1-master1:~# cat /etc/cni/net.d/10-weave.conflist
{
    "cniVersion": "0.3.0",
    "name": "weave",
...
```

By default the kubelet looks into /etc/cni/net.d to discover the CNI plugins. This will be the same on every master and worker nodes.

 

Which suffix will static pods have that run on cluster1-worker1?
The suffix is the node hostname with a leading hyphen. It used to be -static in earlier Kubernetes versions.

 

The resulting /opt/course/14/cluster-info.txt could look like:

￼
```
# /opt/course/14/cluster-info.txt
1. How many master and worker nodes are available:
1 master, 2 worker
​
2. What are the Pod CIDR of master1, worker1 and worker2:
master1: 10.244.0.0/24
worker1: 10.244.1.0/24
worker2: 10.244.2.0/24
​
3. What is the Service CIDR:
10.96.0.0/12
​
4. Which Networking (or CNI Plugin) is configured and where is its config:
Weave, config located at /etc/cni/net.d/10-weave.conflist on every node
​
5. Which suffix will static pods have that run on cluster1-worker1:
-cluster1-worker1

```
# Question 15 | Cluster Event Logging

Use context: kubectl config use-context k8s-c2-AC

Write a command into /opt/course/15/cluster_events.sh which shows the latest events in the whole cluster, ordered by time.

Now kill the kube-proxy Pod running on node cluster2-worker1 and write the events this caused into /opt/course/15/pod_kill.log.

Finally kill the main docker container of the kube-proxy Pod on node cluster2-worker1 and write the events into /opt/course/15/container_kill.log.

Do you notice differences in the events both actions caused?

### Answer:

```
# /opt/course/15/cluster_events.sh
kubectl get events -A --sort-by=.metadata.creationTimestamp

```
Now we kill the kube-proxy Pod:
```
k -n kube-system get pod -o wide | grep proxy # find pod running on cluster2-worker1
​
k -n kube-system delete pod kube-proxy-z64cg
```

Now check the warnings:
```
sh /opt/course/15/cluster_events.sh


```
Write the warnings the killing caused into /opt/course/15/pod_kill.log:

￼
```
# /opt/course/15/pod_kill.log
kube-system   9s          Normal    Killing           pod/kube-proxy-jsv7t   ...
kube-system   3s          Normal    SuccessfulCreate  daemonset/kube-proxy   ...
kube-system   <unknown>   Normal    Scheduled         pod/kube-proxy-m52sx   ...
default       2s          Normal    Starting          node/cluster2-worker1  ...
kube-system   2s          Normal    Created           pod/kube-proxy-m52sx   ...
kube-system   2s          Normal    Pulled            pod/kube-proxy-m52sx   ...
kube-system   2s          Normal    Started           pod/kube-proxy-m52sx   ...

```

Finally we will try to provoke events by killing the docker container belonging to the main container of the kube-proxy Pod:

￼
```
➜ ssh root@cluster2-worker1
​
➜ root@cluster2-worker1:~# docker ps | grep kube-proxy
5d4958901f3a        9b65a0f78b09           "/usr/local/bin/kube…"  5 minutes ago  ...
f8c56804a9c7        k8s.gcr.io/pause:3.1   "/pause"                5 minutes ago  ...
​
➜ root@cluster2-worker1:~# docker container rm 5d4958901f3a --force
​
➜ root@cluster2-worker1:~# docker ps | grep kube-proxy
52095b7d8107        9b65a0f78b09           "/usr/local/bin/kube…"   5 seconds ago  ...
f8c56804a9c7        k8s.gcr.io/pause:3.1   "/pause"                 6 minutes ago  ...
```

We killed the main container (5d4958901f3a), but also noticed that a new container (52095b7d8107) was directly created. Thanks Kubernetes!

Now we see if this caused events again and we write those into the second file:

￼
```
sh /opt/course/15/cluster_events.sh
```

```
# /opt/course/15/container_kill.log
kube-system   13s         Normal    Created      pod/kube-proxy-m52sx    ...
kube-system   13s         Normal    Pulled       pod/kube-proxy-m52sx    ...
kube-system   13s         Normal    Started      pod/kube-proxy-m52sx    ...
default       13s         Normal    Starting     node/cluster2-worker1   ...
```


Comparing the events we see that when we deleted the whole Pod there were more things to be done, hence more events. For example was the DaemonSet in the game to re-create the missing Pod. Where when we manually killed the main container of the Pod, the Pod would still exist but only its container needed to be re-created, hence less events.


# Question 18 | Fix Kubelet


Use context: kubectl config use-context k8s-c3-CCC

 

There seems to be an issue with the kubelet not running on cluster3-worker1. Fix it and confirm that cluster3 has node cluster3-worker1 available in Ready state afterwards. Schedule a Pod on cluster3-worker1.

Write the reason of the is issue into /opt/course/18/reason.txt.

Answer:

The procedure on tasks like these should be to check if the kubelet is running, if not start it, then check its logs and correct errors if there are some.

Always helpful to check if other clusters already have some of the components defined and running, so you can copy and use existing config files. Though in this case it might not need to be necessary.

Check node status:
```
➜ k get node
NAME               STATUS     ROLES    AGE   VERSION
cluster3-master1   Ready      master   27h   v1.19.1
cluster3-worker1   NotReady   <none>   26h   v1.19.1
```

First we check if the kubelet is running:

```
➜ ssh root@cluster3-worker1
​
➜ root@cluster3-worker1:~# ps aux | grep kubelet
root     29294  0.0  0.2  14856  1016 pts/0    S+   11:30   0:00 grep --color=auto kubelet
```





















# Extra Question 4 | Curl Manually Contact API
Use context: kubectl config use-context k8s-c1-H

 

There is an existing ServiceAccount secret-reader in Namespace project-hamster. Create a Pod of image curlimages/curl:7.65.3 named tmp-api-contact which uses this ServiceAccount. Make sure the container keeps running. Exec into the Pod and use curl to access the Kubernetes Api of that cluster manually, listing all available secrets. You can ignore insecure https connection. Write the command(s) for this into file /opt/course/e4/list-secrets.sh.

 ### Answers:

It's important to understand how the Kubernetes API works. For this it helps connecting to the api manually, for example using curl. You can find information fast by search in the Kubernetes docs for "curl api" for example.

First we create our Pod:

```
k run tmp-api-contact \
  --image=curlimages/curl:7.65.3 $do \
  --command > 6.yaml -- sh -c 'sleep 1d'
​
vim 6.yaml
```

Add the service account name and Namespace:
```
# 6.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    run: tmp-api-contact
  name: tmp-api-contact
  namespace: project-hamster          # add
spec:
  serviceAccountName: secret-reader   # add
  containers:
  - command:
    - sh
    - -c
    - sleep 1d
    image: curlimages/curl:7.65.3
    name: tmp-api-contact
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

```

Then run and exec into:
```
k -f 6.yaml create
​
k -n project-hamster exec tmp-api-contact -it -- sh
```

￼Once on the container we can try to connect to the api using curl, the api is usually available via the Service named kubernetes in Namespace default (You should know how dns resolution works across Namespaces.). Else we can find the endpoint IP via environment variables running env.

So now we can do:

```
curl https://kubernetes.default
curl -k https://kubernetes.default # ignore insecure as allowed in ticket description
curl -k https://kubernetes.default/api/v1/secrets # should show Forbidden 403
```

The last command shows 403 forbidden, this is because we are not passing any authorisation information with us. The Kubernetes Api Server thinks we are connecting as system:anonymous. We want to change this and connect using the Pods ServiceAccount named secret-reader.

We find the the token in the mounted folder at /var/run/secrets/kubernetes.io/serviceaccount, so we do:
```
➜ TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
➜ curl -k https://kubernetes.default/api/v1/secrets -H "Authorization: Bearer ${TOKEN}"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0{
  "kind": "SecretList",
  "apiVersion": "v1",
  "metadata": {
    "selfLink": "/api/v1/secrets",
    "resourceVersion": "10697"
  },
  "items": [
    {
      "metadata": {
        "name": "default-token-5zjbd",
        "namespace": "default",
        "selfLink": "/api/v1/namespaces/default/secrets/default-token-5zjbd",
        "uid": "315dbfd9-d235-482b-8bfc-c6167e7c1461",
        "resourceVersion": "342",
...
```

Now we're able to list all Secrets, registering as the ServiceAccount secret-reader under which our Pod is running.

To use encrypted https connection we can run:

```

CACERT=/var/run/secrets/kubernetes.io/serviceaccount/ca.crt
curl --cacert ${CACERT} https://kubernetes.default/api/v1/secrets -H "Authorization: Bearer ${TOKEN}"

```
For troubleshooting we could also check if the ServiceAccount is actually able to list Secrets using:
```
➜ k auth can-i get secret --as system:serviceaccount:project-hamster:secret-reader
yes
```

Finally write the commands into the requested location:
```
# /opt/course/e4/list-secrets.sh
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
curl -k https://kubernetes.default/api/v1/secrets -H "Authorization: Bearer ${TOKEN}"

￼```

```





# Preview Question 3

Use context: kubectl config use-context k8s-c1-H

There should be two schedulers on cluster1-master1, but only one is is reported to be running. Write all scheduler Pod names and their status into /opt/course/p3/schedulers.txt.

There is an existing Pod named special in Namespace default which should be scheduled by the second scheduler, but it's in a pending state.

Fix the second scheduler. Confirm it's working by checking that Pod special is scheduled on a node and running.

### Answer:

Write the scheduler info into file:
```
k -n kube-system get pod --show-labels # find labels
k -n kube-system get pod -l component=kube-scheduler > /opt/course/p3/schedulers.txt
```
The file could look like: 
```
# /opt/course/p3/schedulers.txt
NAME                                      READY   STATUS             RESTARTS   AGE
kube-scheduler-cluster1-master1           1/1     Running            0          26h
kube-scheduler-special-cluster1-master1   0/1     CrashLoopBackOff   20         26h
```

Check that Pod: 
```
➜ k get pod special -o wide
NAME      READY   STATUS    ...     NODE     NOMINATED NODE   READINESS GATES
special   0/1     Pending   ...     <none>   <none>           <none>
​
➜ k get pod special -o jsonpath="{.spec.schedulerName}{'\n'}"
kube-scheduler-special
```
Seems it has no node assigned because of the scheduler not working.

 

Fix the Scheduler
First we get the available schedulers:
```

➜ k -n kube-system get pod | grep scheduler                                                                 
kube-scheduler-cluster1-master1            1/1     Running            0          26h
kube-scheduler-special-cluster1-master1    0/1     CrashLoopBackOff   20         26h
```
It seems both are running as static Pods due to their name suffixes. First we check the logs:

```
➜ k -n kube-system logs kube-scheduler-special-cluster1-master1 | grep -i error
Error: unknown flag: --this-is-no-parameter
      --alsologtostderr                  log to standard error as well as files
      --logtostderr                      log to standard error instead of files (default true)
```
Well, it seems there is a unknown parameter set. So we connect into the master node, and check the manifests file:

```￼
➜ ssh root@cluster1-master1
​
➜ root@cluster1-master1:~# vim /etc/kubernetes/manifests/kube-scheduler-special.yaml
```
The kubelet could also have a different manifests directory specified via parameter --pod-manifest-path which you could find out via ps aux | grep kubelet and checking the kubelet systemd config. But in our case it's the default one.

Let's check the schedulers yaml:

```
# /etc/kubernetes/manifests/kube-scheduler-special.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: kube-scheduler-special
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
    - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
    - --bind-address=127.0.0.1
    - --port=7776
    - --secure-port=7777
    - --kubeconfig=/etc/kubernetes/kube-scheduler.conf
    - --leader-elect=false
    - --scheduler-name=kube-scheduler-special
    #- --this-is-no-parameter=what-the-hell                 # remove this obvious error
    image: k8s.gcr.io/kube-scheduler:v1.19.1
...
```

Changes on static Pods are recognised automatically by the kubelet, so we wait shortly and check again (you might need to give it a few seconds):

```￼
➜ root@cluster1-master1:~# kubectl -n kube-system get pod | grep scheduler
kube-scheduler-cluster1-master1            1/1     Running   0          26h
kube-scheduler-special-cluster1-master1    0/1     Error     0          9s

```

Also: we can get a Running state shortly, but it can turn into Error. Check a few times by repeating the command. So we still have an error, let's check the logs again:

```
➜ root@cluster1-master1:~# kubectl -n kube-system logs kube-scheduler-special-cluster1-master1
...
stat /etc/kubernetes/kube-scheduler.conf: no such file or directory

```
Well, it seems there is a file missing or a wrong path specified for that scheduler. So we check the manifests file again:

```￼
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  name: kube-scheduler-special
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
    - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
    - --bind-address=127.0.0.1
    - --port=7776
    - --secure-port=7777
    #- --kubeconfig=/etc/kubernetes/kube-scheduler.conf          # wrong path
    - --kubeconfig=/etc/kubernetes/scheduler.conf                # correct path
    - --leader-elect=false
    - --scheduler-name=kube-scheduler-special         
    #- --this-is-no-parameter=what-the-hell
    image: k8s.gcr.io/kube-scheduler:v1.19.1
...
```
Save and check the logs again:

```
➜ root@cluster1-master1:~# kubectl -n kube-system logs kube-scheduler-special-cluster1-master1
...
I0430 21:09:56.783926       1 configmap_cafile_content.go:205] Starting client-ca::kube-system::extension-apiserver-authentication::client-ca-file
I0430 21:09:56.783963       1 shared_informer.go:197] Waiting for caches to sync for client-ca::kube-system::extension-apiserver-authentication::client-ca-file
I0430 21:09:56.787005       1 secure_serving.go:178] Serving securely on 127.0.0.1:7777
I0430 21:09:56.787315       1 tlsconfig.go:219] Starting DynamicServingCertificateController
I0430 21:09:56.890232       1 shared_informer.go:204] Caches are synced for client-ca::kube-system::extension-apiserver-authentication::client-ca-file 
I0430 21:09:56.890658       1 shared_informer.go:204] Caches are synced for client-ca::kube-system::extension-apiserver-authentication::requestheader-client-ca-file 
```


Looking better, and the status:
```
➜ root@cluster1-master1:~# kubectl -n kube-system get pod | grep scheduler
kube-scheduler-cluster1-master1            1/1     Running   0          26h
kube-scheduler-special-cluster1-master1    1/1     Running   0          32s
```

Check the Pod again
Finally, is the Pod running and scheduled on a node?

```➜ k get pod special -o wide
NAME      READY   STATUS    RESTARTS   ...  NODE               NOMINATED NODE
special   1/1     Running   0          ...  cluster1-worker2   <none>        


```

Yes, we did it!

If you have to troubleshoot Kubernetes services in the CKA exam you should first check the logs. Then check its config and parameters for obvious misconfigurations. A good starting point is checking if all paths (to config files or certificates) are correct.

 
