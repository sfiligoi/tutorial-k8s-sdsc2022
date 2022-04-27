# Kubernetes for Science Tutorial

**Scheduling in Kubernetes\
Hands on session**

## Explore the system

Let's start with looking what's available in the  system.
Start by getting the list of all the nodes:

```
kubectl get nodes
```

Let's now dig a little deeper. For example, you can see which nodes have which GPU type:

```
kubectl get nodes -L nvidia.com/gpu.product
```

Now, pick one node, and see what other resources it has:

```
kubectl get nodes -o yaml <nodename>
```

If you picked a node with a GPU, look for the "nvidia.com/gpu.product" in the output. Did it match?

Now pick another label from the output, and use it instead of nvidia.com/gpu.product in the above all-node query. Did it return what you expected?

## Validating requirements

In the next section we will play with adding requirements to Pod yamls. But first let's make sure those resources are actually available.

As a simple example, let's pick a specific GPU type:

```
get nodes -l 'nvidia.com/gpu.product=NVIDIA-GeForce-GTX-1080-Ti'
```

Did you get any hits?

Here we look for nodes with 100Gbps NICs:

```
kubectl get node -l 'nautilus.io/network=100000'
```

Did you get any hits?

How about a negative selector? And let's see what do we get:

```
kubectl get node -l 'nvidia.com/gpu.product!=NVIDIA-GeForce-GTX-1080, nvidia.com/gpu.product!=NVIDIA-GeForce-GTX-1070' -L nvidia.com/gpu.product
```

## Requirements in pods

You have already used requirements in the pods, indirectly via the resource request. Here is the very first pod we made you launch:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-<username>
spec:
  containers:
  - name: mypod
    image: centos:centos8
    resources:
      limits:
        memory: 1.5Gi
        cpu: 1
      requests:
        memory: 500Mi
        cpu: 100m
    command: ["sh", "-c", "sleep infinity"]
```

But we set them to be really low, so it was virtually guaranteed that the Pod would start.

Now, let's add one more requirement. Let's ask for a GPU. We also change the container, so that we get the proper drivers in place.\
**Note:** While you can ask for a fraction of a CPU, you cannot ask for a fraction of a GPU in our current setup. You should also keep the same number for requirements and limits.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpupod-<username>
spec:
  containers:
  - name: mypod
    image: nvidia/cuda:11.4.1-runtime-centos8
    resources:
      limits:
        memory: 1.5Gi
        cpu: 1
        nvidia.com/gpu: 1
      requests:
        memory: 500Mi
        cpu: 100m
        nvidia.com/gpu: 1
    command: ["sh", "-c", "sleep infinity"]
```

Once the pod started, login using kubectl exec and check what kind of GPU you got:

```
nvidia-smi
```

Once you are done examinining the pod, you can delete it.

Let's now ask for a specific GPU type (remember to destroy the old pod):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpupod-<username>
spec:
  nodeSelector:
    nvidia.com/gpu.product: "NVIDIA-GeForce-GTX-1080-Ti"
  containers:
  - name: mypod
    image: nvidia/cuda:11.4.1-runtime-centos8
    resources:
      limits:
        memory: 1.5Gi
        cpu: 1
        nvidia.com/gpu: 1
      requests:
        memory: 500Mi
        cpu: 100m
        nvidia.com/gpu: 1
    command: ["sh", "-c", "sleep infinity"]
```

Did the pod start?

Log into the Pod and check if you indeed got the desired GPU type.

Once you are done examinining the pod, you can delete it.

## Preferences in pods

Sometimes you would prefer something, but can live with less. In this example, let's ask for the fastest GPU in out pool, but not as a hard requirement:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpupod-<username>
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: nvidia.com/gpu.product
            operator: In
            values:
            - "NVIDIA-TITAN-RTX"
            - "NVIDIA-RTX-A6000"
  containers:
  - name: mypod
    image: nvidia/cuda:11.4.1-runtime-centos8
    resources:
      limits:
        memory: 1.5Gi
        cpu: 1
        nvidia.com/gpu: 1
      requests:
        memory: 500Mi
        cpu: 100m
        nvidia.com/gpu: 1
    command: ["sh", "-c", "sleep infinity"]
```

Now check you Pod. How long did it take for it to start? What GPU type did you get?

Once you are done examinining the pod, you can delete it.

## Using tolerations 

There are several parts of the PRP Kubernetes pool that are off-limits to regular users. One example is a set of nodes only connected with standard (not jumbo-frame) MTU size.

Here is a Pod yaml that will try to run on those:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mtu1500-<username>
spec:
  nodeSelector:
    mtu: "1500"
  containers:
  - name: mypod
    image: centos:centos8
    resources:
      limits:
        memory: 1.5Gi
        cpu: 1
      requests:
        memory: 50Mi
        cpu: 10m
    command: ["sh", "-c", "sleep infinity"]
```

Go ahead, request a pod like that. Did the pod start?

It should not have. There are essentially no nodes that will match the above requirements. Check for yourself:

```
kubectl get events --sort-by=.metadata.creationTimestamp
```

Now, look up the list of nodes that were supposed to match:

```
kubectl get nodes -l 'mtu=1500'
```

Pick a node and look for details. Search for any taints. You should see something along the lines of:

```yaml
...
spec:
  taints:
  - effect: NoSchedule
    key: nautilus.io/testing
    value: "true"
...
```

You have been granted permission to run on those nodes, so let's now add the toleration that will allow you to run there (remember to remove the old Pod):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mtu1500-<username>
spec:
  nodeSelector:
    mtu: "1500"
  tolerations:
  - effect: NoSchedule
    key: nautilus.io/testing
    operator: Exists
  containers:
  - name: mypod
    image: centos:centos8
    resources:
      limits:
        memory: 1.5Gi
        cpu: 1
      requests:
        memory: 50Mi
        cpu: 10m
    command: ["sh", "-c", "sleep infinity"]
```

Try to submit this one.

Did the Pod start?

Inside the pod check that you're indeed on 1500 (1440 for tunneled interface) MTU:

```
ip a
```

You can delete the pod now.

## The end

We do not include hands-on exercises based on priorities, as they are very non-deterministic. 
Hopefully the slides provided clear enough instructions for you to understand the concept.

Please make sure you did not leave any pods behind.
