# Kubernetes for Science Tutorial

Basic Kubernetes\
Hands on session

## Setup

There are several basic step you must take to get access to the cluster.

1. Install the kubectl Kubernetes client.\
   Instructions at <https://kubernetes.io/docs/tasks/tools/install-kubectl/>\
   If you have homebrew installed on mac, use that. Otherwise try downloading the static binary (the curl way)
2. Get your configuration file for the PRP/Nautilus Kubernetes cluster from organizers, and put it as \~/.kube/config.

The config files for this tutorial will have the right namespace pre-set. In general you need to be aware of which namespace you are working in, and either set it with `kubectl config set-context nautilus --namespace=the_namespace` or specify in each `kubectl` command by adding `-n namespace`.

## Launch a simple pod

Let’s create a simple generic pod, and login into it.

You can copy-and-paste the lines below, but please do replace “username” with your own id.\
All the participants in this hands-on session share the same namespace, so you will get name collisions if you don’t.

```
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
        memory: 100Mi
        cpu: 100m
      requests:
        memory: 100Mi
        cpu: 100m
    command: ["sh", "-c", "sleep infinity"]
```

Reminder, indentation is important in YAML, just like in Python.

*If you don't want to create the file and are using mac or linux, you can create yaml's dynamically like this:*

```
kubectl create -f - << EOF
<contents you want to deploy>
EOF
```

Now let’s start the pod:

```
kubectl create -f pod1.yaml
```

See if you can find it:

```
kubectl get pods
```

Note: You may see the pods from the other participants, too.

If it is not yet in Running state, you can check what is going on with

```
kubectl get events --sort-by=.metadata.creationTimestamp
```

Then let’s log into it

```
kubectl exec -it pod-<username> -- /bin/bash
```

You are now inside the (container in the) pod!

Does it feel any different than a regular, dedicated node?

**Try to create some directories and some files with content.**

(Hello world will do, but feel free to be creative)

**❗️ Note that some containers don't have /bin/bash installed. In other examples if you see the "Not Found" error, you might try** `/bin/sh`. Some containers (not in this tutorial) don't have any shell installed, and you can't exec into those.

We will want to check the status of the networking.

But ifconfig is not available in the image we are using; so let’s install it

```
yum install net-tools
```

Now check the the networking:

```
ifconfig -a
```

Get out of the Pod (with either Control-D or exit).

You should see the same IP displayed with kubectl

```
kubectl get pod -o wide pod-<username>
```

We can now destroy the pod

```
kubectl delete -f pod1.yaml
```

Check that it is actually gone:

```
kubectl get pods
```

Now, let’s create it again:

```
kubectl create -f pod1.yaml
```

Does it have the same IP?

```
kubectl get pod -o wide pod-<username>
```

Log back into the pod:

```
kubectl exec -it pod-<username> -- /bin/bash
```

What does the network look like now?

What is the status of the files your created?

Finally, let’s delete explicitly the pod:

```
kubectl delete pod pod-<username>
```

## Let’s make it a deployment

You saw that when a pod was terminated, it was gone.

While above we did it by ourselves, the result would have been the same if a node died or was restarted.

In order to gain a higher availability, the use of Deployments is recommended. So, that’s what we will do next.

You can copy-and-paste the lines below, but please do replace “username” with your own id.

###### dep1.yaml:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dep-<username>
  labels:
    k8s-app: dep-<username>
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: dep-<username>
  template:
    metadata: 
      labels:
        k8s-app: dep-<username>
    spec:
      containers:
      - name: mypod
        image: centos:centos7
        resources:
           limits:
             memory: 1.5Gi
             cpu: 1
           requests:
             memory: 0.5Gi
             cpu: 0.1
        command: ["sh", "-c", "sleep infinity"]
```

Now let’s start the deployment:

```
kubectl create -f dep1.yaml
```

See if you can find it:

```
kubectl get deployments
```

Note: You may see the deployments from the other participants, too.

The Deployment is just a conceptual service, though.

See if you can find the associated pod:

```
kubectl get pods
```

Once you have found its name, let’s log into it

```
kubectl get pod -o wide dep-<username>-<hash>
kubectl exec -it dep-<username>-<hash> -- /bin/bash
```

You are now inside the (container in the) pod!

Create directories and files as before.

Try various commands as before.

Let’s now delete the pod!

```
kubectl delete pod dep-<username>-<hash>
```

Is it really gone?

```
kubectl get pods 
```

What happened to the deployment?

```
kubectl get deployments
```

Get into the new pod

```
kubectl get pod -o wide dep-<username>-<hash>
kubectl exec -it dep-<username>-<hash> -- /bin/bash
```

Was anything preserved?

Let’s now delete the deployment:

```
kubectl delete -f dep1.yaml
```

Verify everything is gone:

```
kubectl get deployments
kubectl get pods
```

## Let’s make it short lived

Most scientific processes do not live forever.
So, let's try something with a short life span.

You can copy-and-paste the lines below, but please do replace “username” with your own id;\
As mentioned before, all the participants in this hands-on session share the same namespace, so you will get name collisions if you don’t.

###### dep2.yaml:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dep-<username>
  labels:
    k8s-app: dep-<username>
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: dep-<username>
  template:
    metadata: 
      labels:
        k8s-app: dep-<username>
    spec:
      containers:
      - name: mypod
        image: centos:centos7
        resources:
           limits:
             memory: 1.5Gi
             cpu: 1
           requests:
             memory: 0.5Gi
             cpu: 0.1
        command: ["sh", "-c", "sleep 30; env; echo Done"]
```

Now let’s create this new deployment:

```
kubectl create -f dep2.yaml
```

As before, check for the associated pod:

```
kubectl get pods 
```

Wait a minute and try again (a few times).

What happened?
(Hint: check the status and restart column)

Once you are satisfied, remove the deployment:
```
kubectl delete -f dep2.yaml
```

## Using the job controller

Most science applications need to run only once, as long as that run was successsful.
The job controller aims at fulfilling that need.

You can copy-and-paste the lines below, but please do replace “username” with your own id.

###### job1.yaml:

```
apiVersion: batch/v1
kind: Job
metadata:
  name: job1-<username>
spec:
  completions: 1
  ttlSecondsAfterFinished: 1800
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: mypod
        image: centos:centos7
        resources:
           limits:
             memory: 1.5Gi
             cpu: 1
           requests:
             memory: 0.5Gi
             cpu: 0.1
        command: ["sh", "-c", "sleep 30; env; echo Done"]
```


Now let’s create the job:

```
kubectl create -f job1.yaml
```

Check for the job and the associated pod:

```
kubectl get jobs
kubectl get pods 
```

Wait a minute and try again (a few times).

What happened?
(Hint: check the status and restart column)

The pod should have ended... now what?

You may have noticed that we are printing out to standard output. You can retrieve that with:
```
kubectl logs job1-<username>-<hash>
```

You could now delete the job. But you don't have to.
It will automatically remove itself after half an hour (1800 seconds).

## Parameter sweep

Running a single application iteration is interesting, but science users often need many executions to get their science done.

So, let's do a simple parameter sweep. I.e. let's execute 10 pods with the same code but different inputs.

As usual, you can copy-and-paste the lines below, but please do replace “username” with your own id.

BTW: You probably noted that you need to provide a unique name for each job you submit.
This is indeed a requirement in Kubernetes.
(You can reuse the same name after you delete an old job, of course)


###### job2.yaml:

```
apiVersion: batch/v1
kind: Job
metadata:
  name: job2-<username>
spec:
  completionMode: Indexed
  completions: 10
  parallelism: 10
  ttlSecondsAfterFinished: 1800
  template:
    spec:
      restartPolicy: OnFailure
      containers:
      - name: mypod
        image: centos:centos7
        resources:
           limits:
             memory: 1.5Gi
             cpu: 1
           requests:
             memory: 0.5Gi
             cpu: 0.1
        command: ["sh", "-c", "sleep 30; env; echo Done $JOB_COMPLETION_INDEX"]
```


Now let’s create the job:

```
kubectl create -f job2.yaml
```

Check for the job and the associated pods:

```
kubectl get jobs
kubectl get pods
```

Did it start 10 pods?

Now wait for them to finish, and then look at the stdout with
```
kubectl logs job2-<username>-<index>-<hash>
``` 

## The end

**Please make sure you did not leave any running pods or deployments. Jobs and associated completed pods are OK.**
