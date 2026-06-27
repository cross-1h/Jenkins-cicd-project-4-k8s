# Creating the EKS Cluster and Granting Jenkins Access

Do this once. It creates the Kubernetes cluster your pipeline deploys to, and gives the Jenkins IAM user permission to deploy to it.

Cost warning: an EKS cluster is not free. The control plane costs about 0.10 US dollars per hour, plus the worker nodes (EC2) and the load balancer. Create it when you start, and delete it the moment you are done (see Cleanup).

You should have finished `AWS-ECR-SETUP.md` first, so you already have the `jenkins-ecr` IAM user and its access key.

## 1. Create the cluster

This single command builds the cluster, a VPC, and two worker nodes. It takes about 15 minutes. The managed node group automatically gets permission to pull images from ECR, which is why no image pull secret is needed.

```bash
eksctl create cluster \
  --name cicd-cluster \
  --region us-east-1 \
  --nodes 2 \
  --node-type t3.small \
  --managed
```

When it finishes, eksctl points `kubectl` at the new cluster. Verify:

```bash
kubectl get nodes
```

You should see two nodes that are `Ready`.

## 2. Let the Jenkins user deploy to the cluster

By default only the identity that created the cluster can use it. Map the `jenkins-ecr` user in so the pipeline can deploy. Replace the account id with yours.

```bash
eksctl create iamidentitymapping \
  --cluster cicd-cluster \
  --region us-east-1 \
  --arn arn:aws:iam::123456789012:user/jenkins-ecr \
  --group system:masters \
  --username jenkins
```

## 3. Allow the Jenkins user to read the cluster

The pipeline runs `aws eks update-kubeconfig`, which needs the `eks:DescribeCluster` permission. Add this inline policy to the `jenkins-ecr` user.

Console: IAM, Users, `jenkins-ecr`, Add permissions, Create inline policy, JSON tab, paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": "eks:DescribeCluster", "Resource": "*" }
  ]
}
```

Name it `jenkins-eks-describe` and save.

## 4. Confirm

```bash
kubectl get svc
```

If that works, your cluster is ready and the pipeline will be able to deploy to it.

## Cleanup (do this to stop charges)

Delete the app's load balancer first by removing the Service, then delete the cluster:

```bash
kubectl delete -f k8s/service.yaml     # removes the load balancer
eksctl delete cluster --name cicd-cluster --region us-east-1
```

Also delete the ECR images and the `jenkins-ecr` access key when you are completely done with the whole project series.
