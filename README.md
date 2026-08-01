# Kubernetes Microservices Deployment on Minikube

This submission deploys four lightweight Node.js microservices on Kubernetes:

| Service | Container port | Kubernetes Service | Purpose |
|---|---:|---|---|
| User Service | 3000 | `user-service` | Returns sample users |
| Product Service | 3001 | `product-service` | Returns sample products |
| Order Service | 3002 | `order-service` | Calls User and Product services and builds an order |
| Gateway Service | 3003 | `gateway-service` | Proxies API paths and provides an aggregate endpoint |

All Services use `ClusterIP`, so Kubernetes DNS provides cluster-level discovery. For example, the Order Service calls `http://user-service:3000` and `http://product-service:3001`.

> **Container images:** The manifests use the public official image `node:22-alpine`. The application code is supplied inline through each Deployment's `node -e` command, making this package runnable without a private registry or separate source-code repository.

## 1. Prerequisites

Install:

- Minikube
- kubectl
- A Minikube-supported driver such as Docker
- curl for host-side testing

Check the tools:

```bash
minikube version
kubectl version --client
```

## 2. Start Minikube

Recommended local profile:

```bash
minikube start --driver=docker --cpus=4 --memory=4096
kubectl cluster-info
minikube status
```

Create the namespace used by all manifests:

```bash
kubectl create namespace microservices --dry-run=client -o yaml | kubectl apply -f -
```

## 3. Deploy the application

Run these commands from the directory that contains `submission/`:

```bash
kubectl apply -f submission/services/
kubectl apply -f submission/deployments/
```

Wait until all Deployments are available:

```bash
kubectl rollout status deployment/user-service -n microservices
kubectl rollout status deployment/product-service -n microservices
kubectl rollout status deployment/order-service -n microservices
kubectl rollout status deployment/gateway-service -n microservices
```

Inspect resources:

```bash
kubectl get deployments,pods,services -n microservices -o wide
kubectl get endpoints -n microservices
```

Expected pod state:

```text
NAME                               READY   STATUS    RESTARTS
user-service-xxxxxxxxxx-xxxxx      1/1     Running   0
product-service-xxxxxxxxxx-xxxxx   1/1     Running   0
order-service-xxxxxxxxxx-xxxxx     1/1     Running   0
gateway-service-xxxxxxxxxx-xxxxx   1/1     Running   0
```

## 4. Test with port-forward

Forward the Gateway Service to host port 8080:

```bash
kubectl port-forward -n microservices service/gateway-service 8080:3003
```

In another terminal:

```bash
curl -s http://localhost:8080/ | python -m json.tool
curl -s http://localhost:8080/api/users | python -m json.tool
curl -s http://localhost:8080/api/products | python -m json.tool
curl -s http://localhost:8080/api/orders | python -m json.tool
curl -s http://localhost:8080/aggregate | python -m json.tool
```

The `/api/orders` response proves this communication path:

```text
Gateway Service -> Order Service -> User Service + Product Service
```

The `/aggregate` endpoint calls all three backend services.

## 5. Validate service-to-service DNS directly

Run a Node.js fetch command inside the Gateway pod. This uses Kubernetes Service names, not localhost:

```bash
kubectl exec -n microservices deployment/gateway-service -- \
  node -e "fetch('http://user-service:3000/api/users').then(r=>r.text()).then(console.log)"

kubectl exec -n microservices deployment/gateway-service -- \
  node -e "fetch('http://product-service:3001/api/products').then(r=>r.text()).then(console.log)"

kubectl exec -n microservices deployment/gateway-service -- \
  node -e "fetch('http://order-service:3002/api/orders').then(r=>r.text()).then(console.log)"
```

Check DNS resolution:

```bash
kubectl exec -n microservices deployment/gateway-service -- \
  node -e "require('dns').lookup('user-service',(e,a)=>console.log(e||a))"
```

## 6. View logs

Generate traffic first, then review logs:

```bash
curl -s http://localhost:8080/api/orders > /dev/null
curl -s http://localhost:8080/aggregate > /dev/null

kubectl logs -n microservices deployment/gateway-service --tail=50
kubectl logs -n microservices deployment/order-service --tail=50
kubectl logs -n microservices deployment/user-service --tail=20
kubectl logs -n microservices deployment/product-service --tail=20
```

Expected Order Service log pattern:

```text
GET /api/orders
Calling http://user-service:3000/api/users and http://product-service:3001/api/products
```

Expected Gateway log pattern:

```text
Proxying GET /api/orders -> http://order-service:3002/api/orders
```

## 7. Bonus: Minikube Ingress

Enable the Minikube ingress addon:

```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```

Apply the Ingress:

```bash
kubectl apply -f submission/ingress/ingress.yaml
kubectl get ingress -n microservices
```

Map the hostname to the Minikube IP.

Linux/macOS:

```bash
echo "$(minikube ip) microservices.local" | sudo tee -a /etc/hosts
```

Windows PowerShell as Administrator:

```powershell
$ip = minikube ip
Add-Content -Path C:\Windows\System32\drivers\etc\hosts -Value "$ip microservices.local"
```

Test the routes:

```bash
curl -s http://microservices.local/api/users
curl -s http://microservices.local/api/products
curl -s http://microservices.local/api/orders
curl -s http://microservices.local/
```

If the Docker driver does not expose the Ingress IP directly on the host, run this in a separate privileged terminal:

```bash
minikube tunnel
```

Then retry the Ingress requests.

## 8. Probes, resources, and security

Every Deployment includes:

- HTTP liveness probe at `/health`
- HTTP readiness probe at `/ready`
- CPU and memory requests and limits
- Non-root container execution
- Read-only root filesystem
- Dropped Linux capabilities
- Consistent labels and selectors

The Order and Gateway readiness endpoints verify their downstream services. Kubernetes will not route Service traffic to a pod until its readiness probe succeeds.

## 9. Capture the required screenshots

The PNG files included in `screenshots/` are clearly marked placeholders because real evidence must be captured from the student's local Minikube cluster. Replace them before final submission.

### `screenshots/pods.png`

Run:

```bash
kubectl get pods -n microservices -o wide
```

Capture the terminal showing all four pods in `Running` and `1/1 Ready` state.

### `screenshots/logs.png`

Run:

```bash
curl -s http://localhost:8080/api/orders > /dev/null
kubectl logs -n microservices deployment/order-service --tail=30
kubectl logs -n microservices deployment/gateway-service --tail=30
```

Capture logs showing the Gateway forwarding a request and the Order Service calling User and Product services.

### `screenshots/service-test.png`

Run:

```bash
curl -i http://localhost:8080/api/orders
```

Capture the HTTP 200 response and JSON output.

## 10. Troubleshooting

### Pods remain Pending

```bash
kubectl describe pod -n microservices <pod-name>
kubectl get events -n microservices --sort-by=.lastTimestamp
minikube status
```

Increase Minikube resources if necessary:

```bash
minikube stop
minikube start --driver=docker --cpus=4 --memory=4096
```

### ImagePullBackOff

```bash
kubectl describe pod -n microservices <pod-name>
minikube ssh -- docker pull node:22-alpine
```

Confirm that the machine has internet access to pull the public Node image.

### Readiness probe fails

```bash
kubectl describe pod -n microservices <pod-name>
kubectl logs -n microservices <pod-name>
kubectl get services,endpoints -n microservices
```

For the Order and Gateway pods, verify that all backend Service endpoints exist.

### Service name does not resolve

```bash
kubectl exec -n microservices deployment/gateway-service -- \
  node -e "require('dns').lookup('user-service',(e,a)=>console.log(e||a))"

kubectl get svc -n microservices
kubectl get endpoints -n microservices
```

Ensure the caller and Services are in the same namespace. A cross-namespace DNS name would be `user-service.microservices.svc.cluster.local`.

### Port-forward exits or port 8080 is busy

```bash
kubectl port-forward -n microservices service/gateway-service 18080:3003
curl http://localhost:18080/
```

### Ingress returns 404

```bash
minikube addons list | grep ingress
kubectl get pods -n ingress-nginx
kubectl describe ingress microservices-ingress -n microservices
kubectl get endpoints -n microservices
```

Confirm that `microservices.local` resolves to `minikube ip` and that the Host header is present:

```bash
curl -H 'Host: microservices.local' http://$(minikube ip)/api/users
```

## 11. Remove the deployment

```bash
kubectl delete -f submission/ingress/ --ignore-not-found
kubectl delete -f submission/deployments/
kubectl delete -f submission/services/
kubectl delete namespace microservices
minikube stop
```

## References

- Minikube start guide: https://minikube.sigs.k8s.io/docs/start/
- Kubernetes probes: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/
- Kubernetes Services: https://kubernetes.io/docs/concepts/services-networking/service/
- Kubernetes Ingress: https://kubernetes.io/docs/concepts/services-networking/ingress/
