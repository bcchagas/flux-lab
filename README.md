# flux-lab

### Prerequisite

You'll need [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/), [KinD](https://kind.sigs.k8s.io/), [flux-cli ](https://fluxcd.io/docs/installation/)and [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) to follow this tutorial.

### Setting up your repo on Github

Go ahead and fork this repo so that you can make changes to it. You'll also need a Personal Access Token as explained [in this article](https://docs.github.com/en/github/authenticating-to-github/keeping-your-account-and-data-secure/creating-a-personal-access-token#creating-a-token).

We'll start by creating a folder to hold the PAT you just created.

```bash
# Clone the repo you've just created
# Replace <username> with your username
git clone https://github.com/<username>/flux-lab.git
cd flux-lab

# Create the folder .local
# This folder is listed in the .gitignore folder 
# so it'll be ignored by git
mkdir .local

# Create the github_cred file and add your github info
# Replace <token> and <username>
cat <<EOF > .local/github_cred
export GITHUB_TOKEN=<token>
export GITHUB_USER=<username>
EOF
```

### Setting up KinD

Create the Kubernetes cluster 

```yaml
# Create the KinD Kubernetes cluster
kind create cluster --name flux-dev --config=kind.yaml

# Switch to the context created
kubectl cluster-info --context kind-flux-dev
```

Check that the prerequisites are met

```bash
flux check --pre
```

### Setting up Flux on the cluster

Run the bootstrap command to install Flux on the cluster:

```bash
# Expose the Github credentials
source .local/github_cred

# Install Flux components on the cluster
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=flux-lab \
  --branch=main \
  --path=./clusters/flux-lab \
  --context=kind-flux-dev \
  --personal
```

The command above will generate the Flux components in the folder `flux-system`  at the specified ` branch` and `path`. After that, it will apply those manifests to the cluster which can be seen at the newly created namespace `flux-system`.

> #### Idempotency
>
> It is safe to run the bootstrap command as many times as you want. If the Flux components are present on the cluster, the bootstrap command will perform an upgrade if needed. You can target a specific Flux [version](https://github.com/fluxcd/flux2/releases) with `flux bootstrap --version=<semver>`.

```bash
# Repo before the bootstrap
├── README.md
├── kind.yaml
└── kustomize
    ├── podinfo
    │   ├── base
    │   │   ├── deployment.yaml
    │   │   ├── hpa.yaml
    │   │   ├── kustomization.yaml
    │   │   ├── namespace.yaml
    │   │   ├── ingress.yaml
    │   │   └── service.yaml
    │   ├── hlg
    │   │   └── kustomization.yaml
    │   └── prd
    │       └── kustomization.yaml
    └── nginx
        └── kustomization.yaml

# Repo after the bootstrap
├── README.md
├── kind.yaml
└── kustomize
│   ├── podinfo
│   │   ├── base
│   │   │   ├── deployment.yaml
│   │   │   ├── hpa.yaml
│   │   │   ├── kustomization.yaml
│   │   │   ├── namespace.yaml
│   │   │   ├── ingress.yaml
│   │   │   └── service.yaml
│   │   ├── hlg
│   │   │   └── kustomization.yaml
│   │   └── prd
│   │       └── kustomization.yaml
│   └── nginx
│        └── kustomization.yaml
└── clusters                         (new)
    └── flux-lab                     (new)
        └── flux-system              (new)
            ├── gotk-components.yaml (new)
            ├── gotk-sync.yaml       (new)
            └── kustomization.yaml   (new)

# Cluster (namespaces) before the bootstrap
NAME                 STATUS   AGE
default              Active   70m
ingress-nginx        Active   69m
kube-node-lease      Active   70m
kube-public          Active   70m
kube-system          Active   70m
local-path-storage   Active   70m

# Cluster (namespaces) after the bootstrap
NAME                 STATUS   AGE
default              Active   70m
flux-system          Active   33m    (new)
ingress-nginx        Active   69m
kube-node-lease      Active   70m
kube-public          Active   70m
kube-system          Active   70m
local-path-storage   Active   70m
```

### Create the Podinfo Kustomization

We will create a Flux [Kustomization](https://fluxcd.io/docs/components/kustomize/kustomization/) manifest for podinfo. This configures Flux to build and apply the [kustomize](https://github.com/stefanprodan/podinfo/tree/master/kustomize) directory located in the podinfo repository.

```sh
# Create folders for the prd and hlg kustomization
mkdir -p ./clusters/flux-lab/podinfo/{prd,hlg}

# Create the kustomization for the podinfo prd
flux create kustomization podinfo-prd \
  --source=flux-system \
  --path="./kustomize/podinfo/prd" \
  --prune=true \
  --validation=client \
  --interval=5m \
  --export > ./clusters/flux-lab/podinfo/prd/kustomization.yaml
  
# Create the kustomization for the podinfo hlg
flux create kustomization podinfo-hlg \
  --source=flux-system \
  --path="./kustomize/podinfo/hlg" \
  --prune=true \
  --validation=client \
  --interval=5m \
  --export > ./clusters/flux-lab/podinfo/hlg/kustomization.yaml  
```

Commit and push the `Kustomization` manifest to the repository:

```sh
git add -A && git commit -m "Add podinfo Kustomization"
git push
```

The structure of the repository should look like this:

```sh
├── README.md
├── clusters
│   └── flux-lab
│       ├── flux-system
│       │   ├── gotk-components.yaml
│       │   ├── gotk-sync.yaml
│       │   └── kustomization.yaml
│       ├── podinfo-kustomization.yaml
│       └── podinfo-source.yaml
└── kind.yaml
```

