# In this lab

This lab provides the instructions to:

* [Overview](https://github.com/tigera-cs/Calico-Security-Observability-Troubleshooting-Training/blob/main/1.%20Install%20Calico%20Enterprise/README.md#overview)
* [Enable Calico Wireguard encryption]()



### Overview

Network encryption is crucial for maintaining the security and privacy of data and communication within a Kubernetes cluster. It plays a significant role in ensuring that information remains confidential and protected. Network encryption provides protection against unauthorized access and eavesdropping. Additionally, it helps meet compliance regulations and safeguards against man-in-the-middle attacks. Implementing network encryption is vital for preserving the integrity and privacy of data, adhering to regulatory standards, and establishing secure Kubernetes deployments. Wireguard is a simple, lightweight, and performant tunneling protocol that enables us to establish secure connections. Calico leverages Wireguard to deliver streamlined and automated encryption for Kubernetes clusters, making the process of securing network connections easier and more efficient. In this lab, we will learn about how to use Calico Wireguard implementation to secure pod connectivity in a Kubernetes cluster. In addition to the pod traffic, Calico provides node-to-node communication encryption for managed clusters deployed on EKS (AWS CNI) and AKS (Azure CNI).




#### Documentation

- https://docs.tigera.io/calico-enterprise/latest/compliance/encrypt-cluster-pod-traffic


____________________________________________________________________________________________________________________________________________________________________________________


# Section 1. Deploy the troubleshooting application and secure it

### a. The application diagram is shown below: 

<p align="center">
  <img src="Images/3.m1lab4-1.png" alt="Connect Cluster" align="center" width="1000">
</p>

| Deployment Name         | Label                     | Service Port/Proto  |
| :-----------:           | :-------------:           | :-----------------: |
| adservice               | app=adservice             | 9555/TCP            |
| cartservice             | app=cartservice           | 7070/TCP            |
| checkoutservice         | app=checkoutservice       | 5050/TCP            |
| currencyservice         | app=currencyservice       | 7000/TCP            |
| emailservice            | app=emailservice          | 5000/TCP            |
| frontend                | app=frontend              | 80/TCP              |
| loadgenerator           | app=loadgenerator         |                     |
| paymentservice          | app=paymentservice        | 50051/TCP           |
| productcatalogservice   | app=productcatalogservice | 3550/TCP            |
| recommendationservice   | app=recommendationservice | 8080/TCP            |
| redis-cart              |app=redis-cart             | 6379/TCP            |
| shippingservice         | app=shippingservice       | 50051/TCP           |


1. Clone the Observability Clinic Repo and grant the executable permission for the lab's script.

```bash
git clone https://github.com/tigera-cs/observability-clinic.git

```

```bash
chmod +x /home/tigera/observability-clinic/tsworkshop/workshop1/lab-script.sh 

```

2. Create a namespace `hipstershop`, which the application will be deployed in.

```bash
kubectl create namespace hipstershop

```

3. Deploy the application Online Boutique (Hipstershop) to the namespace. This will install the application from the Google repository.

```bash
kubectl apply -n hipstershop -f https://raw.githubusercontent.com/GoogleCloudPlatform/microservices-demo/v0.3.9/release/kubernetes-manifests.yaml

```

4. Wait for all PODs get in a running status.

```bash
watch kubectl get pods -n hipstershop

```
You should see an output similar to the following.

```bash
NAME                                     READY   STATUS    RESTARTS   AGE
adservice-6f498fc6c6-c5rhh               1/1     Running   0          2m40s
cartservice-bc9b949b-rgqpc               1/1     Running   0          2m40s
checkoutservice-598d5b586d-nxjck         1/1     Running   0          2m41s
currencyservice-6ddbdd4956-vjzs8         1/1     Running   0          2m40s
emailservice-68fc78478-qg8qp             1/1     Running   0          2m41s
frontend-5bd77dd84b-l8qqx                1/1     Running   0          2m41s
loadgenerator-8f7d5d8d8-d7gwj            1/1     Running   0          2m40s
paymentservice-584567958d-hr5vl          1/1     Running   0          2m41s
productcatalogservice-75f4877bf4-jvnlq   1/1     Running   0          2m40s
recommendationservice-646c88579b-2t55b   1/1     Running   0          2m41s
redis-cart-5b569cd47-l29tx               1/1     Running   0          2m40s
shippingservice-79849ddf8-nlfjv          1/1     Running   0          2m40s
```

5. Configure the ingress for hipstershop application.

```yaml
kubectl apply -f -<<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: "nginx"
  name: hipstershop-ingress
  namespace: hipstershop
spec:
  rules:
  - host: "hipstershop.calico.labs.tigera.fr"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
EOF
```

6. Check the connectivity to hipstershop application. `https:\\hipstershop.<LabName>.labs.tigera.fr` via your browser.


7. Apply the following GlobalNetworkSets.

```yaml
kubectl apply -f -<<EOF
# apiVersion: projectcalico.org/v3
# kind: GlobalNetworkSet
# metadata:
#   name: loopback-cidr
#   labels:
#     loopback-cidr: 'true'
# spec:
#   nets:
#     - 127.0.0.0/8
---
apiVersion: projectcalico.org/v3
kind: GlobalNetworkSet
metadata:
  name: bastion
  labels:
    bastion: 'true'
    type: 'bastion'
spec:
  nets:
    - 10.0.1.10/32
---
apiVersion: projectcalico.org/v3
kind: GlobalNetworkSet
metadata:
  name: kube-api
  labels:
    kube-api: 'true'
spec:
  allowedEgressDomains:
    - '*.labs.tigera.fr'
---
apiVersion: projectcalico.org/v3
kind: GlobalNetworkSet
metadata:
  name: trusted-repos
  labels:
    external-ep: trusted-repos
spec:
  allowedEgressDomains:
    - '*.docker.com'
    - '*.quay.io'
    - '*.ubuntu.com'
    - '*.docker.io'
    - gcr.io
EOF
```

The globalnetworksets applied:
* loopback: to account for the communication with loopback interface once we enable host endpoint protection (typical behavior of some linux kernel and a common trap to avoid)
* bastion: to account for communication with bastion host, namely ssh and kube api tcp 6443 port
* kube-api (*.lynx.tigera.ca): to account for kube-api fqdn, wildcard used for simplicity
* trusted-repos: to be able to pull the images for our pods


Apply the Security Policies for the Hipstershop Application:

```yaml
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app.checkoutservice
  namespace: hipstershop
spec:
  tier: app
  order: 650
  selector: app == "checkoutservice"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: app == "frontend"
      destination:
        ports:
          - "5050"
  egress:
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "cartservice"
        ports:
          - "7070"
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "emailservice"
        ports:
          - "8080"
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "paymentservice"
        ports:
          - "50051"
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "productcatalogservice"
        ports:
          - "3550"
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "shippingservice"
        ports:
          - "50051"
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "currencyservice"
        ports:
          - "7000"
  types:
    - Ingress
    - Egress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app.cartservice
  namespace: hipstershop
spec:
  tier: app
  order: 875
  selector: app == "cartservice"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: app == "checkoutservice"||app == "frontend"
      destination:
        ports:
          - "7070"
  egress:
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "redis-cart"
        ports:
          - "6379"
  types:
    - Ingress
    - Egress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app.frontend
  namespace: hipstershop
spec:
  tier: app
  order: 1100
  selector: app == "frontend"
  ingress:
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        ports:
          - "8080"
  egress:
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "cartservice"
        ports:
          - "7070"
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "productcatalogservice"
        ports:
          - "3550"
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "recommendationservice"
        ports:
          - "8080"
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "currencyservice"
        ports:
          - "7000"
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "checkoutservice"
        ports:
          - "5050"
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "shippingservice"
        ports:
          - "50051"
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "adservice"
        ports:
          - "9555"
  types:
    - Ingress
    - Egress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app.redis-cart
  namespace: hipstershop
spec:
  tier: app
  order: 1300
  selector: app == "redis-cart"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: app == "cartservice"
      destination:
        ports:
          - "6379"
  types:
    - Ingress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app.productcatalogservice
  namespace: hipstershop
spec:
  tier: app
  order: 1400
  selector: app == "productcatalogservice"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: >-
          app == "checkoutservice"||app == "frontend"||app ==
          "recommendationservice"
      destination:
        ports:
          - "3550"
  types:
    - Ingress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app.emailservice
  namespace: hipstershop
spec:
  tier: app
  order: 1500
  selector: app == "emailservice"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: app == "checkoutservice"
      destination:
        ports:
          - "8080"
  types:
    - Ingress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app.paymentservice
  namespace: hipstershop
spec:
  tier: app
  order: 1600
  selector: app == "paymentservice"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: app == "checkoutservice"
      destination:
        ports:
          - "50051"
  types:
    - Ingress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app.currencyservice
  namespace: hipstershop
spec:
  tier: app
  order: 1650
  selector: app == "currencyservice"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: app == "checkoutservice"||app == "frontend"
      destination:
        ports:
          - "7000"
  types:
    - Ingress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app.shippingservice
  namespace: hipstershop
spec:
  tier: app
  order: 1700
  selector: app == "shippingservice"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: app == "checkoutservice"||app == "frontend"
      destination:
        ports:
          - "50051"
  types:
    - Ingress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app.recommendationservice
  namespace: hipstershop
spec:
  tier: app
  order: 1800
  selector: app == "recommendationservice"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: app == "frontend"
      destination:
        ports:
          - "8080"
  egress:
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "productcatalogservice"
        ports:
          - "3550"
  types:
    - Ingress
    - Egress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app.adservice
  namespace: hipstershop
spec:
  tier: app
  order: 1900
  selector: app == "adservice"
  ingress:
    - action: Allow
      protocol: TCP
      source:
        selector: app == "frontend"
      destination:
        ports:
          - "9555"
  types:
    - Ingress
---
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app.allow-loadgenerator
  namespace: hipstershop
spec:
  tier: app
  order: 2000
  selector: app == "loadgenerator"
  egress:
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: app == "frontend"
        ports:
          - "8080"
  types:
    - Egress
EOF

```

# Section 2 - Investigate denied flows using the Calico Enterprise Observability tools


## 1. Identify denied flows using the Dynamic Service and Threat Graph

### Demo

#### a. Opening the lab and run the script below:

```bash
/home/tigera/observability-clinic/tsworkshop/workshop1/lab-script.sh
```

#### b. Typing the option “1” (Demo Break Online Boutique - Dynamic Service and Threat Graph) and press “Enter”

#### c. Type "99" and enter to exit

#### d. Open the Online Boutique with the URL shown through the command below and select any product:
```bash
echo https://hipstershop.$(kubectl cluster-info | grep -i control | awk -F "://" '{print $2}' | cut -d. -f1).lynx.tigera.ca
```

<p align="center">
  <img src="Images/1.m2lab1-1.png" alt="Online Boutique" align="center" width="800">
</p>

#### e. Add the product to the Cart:

<p align="center">
  <img src="Images/2.m2lab1-2.png" alt="Online Boutique - Add to the Cart" align="center" width="800">
</p>

#### f. Click place order:

<p align="center">
  <img src="Images/3.m2lab1-3.png" alt="Online Boutique - Place order" align="center" width="800">
</p>

#### g. The request didn’t complete and return HTTP 500 Internal error:

```nano
HTTP Status: 500 Internal Server Error

rpc error: code = Internal desc = failed to charge card: could not charge the card: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing dial tcp 10.49.11.61:50051: i/o timeout"
failed to complete the order
```
<p align="center">
  <img src="Images/4.m2lab1-4.png" alt="500 Internal Server Error" align="center" width="800">
</p>

#### h. Why not take some time to try to solve this issue yourself before continuing further

#### i. Go to the Dynamic Service and Threat Graph in the hipstershop namespace, where the Online Boutique application is running (Calico Cloud login -> <img src="Images/icon-1.png" alt="Service Graph" width="30"> -> Views: Default -> Double-click on hipstershop ns  <img src="Images/icon-2.png" alt="Hipstershop Namespace" width="40"> )

<p align="center">
  <img src="Images/5.m2lab1-5.png" alt="Service Graph" align="center" width="800">
</p>

#### j. From the Online Boutique Diagram, we can see that the CheckoutService will send the request to PaymentService:

<p align="center">
  <img src="Images/6.m2lab1-6.png" alt="Hipstershop Diagram" align="center" width="800">
</p>

#### k. Therefore, we can see some red arrows between the checkoutservice and paymentservice and, when we pass on the cursor under red arrow we can see denied flows in red:

<p align="center">
  <img src="Images/7.m2lab1-7.png" alt="Service Graph - Denied flows" align="center" width="800">
</p>

#### l. By selecting the red arrow, it will show the flows related to that arrow on the bottom of the screen:

<p align="center">
  <img src="Images/8.m2lab1-8.png" alt="Service Graph - Detailed flows" align="center" width="800">
</p>

#### m. When we expand one denied flow from checkoutservice to paymentservice, it shows many information and scrolling down we can see the Action =  Deny and the Policies = *“0|security|security.tenant-histershop|pass|0, 1|default|default.default-deny|deny|-1”*. This policy entry means that the flow did not match with any Security Policy rule configured and it ended up on an implicit default deny.

<p align="center">
  <img src="Images/9.m2lab1-9.png" alt="Service Graph - Detailed flow" align="center" width="800">
</p>

<p align="center">
  <img src="Images/10.m2lab1-10.png" alt="Service Graph - Denied flow" align="center" width="800">
</p>

#### n. As the flow did not match with security policy, let’s review the Security Policy created for the paymentservice.

#### o. When opening the Security Policy paymentservice, we can see that "0" Endpoints are associated with that policy:

<p align="center">
  <img src="Images/11.m2lab1-11.png" alt="Security Policy - No endpoints" align="center" width="800">
</p>

#### p. As the recommended way to match endpoints to Security Policy are labels, let’s check the labels from the paymentservice pod as per command below (the label can also be checked through the Tigera UI in Endpoint tab):

```bash
kubectl get po -n hipstershop --show-labels | grep payment
```
```bash
paymentservice-584567958d-5z8jx          1/1     Running   1 (38h ago)   7d    app=paymentservice,pod-template-hash=584567958d
```

#### q. We can see the label of the payment pod is app=paymentservice and in the Security Policy the label configured is app=paymentserviceee. Therefore there is a typo on the Security Policy label.

<p align="center">
  <img src="Images/12.m2lab1-12.png" alt="Security Policy - Wrong label selector" align="center" width="800">
</p>

#### r. After editing the Security Policy and inserting the right label for the paymentservice pod (app=paymentservice), we can see the Endpoints in the Security Policy shows “1”.

<p align="center">
  <img src="Images/13.m2lab1-13.png" alt="Security Policy - Right endpoint" align="center" width="800">
</p>

#### s. If we click on the “1” it will show the endpoint enforced by the Security Policy, in this case the paymentservice pod.

<p align="center">
  <img src="Images/14.m2lab1-14.png" alt="Security Policy - Endpoint" align="center" width="800">
</p>

#### t. After fixing the paymentservice label, we click on “Place Order” and it is completed (alternatively if you are still on the error 500 page, simply refresh the page):

<p align="center">
  <img src="Images/15.m2lab1-15.png" alt="Online Boutique - Order complete" align="center" width="800">
</p>

### LAB

#### a. Open the lab and run the script below:

```bash
/home/tigera/observability-clinic/tsworkshop/workshop1/lab-script.sh
```
#### b. Type the option “2” (LAB Break Online Boutique - Dynamic Service and Threat Graph) and press “Enter”

#### c. Type "99" and enter to exit

#### d. Open the browser with the URL shown through the command below:

```bash
echo https://hipstershop.$(kubectl cluster-info | grep -i control | awk -F "://" '{print $2}' | cut -d. -f1).lynx.tigera.ca
```

#### e. Change the currency = EUR and select any product:

<p align="center">
  <img src="Images/16.m2lab2-1.png" alt="Online Boutique - Currency to EUR" align="center" width="800">
</p>

#### f. Add any product to the Cart:

<p align="center">
  <img src="Images/17.m2lab2-2.png" alt="Online Boutique - Add to Cart" align="center" width="800">
</p>

#### g. Place the Order:

<p align="center">
  <img src="Images/18.m2lab2-3.png" alt="Online Boutique - Place Order" align="center" width="800">
</p>

#### h. The internal error below will return on converting the price to EUR:

```nano
HTTP Status: 500 Internal Server Error

rpc error: code = Internal desc = failed to prepare order: failed to convert price of "XXXXXXXXXX" to EUR
failed to complete the order
```

<p align="center">
  <img src="Images/19.m2lab2-4.png" alt="Online Boutique - 500 Internal Server Error" align="center" width="800">
</p>

#### i. Investigate through the Dynamic Service and Threat Graph which flows have been denied and the Security Policy related to it, and how to fix this issue. Use the hipstershop application information provided in the **[Module 1 - Topic 4](https://github.com/tigera-cs/observability-clinic/blob/main/1.%20Overview/readme.md#4-install-hipstershop-application)**. 

#### j. To revert back the misconfiguration applied, run the script and type “21” (LAB Fix Online Boutique - Dynamic Service and Threat Graph) and press Enter. To exit type "99" and press “Enter”.

## 2. Identify denied flows using the Flow Visualizations

### Demo

#### a. Opening the lab and run the script below:

```bash
/home/tigera/observability-clinic/tsworkshop/workshop1/lab-script.sh
```

#### b. Typing the option “3” (Demo Break Online Boutique - Flow Visualizations) and press “Enter”

#### c. Type "99" and “Enter” to exit

#### d. Opening the Online Boutique with the URL shown through the command below and Trying to access the Online Boutique we get the error: *“504 Gateway Time-out”*

```bash
echo https://hipstershop.$(kubectl cluster-info | grep -i control | awk -F "://" '{print $2}' | cut -d. -f1).lynx.tigera.ca
```

#### e. On Tigera-UI > <img src="Images/icon-3.png" alt="FlowViz" width="30"> > Flow Visualizations, we can select the Denied flows in the Status field:

<p align="center">
  <img src="Images/20.m2lab3-1.png" alt="FlowViz" align="center" width="800">
</p>

#### f. When selecting the Denied flow in the Flow Visualizations, the details about this flow shows in the right side of the screen. If we click on the “All Flows” dropdown, we can see the Denied flow from pvt (private network) to frontend illustrated with a red arrow.  Select this flow to see more information about it. Now we can see that the Security Policy that is denying this flow is hipstershop.app-hipstership.fronted

<p align="center">
  <img src="Images/21.m2lab3-2.png" alt="FlowViz - Denied flows" align="center" width="800">
</p>

#### g. When clicking on the Security Policy, it will redirect to the Security Policy (hipstershop.app-hipstership.fronted) configuration and when looking into the ingress rules, it is configured to Log the flow and not Allow it.

<p align="center">
  <img src="Images/22.m2lab3-3.png" alt="Security Policy" align="center" width="800">
</p>

#### h. After editing the Security Policy and changing the ingress rule to Allow instead of Log, the Online Boutique is accessible again.

### LAB

#### a. Open the lab and run the script below:

```bash
/home/tigera/observability-clinic/tsworkshop/workshop1/lab-script.sh
```

#### b. Type the option “4” (LAB Break Online Boutique - Flow Visualizations) and press “Enter”

#### c. Type "99" and “Enter” to exit

#### d. Open the browser with the URL shown through the command below:

```bash
echo https://hipstershop.$(kubectl cluster-info | grep -i control | awk -F "://" '{print $2}' | cut -d. -f1).lynx.tigera.ca
```

#### e. Select any product(s), add to the Cart and click on “Place Order”.


#### f. An internal server error will return saying *“failed to get product #"OLJCESPC7Z"“*:

```nano
HTTP Status: 500 Internal Server Error
rpc error: code = Internal desc = failed to prepare order: failed to get product #"OLJCESPC7Z"
failed to complete the order
```

<p align="center">
  <img src="Images/23.m2lab4-1.png" alt="Online Boutique - Error" align="center" width="800">
</p>

#### g. Investigate through the Flow Visualizations which flows have been denied and the Security Policy related to it, and how to fix this issue. Use the hipstershop application information provided in the **[Module 1 - Topic 4](https://github.com/tigera-cs/observability-clinic/blob/main/1.%20Overview/readme.md#4-install-hipstershop-application)**.

#### h. To revert back the misconfiguration applied, run the script and type “41” (LAB Fix Online Boutique - Flow Visualizations) and press Enter. To exit type "99" and press “Enter”.

## 3. Identify denied flows using Kibana

### Demo

#### a. Opening the lab and run the script below:

```bash
/home/tigera/observability-clinic/tsworkshop/workshop1/lab-script.sh
```

#### b. Typing the option “5” (Demo Break Online Boutique - Kibana) and press “Enter”

#### c. Type "99" and press “Enter” to exit

#### d. Opening the Online Boutique with the URL shown through the command below and Trying to access the Online Boutique we get the error: *“504 Gateway Time-out”*

```bash
echo https://hipstershop.$(kubectl cluster-info | grep -i control | awk -F "://" '{print $2}' | cut -d. -f1).lynx.tigera.ca
```

#### e. Open the Calico Cloud UI and go to Kibana by clicking in the icon <img src="Images/icon-4.png" alt="Kibana" width="30">

#### f. From Kibana, there are multiple ways to track the flow and we will use a predefined dashboard hence go to Home > Analytics > Dashboard

<p align="center">
  <img src="Images/24.m2lab5-1.png" alt="Kibana - Dashboard" align="center" width="800">
</p>

#### g. Clicking on “Tigera Secure EE Flow Logs” dashboard

<p align="center">
  <img src="Images/25.m2lab5-2.png" alt="Kibana - Tigera EE Flow logs" align="center" width="800">
</p>

#### h. Add the filters *“action"* is *"deny"*, *"dest_namespace"* is *"hipstershop"*, *"_index"* is *"\*<name_of_cluster>\** (as per the screenshot below). Kibana will show the logs of all clusters connected then if you want to see logs from a specific cluster, it is needed to filter by _index field. 

<p align="center">
  <img src="Images/26.m2lab5-3.png" alt="Kibana - Flow Logs filter" align="center" width="800">
</p>

#### i. We can see the denied flows. 

<p align="center">
  <img src="Images/27.m2lab5-4.png" alt="Kibana - Denied Flows" align="center" width="800">
</p>

#### j. Expanding the flow logs to frontend service as it is not accessible, it shows the Security Policy *"0|security|security.tenant-histershop|deny|0"* has denied this flow.

<p align="center">
  <img src="Images/28.m2lab5-5.png" alt="Kibana - Denied Policy" align="center" width="800">
</p>

#### k. When checking on the Security Policy “tenant-hipstershop”, we can see the Ingress rule set to Deny and it should be Pass as configured in the topic **[Configure the Security and DNS Policies in the cluster](https://github.com/tigera-cs/observability-clinic/blob/main/1.%20Overview/readme.md#6-configure-the-security-and-dns-policies-in-the-cluster)**.

<p align="center">
  <img src="Images/29.m2lab5-6.png" alt="Security Policy - Denied Policy" align="center" width="800">
</p>

#### i. After changing the Ingress rule to ***Pass*** in the “tenant-hipstershop” Security Policy, the Online Boutique should be accessible again.

### LAB

#### a. Open the lab and run the script below:

```bash
/home/tigera/observability-clinic/tsworkshop/workshop1/lab-script.sh
```

#### b. Type the option “6” (LAB Break Online Boutique - Kibana) and press “Enter”

#### c. Type "99" and press “Enter” to exit the script

#### d. Open the browser with the URL shown through the command below:

```bash
echo https://hipstershop.$(kubectl cluster-info | grep -i control | awk -F "://" '{print $2}' | cut -d. -f1).lynx.tigera.ca
```

#### e. Select any product(s), add to the Cart and click on “Place Order”.

#### f. An internal server error will return saying “failed to get user cart during checkout”

```nano
HTTP Status: 500 Internal Server Error
rpc error: code = Internal desc = cart failure: failed to get user cart during checkout: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing dial tcp 10.49.114.104:7070: i/o timeout"
failed to complete the order
main.(*frontendServer).placeOrderHandler
```

<p align="center">
  <img src="Images/30.m2lab6-1.png" alt="Online Boutique - 500 Internal Server Error" align="center" width="800">
</p>

#### g. Investigate through the Kibana which flows have been denied and the Security Policy related to it, and how to fix this issue. Use the hipstershop application information provided in the **[Module 1 - Topic 4](https://github.com/tigera-cs/observability-clinic/blob/main/1.%20Overview/readme.md#4-install-hipstershop-application)**.

#### h. To revert back the misconfiguration applied, run the script and type “61” (LAB Fix Online Boutique - Kibana) and press Enter. To exit type "99" and press “Enter”.



> ## You have completed `4.Secure Workload Egress Access Using Calico DNS Policy` lab. [Next: Lesson 5.Secure Kubernetes Network Using Wireguard Encryption](https://github.com/tigera-cs/quickstart-self-service/blob/main/modules/analyze-networksets-external-services.md) 