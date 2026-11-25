# Kubernetes Deployment (Minikube)

Three-tier chat app on a local Minikube cluster: React + nginx frontend, Node/Express + Socket.io backend, MongoDB with persistent storage.

## Architecture

```
Browser -> Ingress (NGINX) -> Service frontend:80 -> nginx pods
                                   | /api, /socket.io
                              Service backend:5001 -> Node pod
                                   |
                              Service mongo:27017 -> Mongo pod -> PVC -> PV
```

| File                                          | Purpose                                                                |
| --------------------------------------------- | ---------------------------------------------------------------------- |
| namespace.yml                                 | `chat-app` namespace                                                   |
| mongodb-pv.yml, mongodb-pvc.yml               | 2Gi hostPath volume and claim for Mongo data                           |
| secret.example.yml                            | Template for the Secret. Copy to `secret.yml` (git-ignored)            |
| configmap.yml                                 | Non-secret settings (`PORT`, `NODE_ENV`)                               |
| mongodb-deployment.yml, mongodb-service.yml   | Mongo, 1 replica, `Recreate` strategy                                  |
| backend-deployment.yml, backend-service.yml   | API and Socket.io, `/health` probes                                    |
| frontend-deployment.yml, frontend-service.yml | nginx serving the build, proxying `/api` and `/socket.io` to `backend` |
| ingress.yml                                   | Routes all traffic to the frontend, long timeouts for WebSockets       |
| hpa.yml                                       | Frontend autoscaling, 1 to 4 replicas at 50% CPU                       |

## Prerequisites

Docker, kubectl, Minikube. Images are on Docker Hub: `dawoodashraf/chatapp-backend:1.0` and `dawoodashraf/chatapp-frontend:1.0`.

## Deploy

```powershell
minikube start --driver=docker
minikube addons enable ingress
minikube addons enable metrics-server

cp k8s/secret.example.yml k8s/secret.yml   # then replace every CHANGE_ME
# MONGODB_URI password must match MONGO_INITDB_ROOT_PASSWORD

kubectl apply -f k8s/namespace.yml
kubectl apply -f k8s/mongodb-pv.yml
kubectl apply -f k8s/mongodb-pvc.yml
kubectl apply -f k8s/secret.yml
kubectl apply -f k8s/configmap.yml
kubectl apply -f k8s/mongodb-deployment.yml
kubectl apply -f k8s/mongodb-service.yml
kubectl apply -f k8s/backend-deployment.yml
kubectl apply -f k8s/backend-service.yml
kubectl apply -f k8s/frontend-deployment.yml
kubectl apply -f k8s/frontend-service.yml
kubectl apply -f k8s/ingress.yml
kubectl apply -f k8s/hpa.yml
```

Order matters: the `backend` Service must exist before the frontend starts, because nginx resolves that name at boot.

## Access

```powershell
minikube tunnel
```

Open http://127.0.0.1 (keep the tunnel terminal open).

## Verify

```powershell
kubectl get all,ingress,hpa,pvc -n chat-app
kubectl logs deploy/backend -n chat-app
```

## Notes

- Cloudinary keys are optional. Without them, text chat works and image upload fails.
- The HPA targets the frontend only. The backend keeps online users in memory, so more than one replica would split Socket.io state.
- Mongo data lives inside the Minikube node. It survives pod deletion and `minikube stop`, not `minikube delete`.
- Mongo probes use `timeoutSeconds: 10`. The default of 1 second kills a healthy `mongosh` check.

## Cleanup

```powershell
kubectl delete namespace chat-app
kubectl delete pv mongodb-pv
```
