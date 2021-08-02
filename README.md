# flux-lab

### Prerequisite

You'll need [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/), [KinD](https://kind.sigs.k8s.io/), [flux-cli ](https://fluxcd.io/docs/installation/)and [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) to follow this tutorial.

### Introduction

In this step-by-step you'll create a local Kubernetes cluster which will host 2 applications: the [Nginx Ingress Controller](https://kubernetes.github.io/ingress-nginx/deploy/) to handle the incoming connections to the cluster and the [podinfo](https://github.com/stefanprodan/podinfo) application which is a lightweight go web application.

The cluster will be installed in a GitOps manner using FluxCD to make sure the definition of the applications in this repo match what is currently running on the cluster.

The initial repo structure is as follow:

```bash
flux-lab
└── kustomize
    ├── nginx 
    │   └── kustomization.yaml
    └── podinfo
        ├── base
        │   ├── deployment.yaml
        │   ├── hpa.yaml
        │   ├── ingress.yaml
        │   ├── kustomization.yaml
        │   ├── namespace.yaml
        │   └── service.yaml
        ├── hlg
        │   └── kustomization.yaml
        ├── kustomization.yaml
        └── prd
            └── kustomization.yaml
```

The kustomize folder contains the two applications we will be deploying: nginx and podinfo. The nginx folder contains a Kubernetes Kustomize definition that references a Kubernetes manifest hosted on Github. The podinfo also contains a Kubernetes Kustomize definition but it references local Kubernetes manifests hosted within this same repo.

We'll started by creating aKubernetes cluster and deploying the applications as is. Next, we'll add Flux to watch the repository for changes and submit then to the cluster.

### Getting started

Go ahead and click on the Fork icon on the upper right corner of this repository and then clone it to your localhost:

```bash
git clone https://github.com/<your_username>/flux-lab.git
cd flux-lab
```

### Setting up KinD

Let's start by creating the Kubernetes cluster. The ` kind.yaml` definition has some configuration required by the Nginx Ingress Controller to run on KinD.

```yaml
# Create the KinD Kubernetes cluster
kind create cluster --name flux-dev --config=kind.yaml

# Switch to the context created
kubectl cluster-info --context kind-flux-dev

# Check the namespaces created
kubectl get ns
```

### Deploying the Applications

Run the commands below to the deploy the nginx ingress controller and the podinfo application

```bash
# Install nginx
kubectl apply -k kustomize/nginx

# Install pod
kubectl apply -k kustomize/podinfo

# Check the newly created namespaces
$ kubectl get ns
NAME                 STATUS   AGE
default              Active   51m
ingress-nginx        Active   14m  (new)
kube-node-lease      Active   51m
kube-public          Active   51m
kube-system          Active   51m
local-path-storage   Active   51m
podinfo-hlg          Active   3m9s (new)
podinfo-prd          Active   3m9s (new)
```

Notice that the podinfo application is deployed in two namespaces: podinfo-hlg and podinfo-prd.

### Accessing the podinfo application

You can access the applications by heading to [http://podinfo-hlg.localho.st:8080](http://podinfo-hlg.localho.st:8080) and [http://podinfo-hlg.localho.st:8080/](http://podinfo-hlg.localho.st:8080/).

We are leveraging the localho.st public domain which points to localhost (your localhost).

### Setting up Flux

Up until now we haven't been using Flux. We just spun up a KinD cluster and published some Kubernetes manifests which are organized using Kustomize (don't mistake with Flux CRD Kustomizations).

We'll use the command `flux bootstrap` to provision the Flux components. The bootstrap not only will install Flux on the cluster (same as `flux install`) but also will publish the manifests on GIthub and create the Flux custom resources definitions to watch for changes to itself, such as:

- the flux-system Kustomization
- the flux-system GitRepository
- the flux-system Kubernetes secret which contains the Github Deploy Key generated using the Personal Access Token that you will provide in the very next step.

Go ahead to Github and create a [PAT](https://docs.github.com/pt/github/authenticating-to-github/keeping-your-account-and-data-secure/creating-a-personal-access-token). After that, expose the variables bellow so that Flux can connect to Github

```bash
export GITHUB_TOKEN=<your_token_here>
export GITHUB_USER=<your-username>
```

Run the bootstrap command to install Flux on the cluster:

```bash
# Check whether Flux prerequisites are met
flux check --pre

# Install Flux components on the cluster
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=flux-lab \
  --branch=main \
  --path=./fluxcd \
  --context=kind-flux-dev \
  --personal
```

The command above will generate the Flux components in the folder `flux-system`  at the specified ` branch` and `path`. If the repo specified doesn't exist it will be created. After that, Flux will apply those manifests to the cluster which can be seen at the newly created namespace `flux-system`. It will also use the PAT you created to generate a [Deploy Key](https://docs.github.com/en/developers/overview/managing-deploy-keys) in the repo.

```bash
# Newly created files on the repo by the Flux bootstrap
flux-lab
└── fluxcd 
    └── flux-system
        ├── gotk-components.yaml # The flux itself (flux install)
        ├── gotk-sync.yaml       # The flux configuration to sync itself
        └── kustomization.yaml

# Namespaces after the bootstrap
NAME                 STATUS   AGE
default              Active   70m
flux-system          Active   33m    (new)
kube-node-lease      Active   70m
kube-public          Active   70m
kube-system          Active   70m
local-path-storage   Active   70m
```

### Configuring the Continuous Delivery

We will create a Flux [Kustomization](https://fluxcd.io/docs/components/kustomize/kustomization/) manifest for podinfo and nginx. This configures Flux to build and apply the [kustomize](https://github.com/stefanprodan/podinfo/tree/master/kustomize) directory.

**nix**

```sh
# Create folder for the nginx
mkdir -p ./fluxcd/nginx

# Create the Kustomization for the nginx
flux create kustomization nginx \
  --source=flux-system \
  --path="./kustomize/nginx" \
  --prune=true \
  --validation=client \
  --interval=1m \
  --export > ./fluxcd/nginx/nginx.yaml

# Create the Kubernetes Kustomize for the nginx Kustomization
cat <<EOF | > ./fluxcd/nginx/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - nginx.yaml
EOF
```

**podinfo**

```bash
# Create folder for the podinfo
mkdir -p ./fluxcd/podinfo

# Create the Kustomization for the podinfo
flux create kustomization podinfo \
  --source=flux-system \
  --path="./kustomize/podinfo" \
  --prune=true \
  --validation=client \
  --interval=1m \
  --export > ./fluxcd/podinfo/podinfo.yaml

# Create the Kubernetes Kustomize for the nginx Kustomization
cat <<EOF | > ./fluxcd/podinfo/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - podinfo.yaml
EOF
```

Commit and push the changes to the  repository:

```sh
git add -A && git commit -m "Add podinfo Kustomization"
git push
```

The structure of the repository should look like this:

```bash
flux-lab
├── README.md
├── fluxcd
│   ├── flux-system
│   │   ├── gotk-components.yaml
│   │   ├── gotk-sync.yaml
│   │   └── kustomization.yaml
│   ├── nginx
│   │   ├── kustomization.yaml
│   │   └── nginx.yaml
│   └── podinfo
│       ├── kustomization.yaml
│       └── podinfo.yaml
├── kind.yaml
└── kustomize
    ├── nginx
    │   └── kustomization.yaml
    └── podinfo
        ├── base
        │   ├── deployment.yaml
        │   ├── hpa.yaml
        │   ├── ingress.yaml
        │   ├── kustomization.yaml
        │   ├── namespace.yaml
        │   └── service.yaml
        ├── hlg
        │   └── kustomization.yaml
        ├── kustomization.yaml
        └── prd
            └── kustomization.yaml
```

Check the Flux custom resource definitions you just created wit

```bash
# Listing resources with kubectl
kubectl get kustomization -n flux-system
kubectl get gitrepositories -n flux-system

# Listing resources with flux-cli
flux get kustomizations
flux get source git

# Getting detailed information
kubectl describe kustomization nginx -n flux-system
kubectl describe kustomization podinfo -n flux-system
kubectl describe gitrepositories flux-system -n flux-system
```

### Clean up

To uninstall Flux run

```bash
flux uninstall
```

Delete the PAT you created. It will automatically delete the Deploy Key it generated for the cluster (Github will tell it as a warning message)

You can as easily reinstall Flux running the bootstrap command again. Give it a try! 
