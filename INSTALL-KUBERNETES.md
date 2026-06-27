# Installing kubectl and eksctl (Windows and Linux)

You need two command-line tools on your own machine:

- `kubectl`: talks to a Kubernetes cluster.
- `eksctl`: creates and deletes Amazon EKS clusters with one command.

You also need the AWS CLI configured with credentials that can create an EKS cluster (your admin credentials, separate from the limited Jenkins key). Run `aws configure` if you have not already.

On an ARM machine, replace `amd64` with `arm64` in the Linux URLs.

## Windows

With `winget`:

```powershell
winget install -e --id Kubernetes.kubectl
winget install -e --id Weaveworks.eksctl
```

Or with Chocolatey:

```powershell
choco install kubernetes-cli -y
choco install eksctl -y
```

## Linux

### kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```

### eksctl

```bash
curl -sSL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz" \
  | sudo tar -xz -C /usr/local/bin
eksctl version
```

## macOS (for reference)

```bash
brew install kubectl eksctl
```

## Verify

```bash
kubectl version --client
eksctl version
aws sts get-caller-identity
```

The last command confirms the AWS CLI knows who you are. If it errors, run `aws configure`.
