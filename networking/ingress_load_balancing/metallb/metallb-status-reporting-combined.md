# MetalLB Status Reporting - Complete Guide

## Understanding MetalLB Status Custom Resources

MetalLB exposes status information for several key components, providing a comprehensive view of its configuration and operation:

* **`IPAddressPool` Status**: Shows the allocation and availability of IP addresses within a defined pool.
* **`ServiceL2Status`**: Indicates the status of L2 advertisements for services.
* **`ServiceBGPStatus`**: Provides details on the status of BGP sessions for services. This is particularly important for Telco environments.

The MetalLB controller, typically deployed as `metallb-system/controller`, is responsible for managing IP address assignments and updating the `IPAddressPool` status. When a service requests a LoadBalancer IP, the controller allocates an IP from an appropriate `IPAddressPool` and updates the status fields to reflect the current number of assigned and available IP addresses.

---

## Viewing IPAddressPool

As a cluster administrator, you can add address pools to your cluster to control the IP addresses that MetalLB can assign to load-balancer services. This worked example shows how to create an `IPAddressPool` custom resource (CR) and view its status. The advertisement mode is set to L2.

### Prerequisites

* You have an OpenShift Container Platform cluster with the MetalLB Operator installed.
* You have at least one MetalLB Custom Resource (CR) instance created.

### Procedure

The `IPAddressPool` CR's status section provides details about IP address allocation from a specific pool.

#### 1. Create an IP address pool

Create a file, such as `ipaddresspool.yaml`, with content like the following example:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: doc-example-l2
  namespace: metallb-system
spec:
  addresses:
  - 192.168.122.200-192.168.122.220
  autoAssign: true
  avoidBuggyIPs: false
```

Apply the configuration for the IP address pool:

```bash
$ oc apply -f ipaddresspool.yaml
```

#### 2. View the status of the IP address pool

Run the following command:

```bash
$ oc get ipaddresspool doc-example-l2 -n metallb-system -o yaml
```

**Example output:**

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"metallb.io/v1beta1","kind":"IPAddressPool","metadata":{"annotations":{},"name":"doc-example-l2","namespace":"metallb-system"},"spec":{"addresses":["192.168.122.200-192.168.122.220"],"autoAssign":true,"avoidBuggyIPs":false}}
  creationTimestamp: "2025-07-17T10:13:37Z"
  generation: 1
  name: doc-example-l2
  namespace: metallb-system
  resourceVersion: "29080"
  uid: 8df1c303-03ac-4d31-8970-6dacdb173dc2
spec:
  addresses:
  - 192.168.122.200-192.168.122.220
  autoAssign: true
  avoidBuggyIPs: false
status:
  assignedIPv4: 0
  assignedIPv6: 0
  availableIPv4: 21
  availableIPv6: 0
```

**Status field explanations:**
- `assignedIPv4: 0` - This integer represents the total number of IPv4 addresses that have been successfully assigned by MetalLB from this pool to LoadBalancer services.
- `assignedIPv6: 0` - This integer represents the total number of IPv6 addresses that have been successfully assigned by MetalLB from this pool to LoadBalancer services.
- `availableIPv4: 21` - This integer indicates the total number of IPv4 addresses remaining and available for assignment within this pool. This value is calculated by subtracting assignedIPv4 from the total theoretical IPv4 addresses in the spec.addresses ranges, potentially accounting for avoidBuggyIPs if enabled.
- `availableIPv6: 0` - This integer indicates the total number of IPv6 addresses remaining and available for assignment within this pool. This value is calculated similarly to availableIPv4, based on the total theoretical IPv6 addresses in the spec.addresses ranges.

#### 3. Create an L2Advertisement for Layer 2 mode

Create the L2Advertisement with the following sample YAML:

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: l2advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
  - doc-example-l2
```

Apply the configuration for the L2 advertisement:

```bash
$ oc apply -f l2advertisement.yaml
```

#### 4. Deploy an application and expose it with a LoadBalancer service

Create a file, such as `nginx.yaml`, with the following content to deploy an Nginx application:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

Apply the configuration by running the following command:

```bash
$ oc apply -f nginx.yaml
```

Expose the deployment as a LoadBalancer service by creating a file, such as `nginx-service.yaml`, with content like the following:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: metallb-system # Or the namespace where your deployment is
  annotations:
    metallb.universe.tf/address-pool: doc-example-l2 # Explicitly tells MetalLB to use your specific IPAddressPool
spec:
  selector:
    app: nginx # Must match the labels of your deployment
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```

Apply the service configuration by running the following command:

```bash
$ oc apply -f nginx-service.yaml
```

#### 5. View the updated IPAddressPool status

View the updated `IPAddressPool` status to see the assigned and available IP addresses by running the following command:

```bash
$ oc get ipaddresspool doc-example-l2 -n metallb-system -o yaml
```

**Example output:**

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"metallb.io/v1beta1","kind":"IPAddressPool","metadata":{"annotations":{},"name":"doc-example-l2","namespace":"metallb-system"},"spec":{"addresses":["192.168.122.200-192.168.122.220"],"autoAssign":true,"avoidBuggyIPs":false}}
  creationTimestamp: "2025-07-17T10:13:37Z"
  generation: 1
  name: doc-example-l2
  namespace: metallb-system
  resourceVersion: "30250"
  uid: 8df1c303-03ac-4d31-8970-6dacdb173dc2
spec:
  addresses:
  - 192.168.122.200-192.168.122.220
  autoAssign: true
  avoidBuggyIPs: false
status:
  assignedIPv4: 1
  assignedIPv6: 0
  availableIPv4: 20
  availableIPv6: 0
```

**Note:** The `assignedIPv4: 1` value indicates that one IPv4 address from this pool has been successfully assigned by MetalLB to your `nginx-service` LoadBalancer. This confirms that MetalLB has allocated an IP and it is in use.

---

## Viewing ServiceL2Status

The ServiceL2Status CR provides details about which node is advertising a service using Layer 2 mode. This is useful for troubleshooting to confirm that the correct node is acting as the announcer.

### Prerequisites

* You have an OpenShift Container Platform cluster with the MetalLB Operator installed.
* You have at least one MetalLB Custom Resource (CR) instance created.

### Procedure

#### 1. View the status of the ServiceL2Status CR

Run the following command:

```bash
$ oc get ServiceL2Status -n metallb-system
```

**Example output:**

```
NAME                    LABELS                                                   AGE
nginx-service-worker0   metallb.io/node=worker0,metallb.io/service=nginx-service 5h
```

#### 2. View the details of the ServiceL2Status CR for your service

Run the following command:

```bash
$ oc get servicel2status nginx-service-worker0 -n metallb-system -o yaml
```

**Example output:**

```yaml
apiVersion: metallb.io/v1beta1
kind: ServiceL2Status
metadata:
  name: nginx-service-worker0
  namespace: metallb-system
  labels:
    metallb.io/node: worker0
    metallb.io/service: nginx-service
spec: {}
status:
    node: worker0
    interfaces:
        - eth0
```

**Status field explanations:**
- `metallb.io/node: worker0` - This label identifies the node that is advertising the service. This is useful for troubleshooting to verify which node is the announcer.
- `metallb.io/service: nginx-service` - This label identifies the service that is being advertised.
- `node: worker0` - This field confirms the name of the node advertising the service.
- `interfaces: [eth0]` - This field lists the network interfaces on the announcing node that are used for the advertisement. If the field is empty, it means all interfaces on the node are being used for the advertisement.

---

## Viewing ServiceBGPStatus

The ServiceBGPStatus CR provides insights into which BGP peers a specific service is being advertised to from a given node. This is a crucial tool for network administrators in Telco environments to verify BGP advertisement and debug connectivity.

### Prerequisites

* You have an OpenShift Container Platform cluster with the MetalLB Operator installed.
* You have at least one MetalLB Custom Resource (CR) instance created.

### Procedure

#### 1. View the status of the ServiceBGPStatus CR

Run the following command:

```bash
$ oc get ServiceBGPStatus -n metallb-system
```

**Example output:**

```
NAME                  LABELS                                                   AGE
nginx-service-worker0 metallb.io/node=worker0,metallb.io/service=nginx-service 5h
```

#### 2. View the details of the ServiceBGPStatus CR for your service

Run the following command:

```bash
$ oc get servicebgpstatus nginx-service-worker0 -n metallb-system -o yaml
```

**Example output:**

```yaml
apiVersion: metallb.io/v1beta1
kind: ServiceBGPStatus
metadata:
  name: service1-worker0
  namespace: servicenamespace
  labels:
    metallb.io/node: worker0
    metallb.io/service: service1
status:
    bgpPeers:
        - peerA
        - peerB
```

**Status field explanations:**
- `metallb.io/node: worker0` - This label indicates the node that is advertising the service via BGP.
- `metallb.io/service: service1` - This label identifies the service being advertised.
- `bgpPeers: [peerA, peerB]` - This field lists the names of the BGP peers to which the service is being advertised. This is useful for confirming that the advertisement is reaching the intended peers.
