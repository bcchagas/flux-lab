# flux-lab

### Prerequisite

- kubectl
- kind
- helm
- git

### Setting up KinD

Create the Kubernetes cluster 

```yaml
# Create the KinD Kubernetes cluster
kind create cluster --name flux-dev --config=kind.yaml

# Switch the context to the cluster created
kubectl cluster-info --context kind-flux-dev

# Install the Nginx Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Wait until the Ingress Controller is ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
```

### Ingress Controller 

Install Nginx Ingress on the cluster


### Flux


```

```
