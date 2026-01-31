# Cilium Network Policy Example

This example demonstrates Cilium's L3/L4 network security capabilities using a 3-tier application architecture.

## Architecture

```
┌──────────┐
│ Internet │
└─────┬────┘
      │ (allowed)
      ▼
┌────────────┐
│  Frontend  │ (nginx)
└─────┬──────┘
      │ (port 8080)
      ▼
┌────────────┐
│  Backend   │ (API)
└─────┬──────┘
      │ (port 5432)
      ▼
┌────────────┐
│  Database  │ (PostgreSQL)
└────────────┘
```

## What This Example Demonstrates

1. **Default Deny Policy**: All traffic is denied by default
2. **Tier-to-Tier Communication**: Frontend can only talk to Backend, Backend can only talk to Database
3. **Port-Level Security**: Each tier only allows specific ports
4. **External Access Control**: Only Frontend is accessible from the internet

## Deploying the Example

### Prerequisites

- Kubernetes cluster with Cilium installed
- `kubectl` configured to access your cluster

### Deploy the Application

```bash
# Deploy the 3-tier application
kubectl apply -f deployment.yaml

# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l tier=frontend --timeout=120s
kubectl wait --for=condition=ready pod -l tier=backend --timeout=120s
kubectl wait --for=condition=ready pod -l tier=database --timeout=120s
```

### Apply Network Policies

```bash
# Apply Cilium Network Policies
kubectl apply -f network-policy.yaml

# Verify policies are applied
kubectl get cnp
```

## Testing the Policies

### Test 1: Frontend Can Access Backend ✅

```bash
# Get a frontend pod
FRONTEND_POD=$(kubectl get pod -l app=frontend -o jsonpath='{.items[0].metadata.name}')

# Try to access backend (should succeed)
kubectl exec -it $FRONTEND_POD -- curl -s http://backend:8080
```

**Expected Output**: `Hello from Backend API`

### Test 2: Frontend Cannot Access Database ❌

```bash
# Try to access database directly from frontend (should fail)
kubectl exec -it $FRONTEND_POD -- nc -zv database 5432 -w 2
```

**Expected Output**: Connection timeout or refused

### Test 3: Backend Can Access Database ✅

```bash
# Get a backend pod
BACKEND_POD=$(kubectl get pod -l app=backend -o jsonpath='{.items[0].metadata.name}')

# Try to access database (should succeed)
kubectl exec -it $BACKEND_POD -- nc -zv database 5432 -w 2
```

**Expected Output**: Connection successful

### Test 4: External Access to Frontend ✅

```bash
# Get frontend service external IP
kubectl get svc frontend

# Access from outside the cluster
curl http://<EXTERNAL-IP>
```

**Expected Output**: nginx welcome page

## Understanding the Policies

### Default Deny (`default-deny`)
- Denies all ingress and egress traffic by default
- Forces explicit allow rules for all communication

### Frontend to Backend (`allow-frontend-to-backend`)
- Allows pods labeled `app=frontend` to access pods labeled `app=backend`
- Only on port 8080/TCP
- All other ports are blocked

### Backend to Database (`allow-backend-to-database`)
- Allows pods labeled `app=backend` to access pods labeled `app=database`
- Only on port 5432/TCP (PostgreSQL)

### Frontend Ingress (`allow-frontend-ingress`)
- Allows external traffic (from `world` entity) to reach frontend
- Only on port 80/TCP

### DNS Resolution (`allow-dns`)
- Allows all pods to perform DNS lookups
- Required for service discovery

## Cleanup

```bash
# Remove network policies
kubectl delete -f network-policy.yaml

# Remove application
kubectl delete -f deployment.yaml
```

## Learn More

- [Cilium Network Policy Documentation](https://docs.cilium.io/en/stable/security/policy/)
- [Cilium L3/L4 Policy Examples](https://docs.cilium.io/en/stable/security/policy/language/#layer-3-examples)
