# Project 4 Assignment: Continuous Delivery to Kubernetes (EKS)

**Due: [set your deadline]**

Follow the `README.md` to deploy your app to an EKS cluster from the Jenkins pipeline, then complete the tasks below and submit the evidence listed.

## What to do

1. Create the EKS cluster and grant the Jenkins user access (`EKS-SETUP.md`).
2. Set `AWS_REGION`, `ECR_ACCOUNT`, `ECR_REPO`, and `EKS_CLUSTER` in the `Jenkinsfile` and push.
3. Rebuild Jenkins so it has `kubectl`, then run the pipeline until all seven stages pass, ending with `Deploy to EKS`.
4. Open the app in a browser through the load balancer URL.
5. Scale the deployment to 4 replicas, then back to 2.
6. Delete one pod and watch Kubernetes self-heal.
7. Roll out a new version through the pipeline: change the app's message, push, and run the pipeline again.

## What to submit

1. The link to your GitHub repository.
2. A screenshot of a green pipeline run showing all seven stages passing, including `Push to ECR` and `Deploy to EKS`.
3. A screenshot of `kubectl get pods` showing the pods `Running` and `Ready`.
4. A screenshot of `kubectl get service cicd-app` showing the load balancer `EXTERNAL-IP`.
5. A screenshot of the app open in a browser at that URL.
6. A screenshot of `kubectl get pods` after scaling to 4 replicas.
7. A screenshot showing self-healing: the pod list right after deleting a pod, with a replacement being created.
8. A screenshot of the new version live after a pipeline rollout (the changed message in the browser).
9. One or two sentences in your own words: how does the image you built reach the cluster, and why is no image pull secret needed?

## Grading (100 points)

- EKS cluster created and Jenkins user mapped in: 15
- Green seven-stage pipeline including Deploy to EKS: 25
- App reachable through the load balancer URL: 20
- Scaling to 4 replicas shown: 10
- Self-healing demonstrated: 15
- New version rolled out through the pipeline: 15

## Cleanup reminder

This project costs money. When done, delete the Service (load balancer), delete the cluster with `eksctl delete cluster`, and stop Jenkins. Delete ECR images and the IAM access key when finished with the series.

## How to submit

Reply to the assignment email with your repository link and the screenshots attached.
