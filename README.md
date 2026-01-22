# MongoDB + Mongo-Express Kubernetes Deployments

⚠️ Security Warning: These manifests contain plain-text credentials for demonstration purposes only. While working in a real-world workflow, never commit unencrypted secrets to a production repository, use Sealed Secrets or the External Secrets Operator.

This documentation provides the optimized workflow for deploying MongoDB and Mongo Express on a Minikube cluster using the Docker driver.
Environment Criteria:
Cluster: Minikube (v1.33+)
Driver: Docker
OS: macOS / Linux / Windows

**Files used:**
- `mongo-depl.yaml` — MongoDB Deployment, Secret, and Service
- `mongo-express.yaml` — Mongo Express Secret, ConfigMap, Deployment, and NodePort Service

## Overview
- MongoDB runs as a single-replica `Deployment` exposing port `27017` via a `Service` named `mongodb-service`.
- Mongo Express runs as a single-replica `Deployment` exposing port `8081` via a `NodePort` (nodePort `30081`) and connects to `mongodb-service`.

## Prerequisites
- A Kubernetes cluster and `kubectl` configured to access it. I suggest Minikube.
- (Optional) `jq`, `curl` or other CLI tools to inspect resources.

## Start Cluster
Ensure Minikube is running with the Docker driver to handle internal DNS and Service routing correctly:
```bash
minikube start --driver=docker
```

## Deploy
Apply the MongoDB manifest first, then the Mongo Express manifest:

```bash
kubectl apply -f mongo-depl.yaml
kubectl apply -f mongo-express.yaml
```

## Verify

```bash
kubectl get pods -l app=mongo-depl
kubectl get pods -l app=mongo-express
kubectl get svc mongodb-service
kubectl get svc mongo-express-service
```

To see logs for a pod:

```bash
kubectl logs -l app=mongo-express
kubectl logs -l app=mongo-depl
```

## Accessing Mongo Express
- The `mongo-express-service` is a `NodePort` with `nodePort: 30081` and forwards to container port `8081`.
- Access the web UI at `http://<node-ip>:30081` (replace `<node-ip>` with your cluster node's IP or use a port-forward):

```bash
# Option A: Minikube critical (recommended)
# This command creates a temporary tunnel and opens your browser to the correct IP:
minikube service mongo-express-service

# Option B: Get the Minikube IP
minikube ip
# Get the assigned NodePort
kubectl get svc mongo-express-service
# Access at http://<minikube-ip>:30081


# Option C: Node IP + NodePort
# http://<node-ip>:30081

# Option D: port-forward (local):
kubectl port-forward deployment/mongo-express 8081:8081
# then open http://localhost:8081
```
## Mongo-Express UI authentication
Per secrets configured for mongo-express, make sure to use `user` as username and  `passer` as password when you get into the UI. You should see the following screen after you authenticate:
<img width="2612" height="900" alt="image" src="https://github.com/user-attachments/assets/d3853458-6fcb-4476-8e4b-fb3cda4f0bfd" />

## Environment variables and Secrets

MongoDB root credentials are stored in a Kubernetes `Secret` named `mongodb-secret` and injected into the `mongo` container as `MONGO_INITDB_ROOT_USERNAME` and `MONGO_INITDB_ROOT_PASSWORD`.

Mongo Express basic auth come from a `Secret` (`mongo-express-secret`) and non-sensitive logic come from a `ConfigMap` (`mongodb-basics`). Values in these resources are base64-encoded in the YAMLs.

To decode a base64 value from the YAML (example):

```bash
echo 'cGFzc3dvcmQ=' | base64 --decode   # may be 'base64 -D' on macOS
```

## Persistent storage
These manifests do not define a `PersistentVolumeClaim` for MongoDB — the `mongo` container will use ephemeral storage. For production use, add a `PersistentVolumeClaim` and mount it at `/data/db` in the `mongo` container.

## Troubleshooting
- If Mongo Express shows connection errors, ensure `mongodb-service` exists and `mongo-depl` pods are `Running`.
- Check the decoded credentials and that `ME_CONFIG_MONGODB_SERVER` in the ConfigMap (set to `mongodb-service`) matches the MongoDB service name.
- Use `kubectl describe pod <pod>` to surface events, and `kubectl logs` for container logs.

## Cleanup

```bash
kubectl delete -f mongo-express.yaml
kubectl delete -f mongo-depl.yaml
```

## Next steps / Recommendations
- Add a `PersistentVolumeClaim` for MongoDB data durability.
- Consider using a specific image tag (not `latest`) for reproducible deployments.
- Add readiness/liveness probes for both containers.


