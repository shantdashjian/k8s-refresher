# Kubernetes Review

## What is it?
- Kubernetes, or K8s for short, is a container orchestration tool. 
- A higher level definition of it is that it's a cloud native deployment and runtime environment. 
- Your application needs resources (CPU, memory, networking, etc.) to run and K8s abstract away all of that and takes care of scheduling those for you.
- K8s is a distributed system/cluster of servers/nodes that hosts applications.

## K8s is the Cloud Native Way
- K8s is the place where you deploy your application to run in the cloud in the **cloud native way**. It's the **cloud native way** because it provides **scaling** and **resilience** for your application in production in a **cost-effective** way and the deployment is **fast**. 

## How does it work?
- You use **YAML manifests** to declare what you want and K8s takes care of the rest.
- You use **kubectl** to manage K8s.
- K8s deploys and runs your application in a cluster of nodes.
- You can use **minikube** to run a local single-node cluster on your machine.
- For production, you'll use a **managed K8s service** from a cloud provider like AWS, Google Cloud, or Azure.

## Ecosystem:
1. Use Kubernetes Dashboard, Octant, Lens, or K9s to monitor the cluster.
2. Use KubeView to visualize it. Here are the [installation steps](kubeview-installation-steps.md).

## Deep Dive
1. **Packaging:** You package your application in an container image and push it to a container registry.
2. **Deployment:** To deploy it on K8s, you declare in a manifest which image you want. K8s will pull the image, run it as a container inside a pod. You tell it how many replicas you want. It will maintain that number of pods/replicas, a.k.a ReplicaSet, distributed over how many servers/nodes are available in the cluster. 
3. **Exposure:** To expose your app internally (or to the outside world), which is inside a container inside a pod inside a node:
    - You can expose a single pod using port forwarding. 
    - Better is to add a Service that will load balance the requests to all your pods in a deployment. Port forward to the service.  
    - Best is to add an Ingress, which will expose any or all your of services to the outside world, acting as an edge service/API gateway, taking care of security, load balancing, and other cross-cutting concerns. 
    - Locally, you open up a tunnel to your cluster to get the requests to the Ingress. 
    - In prod, it will be managed by the cloud service via an external cloud service provided load balancer that will route the requests to your cluster and Ingress.
4. **Environment Variables:** Use ConfigMap for non-sensitive env vars, and Secrets for sensitive ones.
5. **Storage:** K8s enables you to store data in the file system of the container, in a temporary volume inside the pod, or in a persistant volume in the cluster. Best is to use the cloud provider service for persistence.
6. **Security:** Use Namespaces to isolate kubernetes objects into groups. Use Secrets to store sensitive information.
6. **Guidelines for resource requests and limits:**
    1. Set memory requests ~10% higher than the average memory usage of your pods
    2. Set CPU requests to 50% of the average CPU usage of your pods
    3. Set memory limits ~100% higher than the average memory usage of your pods
    4. Set CPU limits ~100% higher than the average CPU usage of your pods

## Commands and Code

### Start the cluster and see it in the Kubernetes Dashboard on http://localhost:63840 and  KubeView on http://localhost:7000:
```
> minikube start

> minikube dashboard --port=63840

> helm install kubeview ./kubeview -f myvalues.yaml

> kubectl port-forward service/kubeview 7000:80
```

### web-deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: synergychat-web
  labels:
    app: synergychat-web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: synergychat-web
  template:
    metadata:
      labels:
        app: synergychat-web
    spec:
      containers:
      - name: synergychat-web
        image: docker.io/bootdotdev/synergychat-web:latest
        ports:
        - containerPort: 8080
```

```
> kubectl apply -f web-deployment.yaml

> kubectl get deployments

> kubectl get pods

> kubectl port-forward PODNAME 8080:8080
```

```
> kubectl logs PODNAME

> kubectl delete pod PODNAME

> kubectl get pods -o wide

> kubectl get replicasets
```

### web-configmap.yaml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: synergychat-web-configmap
data:
  WEB_PORT: "8080"
  API_URL: http://synchatapi.internal
```

```
> kubectl apply -f web-configmap.yaml

> kubectl get configmaps
```

### In web-deployment.yaml, spec > template > spec > containers > CONTAINER_ENTRY > you add all env vars like this:
```
envFrom:
  - configMapRef:
      name: synergychat-web-configmap
```     
### Or individually:

```
env:
  - name: API_PORT
    valueFrom:
      configMapKeyRef:
        name: synergychat-api-configmap
        key: API_PORT
```
### web-service.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  selector:
    app: synergychat-web
  ports:
  - port: 80
    protocol: TCP
    targetPort: 8080
```

```
> kubectl apply -f web-service.yaml

> kubectl port-forward service/web-service 8080:80

> kubectl get svc
```

### Service Types:
1. ClusterIP (the default)
2. NodePort
3. LoadBalancer
4. ExternalName

### Ingress
```
> minikube addons enable ingress
```

### app-ingress.yaml
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: synchat.internal
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web-service
                port:
                  number: 80
    - host: synchatapi.internal
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

```
> kubectl apply -f app-ingress.yaml
```

### Add in /etc/hosts:
```
127.0.0.1        synchat.internal
127.0.0.1        synchatapi.internal
```

### Open a tunnel to your cluster:
```
> minikube tunnel -c
```

### Temporary Volume:
In spec > template > spec:
```
volumes:
  - name: cache-volume
    emptyDir: {}
```

In the container entry:
```
volumeMounts:
  - name: cache-volume
    mountPath: /cache
```

### Persistant Volume:
### api-pvc.yaml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: synergychat-api-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```
> kubectl get pvc

> kubectl get pv
```

```
volumes:
  - name: synergychat-api-volume
    persistentVolumeClaim:
      claimName: synergychat-api-pvc
```
```
volumeMounts:
  - name: synergychat-api-volume
    mountPath: /persist
```

### Namespaces are a way to isolate resources into groups, for security.

```
> kubectl get namespaces

> kubectl get ns

> kubectl -n kube-system get pods

> kubectl create ns crawler
```

### You can refer to a service inside a namespace like this:
```
http://<service-name>.<namespace>.svc.cluster.local
```

### To see resource usage:
```
> minikube addons enable metrics-server

> kubectl top pod
```

### Limit resources for a container:
```
spec:
  containers:
    - image: bootdotdev/synergychat-testcpu:latest
      name: synergychat-testcp
      resources:
        limits:
          cpu: 10m
          memory: 400Mi
```

### HPA: Horizontal Pod Autoscaler
- Let's K8s autoscale the app for you.
### web-hpa.yaml
```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: synergychat-web
  minReplicas: 1
  maxReplicas: 4
  targetCPUUtilizationPercentage: 50

```

### Nodes:
```
> kubectl get nodes
```

### Request sources:
```
resources:
  limits:
    memory: 800Mi
  requests:
    memory: 410Mi
```

```
> kubectl describe pod <pod-name>
```

## Stop and delete the cluster
```
> minikube stop

> minikube delete
```