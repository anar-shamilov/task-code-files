### We can use Kubernetes NetworkPolicy to restrict access from specific pods in EKS cluster to the databases. 

#### Here’s how you can create such a policy:

```bash
$ kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-db-access
  namespace: your-namespace
spec:
  podSelector:
    matchLabels:
      access: db
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 10.100.100.100/32
      ports:
      - protocol: TCP
        port: 5432
    - ipBlock:
        cidr: 10.100.100.101/32
      ports:
      - protocol: TCP
        port: 27017
EOF
```

#### Ensure the pods that need database access have the appropriate label:

```bash
$ kubectl label pod <pod-name> access=db -n your-namespace
```

Explanation:
- `podSelector`: Selects pods with the label `access: db` in the specified namespace.
- `egress`: Allows outbound traffic to the specified IP addresses and ports.

This setup ensures that only labeled pods can access the specified database IPs and ports, meeting the requirement to restrict database access.

`Note`: We can use Istio at the same time to control traffic to outside from the Mesh too. Du to that we must register DNS names for mongo and postgresql with the Kind `ServiceEntry`. Then control traffic from out namespace to Gateway of the istio with the `AuthorizationPolicy` and `Sidecar` kinds.
