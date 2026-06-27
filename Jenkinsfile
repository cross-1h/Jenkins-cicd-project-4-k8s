// Project 4: Continuous delivery to Kubernetes (Amazon EKS).
//
// This continues Project 3. The first six stages are the same (build, test,
// package, image, push to ECR). The final stage now deploys to an EKS cluster
// with kubectl instead of running a single container by hand.
//
// EDIT the values marked below to match your AWS account and cluster.

pipeline {
    agent any

    environment {
        // ----- EDIT THESE -----
        AWS_REGION  = 'us-east-1'          // your region
        ECR_ACCOUNT = '922685583704'       // your 12-digit AWS account id
        ECR_REPO    = 'cicd-project-3'     // the ECR repository (reused from Project 3)
        EKS_CLUSTER = 'cicd-cluster'       // your EKS cluster name
        // ----------------------

        IMAGE_TAG    = "${BUILD_NUMBER}"
        ECR_REGISTRY = "${ECR_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE        = "${ECR_REGISTRY}/${ECR_REPO}"

        // Jenkins credential 'aws-ecr': USR = access key id, PSW = secret.
        AWS_CREDS    = credentials('aws-ecr')
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build') {
            steps { dir('app') { sh 'mvn -B -ntp clean compile' } }
        }

        stage('Test') {
            steps { dir('app') { sh 'mvn -B -ntp test' } }
            post { always { junit 'app/target/surefire-reports/*.xml' } }
        }

        stage('Package') {
            steps { dir('app') { sh 'mvn -B -ntp package -DskipTests' } }
        }

        stage('Docker Build') {
            steps {
                dir('app') {
                    sh 'docker build --platform linux/amd64 -t ${IMAGE}:${IMAGE_TAG} -t ${IMAGE}:latest .'
                }
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                    export AWS_ACCESS_KEY_ID=$AWS_CREDS_USR
                    export AWS_SECRET_ACCESS_KEY=$AWS_CREDS_PSW
                    
                    # Explicity set the default region for the AWS CLI session
                    export AWS_DEFAULT_REGION="${AWS_REGION}"

                    aws ecr describe-repositories --region ${AWS_REGION} --repository-names ${ECR_REPO} \
                      || aws ecr create-repository --region ${AWS_REGION} --repository-name ${ECR_REPO}
                    aws ecr get-login-password --region ${AWS_REGION} \
                      | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    docker push ${IMAGE}:${IMAGE_TAG}
                    docker push ${IMAGE}:latest
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                    export AWS_ACCESS_KEY_ID=$AWS_CREDS_USR
                    export AWS_SECRET_ACCESS_KEY=$AWS_CREDS_PSW

                    # Point kubectl at the cluster.
                    aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER}

                    # Put this build's image into the manifest, then apply.
                    sed "s|IMAGE_PLACEHOLDER|${IMAGE}:${IMAGE_TAG}|g" k8s/deployment.yaml | kubectl apply -f -
                    kubectl apply -f k8s/service.yaml

                    # Wait for the rollout, then show the public address.
                    kubectl rollout status deployment/cicd-app --timeout=180s
                    kubectl get service cicd-app
                '''
            }
        }
    }

    post {
        success {
            echo "Deployed ${IMAGE}:${IMAGE_TAG} to ${EKS_CLUSTER}."
            echo "Get the app URL with: kubectl get service cicd-app (use the EXTERNAL-IP)."
        }
        failure {
            echo 'Pipeline failed. Scroll up and read the first red error in the log.'
        }
    }
}
