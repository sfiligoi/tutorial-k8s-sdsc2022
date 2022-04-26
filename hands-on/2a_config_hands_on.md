# Kubernetes for Science Tutorial

**Configuration management\
Hands on session**

## Simple config files

Let's start with creating a simple test file and import it into a pod.

Create a local file named `hello.txt` with any content you fancy.

Let's now import this file into a configmap (replace username, as before):
```
kubectl create configmap config1-<username> --from-file=hello.txt
```

Can you see it?
```
kubectl get configmap
```

You can also look at its content with
```
kubectl get configmap -o yaml config1-<username>
```

Import that file into a pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: c1-<username>
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
    volumeMounts:
    - name: hello
      mountPath: /tmp/hello.txt
      subPath: hello.txt
  volumes:
  - name: hello
    configMap:
      name: config1-<username>
      items:
      - key: hello.txt
        path: hello.txt
```

Create the pod and once it has started, login using kubectl exec and check if the file is indeed in the /tmp directory.

Inside the pod, make some changes to 
```
/tmp/hello.txt
```

Did that affect the content of the configmap?

What happens if you delete and recreate the pod?

You can now delete the pod and the configmap:
```
kubectl delete pod c1-<username>
kubectl delete configmap config1-<username> 
```

## Importing a whole directory

Create an additional local file named `world.txt` with any content you fancy.

Let's now import this file into a configmap (replace username, as before):
```
kubectl create configmap config2-<username> --from-file=hello.txt --from-file=world.txt
```

Check its content, as before.

Let's now import the whole configmap into a pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: c2-<username>
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
    volumeMounts:
    - name: cfg
      mountPath: /tmp/myconfig
  volumes:
  - name: cfg
    configMap:
      name: config2-<username>
```

Create the pod and once it has started, login using kubectl exec and check if the file is indeed in the /tmp/myconfig directory.

You can now delete the pod and the configmap:
```
kubectl delete pod c2-<username>
kubectl delete configmap config2-<username> 
```

## Importing code

Let's now do something (semi) useful. Let's [compute pi](https://www.geeksforgeeks.org/calculate-pi-with-python/).

###### pi.py:

```
import os

itr=1000
if 'PI_STEPS' in os.environ:
  itr=int(os.environ['PI_STEPS'])


# compute pi
k = 1
pi = 0

print("Computing pi using %i steps"%itr)
for i in range(itr):
    pi += (1 if ((i % 2)==0) else -1) * 4/k
    k += 2
     
print(pi)
```
As before, let's put it in a configmap:
```
kubectl create configmap config3-<username> --from-file=pi.py
```

Now import that file into a pod and compute pi:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: c3-<username>
spec:
  restartPolicy: Never
  containers:
  - name: mypod
    image: tensorflow/tensorflow:latest
    resources:
      limits:
        memory: 1.5Gi
        cpu: 1
      requests:
        memory: 500Mi
        cpu: 1
    command: ["sh", "-c", "python3 /opt/code/pi.py; echo Done"]
    volumeMounts:
    - name: pi
      mountPath: /opt/code/pi.py
      subPath: pi.py
  volumes:
  - name: pi
    configMap:
      name: config3-<username>
      items:
      - key: pi.py
        path: pi.py
```

Once the pod terminated, check if the result is correct:
```
kubectl logs c3-<usernam>
```

You may have noticed that we used the default 1000 steps. Let's get more precision, and use 100000 steps.
Our script can be configured by passing the appropriate environment variable (and no other changes are needed):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: c3m-<username>
spec:
  restartPolicy: Never
  containers:
  - name: mypod
    image: tensorflow/tensorflow:latest
    resources:
      limits:
        memory: 1.5Gi
        cpu: 1
      requests:
        memory: 500Mi
        cpu: 1
    command: ["sh", "-c", "python3 /opt/code/pi.py; echo Done"]
    env:
    - name: PI_STEPS
      value: "100000"
    volumeMounts:
    - name: pi
      mountPath: /opt/code/pi.py
      subPath: pi.py
  volumes:
  - name: pi
    configMap:
      name: config3-<username>
      items:
      - key: pi.py
        path: pi.py
```

Once the pod terminated, check if the result is correct:
```
kubectl logs c3m-<usernam>
```

You can now delete the pods and the configmap:
```
kubectl delete pod c3m-<username>
kubectl delete pod c3-<username>
kubectl delete configmap config3-<username> 
```

## Using a secret

Secrets are very similar to configmaps, but provide a little additional protecton for sensitive information.

Create a local file named `mysecret.txt` with any content you fancy.

Let's now import this file into a secret object (replace username, as before):
```
kubectl create secret generic secret1-<username> --from-file=mysecret.txt
```

Can you see it?
```
kubectl get secret
```

You can also look at its content with
```
kubectl get secret -o yaml secret1-<username>
```

How does it differ from the configmap?


Import that file into a pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: c4-<username>
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
    volumeMounts:
    - name: s1
      mountPath: /etc/secret/mysecret.txt
      subPath: mysecret.txt
  volumes:
  - name: s1
    secret:
      secretName: secret1-<username>
      defaultMode: 384
      items:
      - key: mysecret.txt
        path: mysecret.txt
```

Create the pod and once it has started, login using kubectl exec and check if the file is indeed in the /etc/secret directory.

Can you see its content?

Do you think you could access the secret of any other user in the namespace that way, too?

You can now delete the pod and the configmap:
```
kubectl delete pod c4-<username>
kubectl delete secret config4-<username> 
```

## Custom images

We do not have an explicit custom image creation hands-on session.

Advanced users are encouraged to try that on their own after the tutorial.

## The end

Please make sure you did not leave any configmaps, secrets or pods behind.
