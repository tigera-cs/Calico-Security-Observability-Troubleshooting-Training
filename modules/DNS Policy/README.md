### Use GlobalNetworkSet to implement egress DNS Policy

After sometime the yaobank `customer` application connectivity requirements has changed and the application needs to connect to a service outside the cluster. In this section, we will use Calico `GlobalNetworkSet` resource to implement egress access control using Calico `DNS Policy`.

Currently, `customer` application can't make any connection to outside the cluster as Calico security policy is blocking the traffic. Let's validate that by the following.

Exec into the `customer` pod.

```bash
kubectl exec -it -n yaobank $(kubectl get pods -n yaobank --show-labels | awk '{print $1}' | grep -v NAME) -- sh

```
curl into the destination domain name.

```bash
curl -vvv www.tigera.io

```

The connection should fail with a message similar to the following.

![curl-fail](img\fail-curl.png)


#### Deploy GlobalNetworkSet

Deploy the following `GlobalNetworkSet` manifest in the cluster. We will use the following `GlobalNetworkSet` to allow egress connections to `www.ubuntu.com`

```yaml
kubectl apply -f -<<EOF
kind: GlobalNetworkSet
apiVersion: projectcalico.org/v3
metadata:
  name: tigera
  labels:
    eep.tigera.io/domain: tigera
spec:
  allowedEgressDomains:
    - 'www.tigera.io'
EOF

```

#### Deploy DNS Policy

Deploy the following DNS policy referencing the previous GlobalNetworkSet to allow egress connections to `www.tigera.io`.

-------------------------------------------------------------------------------------------------
Update this policy based on the changes to the yaobank policy

```yaml
kubectl apply -f -<<EOF
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: app.customer-dns-policy
  namespace: yaobank
spec:
  tier: app
  order: 10
  selector: app == "customer"
  egress:
    - action: Allow
      protocol: TCP
      source: {}
      destination:
        selector: eep.tigera.io/domain == "tigera"
        namespaceSelector: global()
        ports:
          - '80'
          - '443'
  types:
    - Egress
EOF

```

-------------------------------------------------------------------------------------------------



Exec into the `customer` pod again.

```bash
kubectl exec -it -n yaobank $(kubectl get pods -n yaobank --show-labels | awk '{print $1}' | grep customer) -- sh

```
curl into the destination domain name.

```bash
curl -vvv www.tigera.io

```

The connection should go through this time.

![curl-successful](img\curl-successful.png)

Navigate to `Servicegraph` from Calico Manager UI and double-click on the yaobank namespace to go into the yaobank namespace in ServiceGraph. Filter the flow logs for the past 10 minutes of the traffic. See how ServiceGraph use the GlobalNetworkSet name `tigera` to correlate the connection from `customer` pod to `www.tigera.io` domain name.
Note: When filtering flow logs based on time, if you go far back in time when we tried to connect to `www.tigera.io` and the connection failed, ServiceGraph will also show the old failed connection as a red line (connection) in the graph.

![service-graph-ubuntu](img\service-graph-ubuntu.png)
