# In this lab

This lab provides the instructions to:

* [Overview](https://github.com/tigera-cs/Calico-Security-Observability-Troubleshooting-Training/blob/main/modules/2.%20Secure%20Pod%20Traffic%20Using%20Calico%20Security%20Policy/README.md#overview)
* [Cluster applications and connectivity](https://github.com/tigera-cs/Calico-Security-Observability-Troubleshooting-Training/blob/main/2.%20Secure%20Pod%20Traffic%20Using%20Calico%20Security%20Policy/README.md#cluster-applications-and-connectivity)
* [Deploy Sample Microservices]()
* [Install Calico Enterprise command line utility "calicoctl"](https://github.com/tigera-cs/Calico-Security-Observability-Troubleshooting-Training/blob/main/1.%20Install%20Calico%20Enterprise/README.md#install-calico-enterprise-command-line-utility-calicoctl)
* [Deploy a three-tier sample application called "yaobank" (Yet Another Online Bank)](https://github.com/tigera-cs/Calico-Enterprise-Networking-Training/blob/main/1.%20Install%20Calico%20Enterprise/README.md#deploy-a-three-tier-sample-application-called-yaobank-yet-another-online-bank)
* [Access CE Manager UI using Ingress](https://github.com/tigera-cs/Calico-Enterprise-Networking-Training/blob/main/1.%20Install%20Calico%20Enterprise/README.md#access-ce-manager-ui-using-ingress)


### Overview

Kubernetes' network model is both flexible and dynamic, enabling applications to be deployed anywhere in the cluster without being tied to the underlying network infrastructure. However, this can present challenges for legacy network firewalls that struggle to keep up with Kubernetes' constantly changing workloads. Calico provides a robust security policy framework that defines and enforces policies, ensuring that only authorized traffic flows between pods and services. Calico's powerful policy engine enhances the security of microservices architectures without compromising the flexibility and agility of Kubernetes. In this lab, we'll demonstrate how to use Calico's policy engine to secure some example microservices that blong to different tenants.

### Cluster applications and connectivity

This lab uses two applications that run across 4 namespaces and belong to two tenants, Tenant-1 and Tenant-2. Star app pods run in three namespaces (management-ui, client, stars) and belong to Tenant-1. Yaobank pods runs in a single and belong to Tenant-2. The ingress-nginx namespace hosts an ingress controller, which will be used to connect to the applications through the respective frontend microservice for each application. The ingress-nginx and kube-system namespaces are owned by the platform and are managed independent of the tenant namespaces.

#### `Tenant-1`
  - Star app
    -  management-ui namesapce
    -  client namesapce
    -  stars namesapce
  
#### `Tenant-2`
  - Yaobank app
    -  yaobank namespace



<h1 align="center">Cluster Topology</h1>


<p align="center">
<img src="img/3.tenants.png">
</p>


### Deploy Sample Microservices

#### Deploy `Star` App

To deploy the Star app, deploy the following manifest in the cluster.

```yaml
kubectl apply -f -<<EOF
kind: Namespace
apiVersion: v1
metadata:
  name: stars
---
apiVersion: v1
kind: Namespace
metadata:
  name: management-ui
  labels:
    role: management-ui
---
kind: Namespace
apiVersion: v1
metadata:
  name: client
  labels:
    role: client
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: stars
spec:
  ports:
    - port: 6379
      targetPort: 6379
  selector:
    role: backend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: stars
  labels:
    role: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      role: backend
  template:
    metadata:
      labels:
        role: backend
    spec:
      containers:
        - name: backend
          image: calico/star-probe:multiarch
          imagePullPolicy: Always
          command:
            - probe
            - --http-port=6379
            - --urls=http://frontend.stars:80/status,http://backend.stars:6379/status,http://client.client:9000/status
          ports:
            - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: stars
spec:
  ports:
    - port: 6379
      targetPort: 6379
  selector:
    role: backend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: stars
  labels:
    role: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      role: backend
  template:
    metadata:
      labels:
        role: backend
    spec:
      containers:
        - name: backend
          image: calico/star-probe:multiarch
          imagePullPolicy: Always
          command:
            - probe
            - --http-port=6379
            - --urls=http://frontend.stars:80/status,http://backend.stars:6379/status,http://client.client:9000/status
          ports:
            - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
  namespace: stars
spec:
  ports:
    - port: 80
      targetPort: 80
  selector:
    role: frontend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: stars
  labels:
    role: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      role: frontend
  template:
    metadata:
      labels:
        role: frontend
    spec:
      containers:
        - name: frontend
          image: calico/star-probe:multiarch
          imagePullPolicy: Always
          command:
            - probe
            - --http-port=80
            - --urls=http://frontend.stars:80/status,http://backend.stars:6379/status,http://client.client:9000/status
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: management-ui
  namespace: management-ui
spec:
  ports:
    - port: 9001
      targetPort: 9001
  selector:
    role: management-ui
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: management-ui
  namespace: management-ui
  labels:
    role: management-ui
spec:
  replicas: 1
  selector:
    matchLabels:
      role: management-ui
  template:
    metadata:
      labels:
        role: management-ui
    spec:
      containers:
        - name: management-ui
          image: calico/star-collect:multiarch
          imagePullPolicy: Always
          ports:
            - containerPort: 9001
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client
  namespace: client
  labels:
    role: client
spec:
  replicas: 1
  selector:
    matchLabels:
      role: client
  template:
    metadata:
      labels:
        role: client
    spec:
      containers:
        - name: client
          image: calico/star-probe:multiarch
          imagePullPolicy: Always
          command:
            - probe
            - --urls=http://frontend.stars:80/status,http://backend.stars:6379/status
          ports:
            - containerPort: 9000
---
apiVersion: v1
kind: Service
metadata:
  name: client
  namespace: client
spec:
  ports:
    - port: 9000
      targetPort: 9000
  selector:
    role: client
EOF

```

You should see an output similar to the following for each one of the Star app namesapces.

```bash
kubectl get pods -n management-ui

```
```bash
NAME                             READY   STATUS    RESTARTS        AGE
management-ui-6c47d46679-xs4h5   1/1     Running   1 (5h56m ago)   27h          -------> to be replaced
```

```bash
kubectl get pods -n client

```
```bash
NAME                      READY   STATUS    RESTARTS     AGE
client-66b456486c-dcxg7   1/1     Running   1 (6h ago)   27h          -------> to be replaced
```

```bash
kubectl get pods -n stars

```
```bash
NAME                       READY   STATUS    RESTARTS       AGE
backend-799bfd69f7-sll9x   1/1     Running   1 (6h1m ago)   27h
frontend-db76f566f-sgm8v   1/1     Running   1 (6h1m ago)   27h         -------> to be replaced
```


```yaml
kubectl apply -f -<<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: yaobank
  labels:
    istio-injection: disabled
---
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: yaobank
  labels:
    app: database
spec:
  ports:
  - port: 2379
    name: http
  selector:
    app: database
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: database
  namespace: yaobank
  labels:
    app: yaobank
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: database
  namespace: yaobank
spec:
  selector:
    matchLabels:
      app: database
      version: v1
  replicas: 1
  template:
    metadata:
      labels:
        app: database
        version: v1
    spec:
      serviceAccountName: database
      containers:
      - name: database
        image: calico/yaobank-database:certification
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 2379
        command: ["etcd"]
        args:
          - "-advertise-client-urls"
          - "http://database:2379"
          - "-listen-client-urls"
          - "http://0.0.0.0:2379"
---
apiVersion: v1
kind: Service
metadata:
  name: summary
  namespace: yaobank
  labels:
    app: summary
spec:
  ports:
  - port: 80
    name: http
  selector:
    app: summary  
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: summary
  namespace: yaobank
  labels:
    app: yaobank
    database: reader
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: summary
  namespace: yaobank
spec:
  replicas: 2
  selector:
    matchLabels:
      app: summary
      version: v1
  template:
    metadata:
      labels:
        app: summary
        version: v1
    spec:
      serviceAccountName: summary
      containers:
      - name: summary
        image: calico/yaobank-summary:certification
        imagePullPolicy: Always
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: customer
  namespace: yaobank
  labels:
    app: customer
spec:
  type: NodePort
  ports:
  - port: 80
    nodePort: 30180
    name: http
  selector:
    app: customer 
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: customer
  namespace: yaobank
  labels:
    app: yaobank
    summary: reader 
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: customer
  namespace: yaobank
spec:
  replicas: 1
  selector:
    matchLabels:
      app: customer
      version: v1
  template:
    metadata:
      labels:
        app: customer
        version: v1
    spec:
      serviceAccountName: customer
      containers:
      - name: customer
        image: calico/yaobank-customer:certification
        imagePullPolicy: Always
        ports:
        - containerPort: 80
---
EOF

```

You should an output similar to the following for yaobank application.

```bash
kubectl get pods -n yaobank

```

```bash
NAME                        READY   STATUS    RESTARTS   AGE
customer-85658df659-nnlx2   1/1     Running   0          5m50s
database-674c4bf5f4-df2jf   1/1     Running   0          5m51s
summary-755668b699-9nvzj    1/1     Running   0          5m51s
summary-755668b699-q6z9c    1/1     Running   0          5m51s
```

