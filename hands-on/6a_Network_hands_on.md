# Kubernetes for Science Tutorial

Networking in Kubernetes\
Hands on session

## Web-enabled applications

These days everything seems to be Web-first. Can we support this in kubernetes?

Let's start with Jupyter notebooks. They are very convenient for interactive analyses, but they are driven through a Web browser.

You can copy-and-paste the lines below to create a deplyment, but please do replace “username” with your own id.

###### jn.yaml:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jn-<username>
  labels:
    k8s-app: jn-<username>
spec:
  selector:
    matchLabels:
      k8s-app: jn-<username>
  template:
    metadata: 
      labels:
        k8s-app: jn-<username>
    spec:
      containers:
      - name: mypod
        image: gitlab-registry.nrp-nautilus.io/prp/jupyter-stack/tensorflow
        command: ["sleep", "infinity"]
        resources:
           limits:
             memory: 2.5Gi
             cpu: 1
             nvidia.com/gpu: 1
           requests:
             memory: 1.5Gi
             cpu: 0.1
             nvidia.com/gpu: 1
```

Make sure the associated pod is running, and log into it:
```
kubectl exec -it jn-<username>-<hash>  -- /bin/bash
```

You could now invoke python3, load tensorflow and, e.g., do your ML research. 

But doing everything on the command line may be tedious and makes it hard to visualize any results.

Let's start a jupyter notebook instead:
```
jupyter notebook --ip='0.0.0.0'
```

You will get a printout like this:
```
[I 19:01:18.125 NotebookApp] Jupyter Notebook 6.4.8 is running at:
[I 19:01:18.125 NotebookApp] http://jn-<username>-<hash>:8888/?token=f843f35d1f8191b907c576c4e6b9f41766bad41320495e5a
[I 19:01:18.125 NotebookApp]  or http://127.0.0.1:8888/?token=f843f35d1f8191b907c576c4e6b9f41766bad41320495e5a
[I 19:01:18.125 NotebookApp] Use Control-C to stop this server and shut down all kernels (twice to skip confirmation).
[C 19:01:18.129 NotebookApp] 
    
    To access the notebook, open this file in a browser:
        file:///home/jovyan/.local/share/jupyter/runtime/nbserver-76-open.html
    Or copy and paste one of these URLs:
        http://jn-<username>-<hash>:8888/?token=f843f35d1f8191b907c576c4e6b9f41766bad41320495e5a
     or http://127.0.0.1:8888/?token=f843f35d1f8191b907c576c4e6b9f41766bad41320495e5a
```

You cannot direclty access `jn-<username>-<hash>` URL and the `127.0.0.1` URL is obviously local to the pod.
In order to access it from your laptop, you must put in place port-forwarding (possibly in a different terminal):

```
kubectl port-forward jn-<username>-<hash> 8888:8888
```

You can now use the 127.0.0.1 URL from your local Web browser and access the (remote) jupyter notebook.
(copy the whole URL into your browser, as it contains the authentication token)

If you have any experience with jupyter, feel free to play with it for a bit, including python3 notebooks and terminal.

When you are done, just Ctrl-C the two connections and delete the deployment.

## Horizontal scaling exercise

Orchestration is often used to spread the load over multiple nodes.

In this exercise, we will launch multiple Web servers.

To make distinguishing the two servers easier, we will force the nodename into their homepages. Using stock images, we achieve this by using an init container.

You can copy-and-paste the lines below, but please do replace “username” with your own id.

###### http2.yaml:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: http-<username>
  labels:
    k8s-app: http-<username>
spec:
  replicas: 2
  selector:
    matchLabels:
      k8s-app: http-<username>
  template:
    metadata: 
      labels:
        k8s-app: http-<username>
    spec:
      initContainers:
      - name: myinit
        image: busybox
        command: ["sh", "-c", "echo '<html><body><h1>I am ' `hostname` '</h1></body></html>' > /usr/local/apache2/htdocs/index.html"]
        volumeMounts:
        - name: dataroot
          mountPath: /usr/local/apache2/htdocs
      containers:
      - name: mypod
        image: httpd:alpine
        resources:
           limits:
             memory: 1.5Gi
             cpu: 1
           requests:
             memory: 0.5Gi
             cpu: 0.1
        volumeMounts:
        - name: dataroot
          mountPath: /usr/local/apache2/htdocs
      volumes:
      - name: dataroot
        emptyDir: {}
```

BTW: Feel free to change the number of replicas (within reason) and the text it is shown in home page of each server, if so desired.

Launch the deployment:

```
kubectl create -f http2.yaml
```

Also launch the pod1 from basic hand on excercise.

Check the pods you have, alongside the IPs they were assigned to:

```
kubectl get pods -o wide
```

Log into pod1

```
kubectl exec -it pod-<username> -- /bin/sh
```

Now try to pull the home pages from the two Web servers; use the IPs you obtained above:\
curl http://*IPofPod*

You should get a different answer from the two.

## Load balancing

Having to manually switch between the two Pods is obviously tedious. What we really want is to have a single logical address that will automatically load-balance between them.

You can copy-and-paste the lines below, but please do replace “username” with your own id.

###### svc2.yaml:

```
apiVersion: v1
kind: Service
metadata:
  labels:
    k8s-app: svc-<username>
  name: svc-<username>
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    k8s-app: http-<username>
  type: ClusterIP
```

Let’s now start the service:

```
kubectl create -f svc2.yaml
```

Look up your service, and write down the IP it is reporting under:

```
kubectl get services
```

Log into pod1

```
kubectl exec -it pod-<username> -- /bin/sh
```

Now try to pull the home page from the service IP:

```
wget http://IPofService
cat index.html
rm index.html
```

Try it a few times… Which Web server is serving you?

Note that you can also use the local DNS name for this (from pod1)

```
wget http://svc-<username>.sdsc-tutorial.svc.cluster.local
cat index.html
rm index.html
```

## Exposing public services

Sometimes you have the opposite problem; you want to export resources of a single node to the public internet.

The above Web services only serve traffic on the private IP network LAN. If you try curl from your laptop, you will never reach those Pods!

What we need is set up an Ingress instance for our service.

###### ingress.yaml:

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: haproxy
  name: ingress-<username>
spec:
  rules:
  - host: <username>.nrp-nautilus.io
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: svc-<username>
            port:
              number: 80
  tls:
  - hosts:
    - <username>.nrp-nautilus.io
```

Launch the new ingress

```
kubectl create -f ingress.yaml
```

See if it succeeded:

```
kubectl get ingress
```

You should now be able to fetch the Web pages from your browser by opening `https://<username>.nrp-nautilus.io`. Note that SSL termination is already provided for you.

You can now delete the ingress:

```
kubectl delete -f ingress.yaml
```

## The end

**Please make sure you did not leave any running pods, deployments, ingresses or services behind.**
