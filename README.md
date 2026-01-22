# MongoDB + Mongo-Express Kubernetes Deployments

This README documents the Kubernetes deployment and service manifests for a MongoDB instance and a Mongo Express web UI located in this folder. It uses only the `mongo-depl.yaml` (MongoDB) and `mongo-express.yaml` (Mongo Express) manifests.

**Files used:**
- `mongo-depl.yaml` — MongoDB Deployment, Secret, and Service
- `mongo-express.yaml` — Mongo Express Secret, ConfigMap, Deployment, and NodePort Service

## Overview
- MongoDB runs as a single-replica `Deployment` exposing port `27017` via a `Service` named `mongodb-service`.
- Mongo Express runs as a single-replica `Deployment` exposing port `8081` via a `NodePort` (nodePort `30081`) and connects to `mongodb-service`.

## Prerequisites
- A Kubernetes cluster and `kubectl` configured to access it.
- (Optional) `jq`, `curl` or other CLI tools to inspect resources.

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
# Option A: Node IP + NodePort
# http://<node-ip>:30081

# Option B: port-forward (local):
kubectl port-forward deployment/mongo-express 8081:8081
# then open http://localhost:8081
```

## Environment variables and Secrets

MongoDB root credentials are stored in a Kubernetes `Secret` named `mongodb-secret` and injected into the `mongo` container as `MONGO_INITDB_ROOT_USERNAME` and `MONGO_INITDB_ROOT_PASSWORD`.

Mongo Express basic auth and server config come from a `Secret` (`mongo-express-secret`) and a `ConfigMap` (`mongodb-basics`). Values in these resources are base64-encoded in the YAMLs.

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


