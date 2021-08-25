## Setup spire for two clusters

This example shows how to simply configure two spire from clusters to know each other.


## Run

1. Make sure that you have two KUBECONFIG files.

Check `KUBECONFIG1` env:
```bash
[[ ! -z $KUBECONFIG1 ]]
```

Check `KUBECONFIG2` env:
```bash
[[ ! -z $KUBECONFIG2 ]]
```

2. Setup spire

Create spire namespace:
```bash
kubectl create ns spire
```

Generate cert and key:
```bash
openssl req -x509 -newkey rsa:4096 -keyout "bootstrap.key" -out "bootstrap.crt" -days 365 -nodes -subj '/CN=localhost' 2>/dev/null
```

Create secret:
```bash
kubectl create secret generic spire-secret --from-file=bootstrap.crt --from-file=bootstrap.key -n spire
```

**Apply spire resources for the first cluster:**
```bash
export KUBECONFIG=$KUBECONFIG1
```

```bash
kubectl apply -k https://github.com/glazychev-art/deployments-k8s/examples/spire?ref=kust_test
```

Wait for PODs status ready:
```bash
kubectl wait -n spire --timeout=1m --for=condition=ready pod -l app=spire-agent
```
```bash
kubectl wait -n spire --timeout=1m --for=condition=ready pod -l app=spire-server
```

Register spire agents in the spire server:
```bash
kubectl exec -n spire spire-server-0 -- \
/opt/spire/bin/spire-server entry create \
-spiffeID spiffe://example.org/ns/spire/sa/spire-agent \
-selector k8s_sat:cluster:nsm-cluster \
-selector k8s_sat:agent_ns:spire \
-selector k8s_sat:agent_sa:spire-agent \
-node
```

**Apply spire resources for the second cluster:**
```bash
export KUBECONFIG=$KUBECONFIG2
```

```bash
kubectl apply -k https://github.com/glazychev-art/deployments-k8s/examples/spire?ref=kust_test
```

Wait for PODs status ready:
```bash
kubectl wait -n spire --timeout=1m --for=condition=ready pod -l app=spire-agent
```
```bash
kubectl wait -n spire --timeout=1m --for=condition=ready pod -l app=spire-server
```

Register spire agents in the spire server:
```bash
kubectl exec -n spire spire-server-0 -- \
/opt/spire/bin/spire-server entry create \
-spiffeID spiffe://example.org/ns/spire/sa/spire-agent \
-selector k8s_sat:cluster:nsm-cluster \
-selector k8s_sat:agent_ns:spire \
-selector k8s_sat:agent_sa:spire-agent \
-node
```

## Cleanup

Cleanup spire for the first cluster:
```bash
export KUBECONFIG=$KUBECONFIG1
```

Delete ns:
```bash
kubectl delete ns spire
```

Cleanup spire for the second cluster:
```bash
export KUBECONFIG=$KUBECONFIG2
```

Delete ns:
```bash
kubectl delete ns spire
```
