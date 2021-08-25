# Spire

## Run

Prepare cert and key for the spire:

```bash
kubectl create ns spire
```

```bash
openssl req -x509 -newkey rsa:4096 -keyout "bootstrap.key" -out "bootstrap.crt" -days 365 -nodes -subj '/CN=localhost' 2>/dev/null
```

```bash
kubectl create secret generic spire-secret --from-file=bootstrap.crt --from-file=bootstrap.key -n spire
```


To apply spire deployments following the next command:
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

Delete ns:
```bash
kubectl delete ns spire
```
