# Kubernetes for Science Tutorial

Storage in Kubernetes \
Hands on session

## Using ephemeral storage

Different Kubernetes clusters will have different storage options available.\
Let’s explore the most basic one: emptyDir. It will allocate the local scratch volume, which will be gone once the pod is destroyed.

You can copy-and-paste the lines below, but please do replace “username” with your own id;\
As mentioned before, all the participants in this hands-on session share the same namespace, so you will get name collisions if you don’t.

###### strg1.yaml:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: strg-<username>
  labels:
    k8s-app: strg-<username>
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: strg-<username>
  template:
    metadata: 
      labels:
        k8s-app: strg-<username>
    spec:
      containers:
      - name: mypod
        image: alpine
        resources:
           limits:
             memory: 600Mi
             cpu: 2
           requests:
             memory: 100Mi
             cpu: 100m
        command: ["sh", "-c", "apk add dumb-init && dumb-init -- sleep 100000"]
        volumeMounts:
        - name: mydata
          mountPath: /mnt/myscratch
      volumes:
      - name: mydata
        emptyDir: {}
```

Now let’s start the deployment:

```
kubectl create -f strg1.yaml
```

Now log into the created pod:

```
kubectl exec -it strg-<username>-<hash> -- /bin/sh
```

and look at the mounted filesystems.

```
df -k / /mnt/myscratch/
```

As you can see, they are completely separate.

Next, create

```
mkdir /mnt/myscratch/username
```

then store some files in it.

Also put some files in some other (unrelated) directories. e.g. 
`/root`

Now kill the container: `kill 1`, wait for a new one to be created.

You can see that the `RESTARTS` counter increased - this means a container inside the pod was restarted.

```
kubectl get pod strg-<username>-<hash>
```

Then log back in.

What happened to files?

Add a few more files in both `/mnt/myscratch/username` and `/root` directories.

Now, delete the pod and wait for a new pod to be created by the deployment.

Log into the new pod.

What happened to files this time?

You can now delete the deployment.

## Using persistent storage

In the cluster you are using we have a proper distributed filesystem, which allows using it for real data persistence.

To get storage, we need to create an abstraction called PersistentVolumeClaim.
By doing that we "Claim" some storage space - "Persistent Volume".
There will actually be PersistentVolume created, but it's a cluster-wide resource which you can not see.

Create the file (replace username as always):

###### pvc.yaml:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: vol-<username>
spec:
  storageClassName: rook-cephfs
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

We're creating a 1GB volume.

Look at it's status with 

```
kubectl get pvc vol-<username>
```
(replace username). 

The `STATUS` field should be equals to `Bound` - this indicates successful allocation.

Note that it may take a few seconds for the system to get there, so be patient.
You can check the progress with

```
kubectl get events --sort-by=.metadata.creationTimestamp --field-selector involvedObject.name=vol-<username>
```

Now we can attach it to our pod. Create one with (replacing `username`):

###### strg2.yaml:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: strg2-<username>
  labels:
    k8s-app: strg2-<username>
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: strg2-<username>
  template:
    metadata: 
      labels:
        k8s-app: strg2-<username>
    spec:
      containers:
      - name: mypod
        image: alpine
        resources:
           limits:
             memory: 600Mi
             cpu: 2
           requests:
             memory: 100Mi
             cpu: 100m
        command: ["sh", "-c", "apk add dumb-init && dumb-init -- sleep 100000"]
        volumeMounts:
        - name: mydata
          mountPath: /mnt/myscratch
      volumes:
      - name: mydata
        persistentVolumeClaim:
          claimName: vol-<username>
```

Now, repeat the steps from the previous execise section.

Start the deployment and log into the created pod.

Look at the mounted filesystems.

```
df -k / /mnt/myscratch/
```

Then create

```
mkdir /mnt/myscratch/username
```

and store some files in it.

Also put some files in some other (unrelated) directories. e.g. 

`/root`

Now kill the container: `kill 1`, wait for a new one to be created.

You can see that the `RESTARTS` counter increased - this means a container inside the pod was restarted.

```
kubectl get pod strg2-<username>-<hash>
```

Then log back in.

What happened to files?

Add a few more files in both `/mnt/myscratch/username` and `/root` directories.

Now, delete the pod and wait for a new pod to be created by the deployment.

Log into the new pod.

What happened to files this time?

You can now delete the deployment and the pvc:
```
kubectl delete deployment strg2-<username>
kubectl delete pvc vol-<username>
```


## Attaching existing storage

Sometimes you just need to attach to existing storage that is shared between multiple users.

In this cluster we have one CVMFS mountpoint available.

Let's mount it in our pod:

###### strg3.yaml:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: strg3-<username>
  labels:
    k8s-app: strg3-<username>
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: strg3-<username>
  template:
    metadata: 
      labels:
        k8s-app: strg3-<username>
    spec:
      containers:
      - name: mypod
        image: centos:centos7
        resources:
           limits:
             memory: 600Mi
             cpu: 2
           requests:
             memory: 100Mi
             cpu: 100m
        command: ["sh", "-c", "sleep infinity"]
        volumeMounts:
        - name: cvmfs
          mountPath: /cvmfs/oasis.opensciencegrid.org
          readOnly: true
        - name: mydata
          mountPath: /mnt/myscratch
      volumes:
      - name: cvmfs
        persistentVolumeClaim:
          claimName: cvmfs-osg-oasis
      - name: mydata
        emptyDir: {}
```

Now let’s start the deployment and log into the created pod.

Try to use git:
```
git
```

You will notice it is not present in the image.

So let's use the one on CVMFS, under `osg-software/osg-wn-client`:
```
source /cvmfs/oasis.opensciencegrid.org/osg-software/osg-wn-client/current/el7-x86_64/setup.sh 
git
```

Feel free to explore the rest of the mounted filesystem.

When you are done, delete the deployment.

## Explicitly moving files in

Most science users have large input data.
If someone has not already pre-loaded them on a PVC, you will have to fetch them yourself.

You can use any tool you are used to, from curl to ssh. You can either pre-load it to a PVC or fetch the data just-in-time, whateve more appropriate.

We do not have an explicit hands-on tutorial, but feel free to try out your favorite tool using what you have learned so far.

## Handling output data

Unlinke most batch systems, there is no shared filesystem between the submit host (aka your laptop) and the execution nodes.

You are responsible for explicit movement of the output files to a location that is useful for you.

The easiest option is to keep the output files on the PVC and do the follow-up analysis inside the kuberenetes cluster.

But when you want any data to be exported outside of the kubernetes cluster, you will have to do it explicitly.
You can use any (authenticated) file transfer tool from ssh to globus. Just remember to inject the necessary creatials into to pod, ideally using a secret.

For *small files*, you can also use the `kubectl cp` command.
It is similar to `scp`, but routes all traffic through a central kubernetes server, making it very slow.
Good enough for testing and debugging, but not much else. 

Again, we do not have an explict hands-on tutorial, and we discourage the uploading of any sensitive cretentials to this shared kubernetes setup.

## End
