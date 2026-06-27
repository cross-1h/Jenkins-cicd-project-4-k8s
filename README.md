# Project 4: Continuous Delivery to Kubernetes (Amazon EKS)

This continues Project 3. There, the pipeline pushed your image to ECR and you ran a single container by hand. That is fine for one copy, but it does not restart a crashed app, run several copies, or update with no downtime. Kubernetes does all of that, and in this project your Jenkins pipeline deploys to a real Kubernetes cluster on AWS (Amazon EKS) on every build.

This is the full picture: a push to GitHub ends with a tested image in ECR and a new version rolled out across a Kubernetes cluster, reachable on a public URL.

## What is Kubernetes

Kubernetes (`k8s`) is a container orchestrator. You declare the state you want, for example "run two copies of this image and keep them healthy", and Kubernetes makes the cluster match and keeps it matching. If a copy dies it starts a new one; if you ask for more it adds them; when you ship a new version it replaces the old copies a few at a time so the app never fully goes down.

### The few concepts you need

- **Cluster**: the whole system, a control plane plus worker nodes. Here it is an EKS cluster on AWS.
- **Node**: a machine (an EC2 instance) that runs your containers.
- **Pod**: the smallest unit, wrapping your container. You do not create these directly.
- **Deployment**: declares how many pod copies to run and which image. Handles self-healing and updates.
- **Service**: a stable address for your pods. On EKS, a `LoadBalancer` Service gets a public URL.
- **Manifest**: a YAML file describing the state you want, sent to the cluster with `kubectl apply`.
- **kubectl**: the command-line tool that talks to the cluster.

## What is new compared with Project 3

- A real Kubernetes cluster (EKS) instead of running one container on the host.
- Two manifests: a `Deployment` and a `Service`.
- The final stage is now `Deploy to EKS` using `kubectl`, replacing the old `docker run`.
- The cluster pulls your image straight from ECR. Because EKS worker nodes are granted ECR read access when the cluster is created, you do not need any image pull secret. This is why ECR and EKS pair so well.

## Learning objectives

After this project you will be able to:

1. Explain what Kubernetes adds over running a single container.
2. Define cluster, node, pod, deployment, and service.
3. Create an EKS cluster and let a pipeline deploy to it.
4. Read and write Deployment and Service manifests.
5. Use `kubectl` to scale, watch self-healing, and roll out a new version.

## Prerequisites

- You have completed Project 3: an ECR repository, the `jenkins-ecr` IAM user and access key, and the Jenkins setup.
- `kubectl` and `eksctl` installed, and the AWS CLI configured. See `INSTALL-KUBERNETES.md`.
- An AWS account. Note the cost warning below.

Cost warning: an EKS cluster, its worker nodes, and the load balancer all cost money while they run. Create the cluster when you start and delete it the moment you finish (see Cleanup).

## Project structure

```
jenkins-cicd-project-4-k8s/
├── app/                          # The same Java HTTP app from Project 3
│   └── Dockerfile.local          # Multi-stage build for local practice (no Maven needed)
├── k8s/
│   ├── deployment.yaml           # Replicas, image placeholder, health probes
│   ├── service.yaml              # LoadBalancer for a public URL
│   └── local/                    # Same app for a free local cluster (minikube)
├── Jenkinsfile                   # 7 stages, ending in Deploy to EKS
├── jenkins/
│   ├── Dockerfile                # Jenkins + Maven + Docker CLI + AWS CLI + kubectl
│   └── docker-compose.yml        # Host-socket Jenkins (from Project 3)
├── AWS-ECR-SETUP.md              # ECR repo, IAM user, access key (from Project 3)
├── EKS-SETUP.md                  # Create the cluster, grant Jenkins access
├── INSTALL-KUBERNETES.md         # Install kubectl and eksctl
├── LOCAL-KUBERNETES.md           # Run a free local cluster with minikube
├── RUN-ON-AWS.md                 # Run Jenkins on EC2
├── ASSIGNMENT.md                 # What to submit
└── README.md                     # This file
```

## Practice locally first (free, recommended)

Before you create the paid EKS cluster, learn Kubernetes for free on your own machine. `LOCAL-KUBERNETES.md` sets up a local cluster with minikube, deploys the same app from a locally built image, and lets you practise every `kubectl` command in this project, scaling, self-healing, and rolling updates, at no cost. The concepts and commands are identical; only the image source and the service type differ. Do this first, get comfortable, then come back here for the pipeline-driven deploy to EKS.

## Step 1: Create the EKS cluster

Follow `EKS-SETUP.md`. It creates the cluster with one `eksctl` command, maps the `jenkins-ecr` user into the cluster so the pipeline can deploy, and adds the `eks:DescribeCluster` permission. Confirm with `kubectl get nodes` before going on.

## Step 2: Point the project at your cluster

Open the `Jenkinsfile` and set the values at the top: `AWS_REGION`, `ECR_ACCOUNT`, `ECR_REPO` (the same repo as Project 3), and `EKS_CLUSTER`. Commit and push.

## Step 3: Start (or rebuild) Jenkins

The Jenkins image now includes `kubectl`, so rebuild it:

```bash
cd jenkins
docker compose up -d --build
```

Your `aws-ecr` credential from Project 3 is reused. If you are starting fresh, add it again (README of Project 3, Step 4): Manage Jenkins, Credentials, Username and password, id `aws-ecr`, username is the access key id, password is the secret.

## Step 4: Create the pipeline job and run it

Create a Pipeline job from SCM exactly as before (your repo, branch `*/main`, Script Path `Jenkinsfile`), then Build Now. Watch all seven stages. The last one, `Deploy to EKS`, points `kubectl` at the cluster, substitutes this build's image into the manifest, applies it, and waits for the rollout.

## Step 5: Open the app

Get the public address the load balancer was given:

```bash
kubectl get service cicd-app
```

Look at the `EXTERNAL-IP` column. It is a load balancer hostname and takes two or three minutes to appear the first time. Open `http://<that-hostname>/` in a browser, and try `/health` and `/add?a=2&b=3`. You are reaching your app, running on Kubernetes, deployed by the pipeline.

## Understanding the manifests

### deployment.yaml

- `replicas: 2` is the number of copies Kubernetes keeps running.
- `image: IMAGE_PLACEHOLDER` is replaced by the pipeline with the exact ECR image and tag from this build, so what you deploy is what you just built and tested.
- `imagePullPolicy: Always` makes the node pull the image from ECR.
- The `readinessProbe` keeps traffic away from a pod until `/health` responds; the `livenessProbe` restarts a pod whose `/health` stops. Together they make rolling updates safe.

### service.yaml

- `type: LoadBalancer` makes AWS create a public load balancer in front of your pods.
- `selector: app: cicd-app` tells it which pods to send traffic to.
- `port: 80` is the public port; `targetPort: 8080` is the container's port.

## Understanding the Deploy stage

```groovy
aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER}
sed "s|IMAGE_PLACEHOLDER|${IMAGE}:${IMAGE_TAG}|g" k8s/deployment.yaml | kubectl apply -f -
kubectl apply -f k8s/service.yaml
kubectl rollout status deployment/cicd-app --timeout=180s
```

The first line writes a kubeconfig so `kubectl` can reach the cluster. The `sed` line swaps the placeholder for this build's image and pipes the result straight into `kubectl apply`, which is a clean way to inject a build-specific value without editing files. The rollout status line makes the stage wait until the new pods are live, so a failed rollout fails the build.

## Play with kubectl (against your cluster)

### Scale

```bash
kubectl scale deployment cicd-app --replicas=4
kubectl get pods
```

### Watch self-healing

```bash
kubectl get pods
kubectl delete pod <one-of-the-pod-names>
kubectl get pods -w
```

A replacement pod appears immediately, because the Deployment keeps the replica count you asked for.

### Roll out a new version through the pipeline

This is the payoff. Change the message in `app/src/main/java/guru/elevatehub/App.java`, commit, and push. Run the pipeline again. Each build has a new image tag, so the `Deploy to EKS` stage rolls the cluster over to the new version a few pods at a time. Watch it with:

```bash
kubectl rollout status deployment/cicd-app
```

Then refresh the URL to see the new message. If a release goes bad, `kubectl rollout undo deployment/cicd-app` returns to the previous version.

### Inspect

```bash
kubectl get pods
kubectl logs <pod-name>
kubectl describe deployment cicd-app
```

## Troubleshooting

- **Deploy stage: `error: You must be logged in to the server (Unauthorized)`.** The `jenkins-ecr` user is not mapped into the cluster. Run the `eksctl create iamidentitymapping` command from `EKS-SETUP.md`.
- **Deploy stage: `AccessDenied` on `eks:DescribeCluster`.** Add the inline policy from `EKS-SETUP.md` Step 3 to the user.
- **Pods in `ImagePullBackOff`.** The node cannot pull from ECR. Confirm the cluster was created with `eksctl` (its node role gets ECR read access), and that the account id and region in the image name are correct.
- **`EXTERNAL-IP` stays `<pending>`.** Give the load balancer two or three minutes. If it never appears, the cluster's subnets may lack the tags eksctl normally adds; recreate with eksctl.
- **`kubectl rollout status` times out.** A pod is not becoming ready. Run `kubectl describe pod <name>` and `kubectl logs <name>` to see why.

## Cleanup (important, this costs money)

```bash
kubectl delete -f k8s/service.yaml     # removes the load balancer first
eksctl delete cluster --name cicd-cluster --region us-east-1
cd jenkins && docker compose down -v
```

Delete the load balancer (by deleting the Service) before deleting the cluster, or the load balancer can be left behind and keep charging. Also delete the ECR images and the IAM access key when you are finished with the whole series.

## What comes next

You now have full CI/CD to Kubernetes. The next steps in the real world are to trigger builds automatically on every push with a GitHub webhook, to package your manifests with Helm so they are reusable, and to adopt a GitOps tool so the cluster syncs itself from Git. Those build directly on what you have here.
