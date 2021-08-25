# Floating interdomain kernel2vxlan2kernel example

This example shows that NSC can reach NSE registered in floating registry.

NSC and NSE are using the `kernel` mechanism to connect to its local forwarder.
Forwarders are using the `vxlan` mechanism to connect with each other.

NSE is registering in the floating registry.

## Run

**1. Prepare cluster2**

Switch to *cluster2*:

```bash
export KUBECONFIG=$KUBECONFIG2
```

Create test namespace:
```bash
NAMESPACE1=($(kubectl create -f https://raw.githubusercontent.com/networkservicemesh/deployments-k8s/main/examples/floating/usecases/namespace.yaml)[0])
NAMESPACE1=${NAMESPACE1:10}
```

Register namespace in `spire` server:
```bash
kubectl exec -n spire spire-server-0 -- \
/opt/spire/bin/spire-server entry create \
-spiffeID spiffe://example.org/ns/${NAMESPACE1}/sa/default \
-parentID spiffe://example.org/ns/spire/sa/spire-agent \
-selector k8s:ns:${NAMESPACE1} \
-selector k8s:sa:default
```

Create kustomization file:
```bash
cat > kustomization.yaml <<EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: ${NAMESPACE1}

bases:
- github.com/networkservicemesh/deployments-k8s/apps/nse-kernel

patchesStrategicMerge:
- patch-nse.yaml
EOF
```

Create NSE patch:
```bash
cat > patch-nse.yaml <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nse-kernel
spec:
  template:
    metadata:
      annotations:
        registration-name: icmp-server@my.cluster3
    spec:
      containers:
        - name: nse
          env:
          - name: NSM_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.annotations['registration-name']
          - name: NSM_CIDR_PREFIX
            value: 172.16.1.2/31
          - name: NSM_SERVICE_NAMES
            value: icmp-responder@my.cluster3
EOF
```

Deploy NSE:
```bash
kubectl apply -k .
```

Find NSE pod by labels:
```bash
NSE=$(kubectl get pods -l app=nse-kernel -n ${NAMESPACE1} --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
```

**2. Prepare cluster1**

Switch to *cluster1*:

```bash
export KUBECONFIG=$KUBECONFIG1
```

Create test namespace:
```bash
NAMESPACE2=($(kubectl create -f https://raw.githubusercontent.com/networkservicemesh/deployments-k8s/main/examples/floating/usecases/namespace.yaml)[0])
NAMESPACE2=${NAMESPACE2:10}
```

Register namespace in `spire` server:
```bash
kubectl exec -n spire spire-server-0 -- \
/opt/spire/bin/spire-server entry create \
-spiffeID spiffe://example.org/ns/${NAMESPACE2}/sa/default \
-parentID spiffe://example.org/ns/spire/sa/spire-agent \
-selector k8s:ns:${NAMESPACE2} \
-selector k8s:sa:default
```

Create kustomization file:
```bash
cat > kustomization.yaml <<EOF
---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: ${NAMESPACE2}

bases:
- github.com/networkservicemesh/deployments-k8s/apps/nsc-kernel

patchesStrategicMerge:
- patch-nsc.yaml
EOF
```

Create NSC patch:
```bash
cat > patch-nsc.yaml <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nsc-kernel
spec:
  template:
    spec:
      containers:
        - name: nsc
          env:
            - name: NSM_NETWORK_SERVICES
              value: kernel://icmp-responder@my.cluster3/nsm-1
EOF
``````

Deploy NSC:
```bash
kubectl apply -k .
```

Wait for applications ready:
```bash
kubectl wait --for=condition=ready --timeout=5m pod -l app=nsc-kernel -n ${NAMESPACE2}
```


Find NSC pod by labels:
```bash
NSC=$(kubectl get pods -l app=nsc-kernel -n ${NAMESPACE2} --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
```


**3. Ping from NSC to NSE:**

Switch to *cluster1*:

```bash
export KUBECONFIG=$KUBECONFIG1
```

```bash
kubectl exec ${NSC} -n ${NAMESPACE2} -- ping -c 4 172.16.1.2
```

Switch to *cluster2*:

```bash
export KUBECONFIG=$KUBECONFIG2
```

Ping from NSE to NSC:
```bash
kubectl exec ${NSE} -n ${NAMESPACE1} -- ping -c 4 172.16.1.3
```

## Cleanup

1. Cleanup resources for *cluster2*:
```bash
export KUBECONFIG=$KUBECONFIG2
```
```bash
kubectl delete ns ${NAMESPACE1}
```

2. Cleanup resources for *cluster1*:
```bash
export KUBECONFIG=$KUBECONFIG1
```
```bash
kubectl delete ns ${NAMESPACE2}
```
