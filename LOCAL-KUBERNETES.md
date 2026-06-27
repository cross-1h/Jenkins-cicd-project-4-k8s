# Practising Kubernetes Locally (free, with minikube)

EKS costs money. Before you use it, or any time you just want to practise, run a cluster on your own machine with minikube. It is free, and the concepts and `kubectl` commands are exactly the same. The only differences are that you build the image locally instead of pulling from ECR, and you use a NodePort service instead of a load balancer.

You already installed `kubectl` (see `INSTALL-KUBERNETES.md`). Here you add minikube.

On an ARM machine, replace `amd64` with `arm64` in the Linux URL.

## 1. Install minikube

Windows (winget or Chocolatey):

```powershell
winget install -e --id Kubernetes.minikube
# or: choco install minikube -y
```

Linux:

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube version
```

macOS:

```bash
brew install minikube
```

## 2. Start the cluster

Docker must be running, since minikube uses it as the engine.

```bash
minikube start --driver=docker
kubectl get nodes
```

One node, status `Ready`, means you are good.

## 3. Build the app image and load it into the cluster

The local Dockerfile builds the JAR inside Docker, so you do not need Maven.

```bash
docker build -t cicd-app:1.0 -f app/Dockerfile.local ./app
minikube image load cicd-app:1.0
```

## 4. Deploy the local manifests

```bash
kubectl apply -f k8s/local/
kubectl get pods
kubectl get svc
```

Wait until both pods are `Running` and `Ready` (`kubectl get pods -w`).

## 5. Open the app

```bash
minikube service cicd-app --url
```

Open the URL it prints. Try `/`, `/health`, and `/add?a=2&b=3`.

## 6. Play with it

Same moves as the EKS path, all free here.

Scale:

```bash
kubectl scale deployment cicd-app --replicas=4
kubectl get pods
```

Self-heal:

```bash
kubectl delete pod <one-of-the-pod-names>
kubectl get pods -w
```

Rolling update (new version):

```bash
docker build -t cicd-app:2.0 -f app/Dockerfile.local ./app
minikube image load cicd-app:2.0
kubectl set image deployment/cicd-app cicd-app=cicd-app:2.0
kubectl rollout status deployment/cicd-app
```

## How local differs from the EKS pipeline path

| | Local (minikube) | EKS (the pipeline) |
| --- | --- | --- |
| Image source | built locally, loaded in | pulled from ECR |
| `imagePullPolicy` | `IfNotPresent` | `Always` |
| Service type | `NodePort` | `LoadBalancer` |
| Deploys by | you, with `kubectl` | the Jenkins pipeline |
| Cost | free | charged while running |

Master the commands here for free, then the EKS path is the same ideas with the pipeline doing the deploy.

## Cleanup

```bash
kubectl delete -f k8s/local/
minikube stop      # keep the cluster
minikube delete    # remove it entirely
```
