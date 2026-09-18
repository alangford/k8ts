# EKS Test: Node.js, Docker, ECR, Terraform, and Helm

A small hands-on project for provisioning an AWS EKS cluster, building a Node.js container, publishing it to Amazon ECR, and deploying it to Kubernetes with Helm.

## What this project creates

- **VPC** with public and private subnets across `us-east-1a` and `us-east-1b`
- **One NAT Gateway** for private-subnet outbound connectivity
- **Amazon EKS** cluster named `eks-test`, Kubernetes `1.33`
- **One managed worker node** (`t3.medium`)
- **EKS add-ons**: VPC CNI and kube-proxy configured before compute; CoreDNS
- **Amazon ECR** repository `eks-test-app` with image scanning on push and a lifecycle policy retaining the 10 most recent images
- **Node.js HTTP app** listening on port 3000
- **Helm chart** in `nodeHello/` that deploys a ConfigMap, Deployment, and LoadBalancer Service

> This is a learning/lab configuration, not a production-ready architecture. It enables the public EKS API endpoint and uses a single NAT Gateway and a single worker node.

## Repository layout

```text
.
├── main.tf                 # VPC, EKS, managed node group, ECR, lifecycle policy
├── providers.tf            # Terraform/provider requirements and AWS region
├── Dockerfile              # Container build instructions (file is named `dockerfile`)
├── package.json            # Node app metadata and start script
├── server.js               # Minimal HTTP server
└── nodeHello/
    ├── Chart.yaml
    ├── values.yaml
    ├── values-dev.yaml
    ├── values-prod.yaml
    └── templates/
        ├── configmap.yaml
        ├── deployment.yaml
        └── service.yaml
```

## Prerequisites

Install and configure:

- AWS CLI
- Terraform (version `>= 1.5.0`)
- Docker
- `kubectl`
- Helm 3

Your AWS identity needs permissions to create/manage the resources in this lab, including VPC, EKS, EC2/IAM, ECR, and related networking resources. Terraform's cluster-creator admin setting grants Kubernetes access to the identity that creates the cluster; it does not replace AWS IAM permissions for provisioning resources.

Confirm your AWS identity and region:

```bash
aws sts get-caller-identity
aws configure get region
```

The provider is configured for `us-east-1`.

## 1. Provision AWS infrastructure

From the repository root:

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
```

Review the plan before approving. Terraform creates the VPC, EKS cluster/node group, and ECR repository.

Get the ECR repository URL:

```bash
terraform output -raw ecr_repository_url
```

Configure `kubectl` after the cluster is available:

```bash
aws eks update-kubeconfig --region us-east-1 --name eks-test
kubectl get nodes
```

Wait for the node to report `Ready`:

```bash
kubectl wait --for=condition=Ready nodes --all --timeout=10m
```

Check system add-ons if the node does not become ready:

```bash
kubectl get pods -n kube-system -o wide
kubectl get daemonsets -n kube-system
aws eks list-addons --cluster-name eks-test --region us-east-1
```

The VPC CNI provides Pod networking; kube-proxy implements Kubernetes Service networking; CoreDNS provides cluster DNS. A node reporting `NetworkPluginNotReady` / `cni plugin not initialized` points first to the VPC CNI.

## 2. Build and push the Node.js image to ECR

The Terraform output supplies the repository URL. In the examples below, replace `<ACCOUNT_ID>` if you prefer to type it manually, or set `ECR_REPO` from Terraform:

```bash
ECR_REPO=$(terraform output -raw ecr_repository_url)
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
```

Authenticate Docker to ECR:

```bash
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin \
    "$AWS_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com"
```

Build and tag the image (run from the repository root):

```bash
docker build -f dockerfile -t eks-test-app:v1 .
docker tag eks-test-app:v1 "$ECR_REPO:v1"
docker push "$ECR_REPO:v1"
```

Use a new version tag (for example, `v2`) when publishing changed application code. The worker node's IAM role needs ECR image-pull permissions; do not put AWS credentials in the application container.

## 3. Deploy with Helm

The chart's default values currently specify the image repository and `latest` tag. Override the tag with the version you pushed:

```bash
helm upgrade --install test nodeHello/ \
  --namespace default \
  --values nodeHello/values.yaml \
  --set image.tag=v1
```

Check the release and Kubernetes resources:

```bash
helm list -n default
kubectl get deployments,pods,services -n default
kubectl rollout status deployment/hellonode -n default
```

The chart creates a `LoadBalancer` Service on port 80, targeting container port 3000. Get its assigned hostname:

```bash
kubectl get svc hellonode -n default
kubectl get svc hellonode -n default \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

Test it using the **exact hostname returned by Kubernetes**:

```bash
curl http://<load-balancer-hostname>
```

The app responds with `Hello from Node.js!` and the serving Pod's hostname.

### Deploying another version

Build and push a new tag, then upgrade the release:

```bash
docker build -f dockerfile -t eks-test-app:v2 .
docker tag eks-test-app:v2 "$ECR_REPO:v2"
docker push "$ECR_REPO:v2"

helm upgrade test nodeHello/ \
  --namespace default \
  --values nodeHello/values.yaml \
  --set image.tag=v2

kubectl rollout status deployment/hellonode -n default
```

A new image tag changes the Deployment's Pod template and triggers a rollout. The Service can remain in place.

### Helm release names and resource ownership

A Helm release owns the Kubernetes resources rendered by its chart. Installing the same chart as a second release can fail if both releases try to create a resource with the same name (for example, a hard-coded ConfigMap name). For an existing app, use `helm upgrade`; for a genuinely separate release, ensure rendered resource names are unique.

Uninstall the release when you no longer need the app:

```bash
helm uninstall test -n default
```

This removes resources managed by that release, including its LoadBalancer Service, which should trigger deletion of the AWS load balancer.

## 4. Useful troubleshooting commands

```bash
# Workload health and placement
kubectl get pods -o wide
kubectl describe pod <pod-name>
kubectl logs deployment/hellonode -n default

# Service and endpoints
kubectl get svc hellonode -n default
kubectl get endpoints hellonode -n default
kubectl describe svc hellonode -n default

# Node and system add-ons
kubectl get nodes
kubectl get pods -n kube-system -o wide

# Helm
helm list -A
helm get values test -n default
helm get manifest test -n default
```

Common symptoms:

- **`ImagePullBackOff`**: inspect `kubectl describe pod` Events. Verify the image URI/tag exists in ECR, the node can reach ECR, and its IAM role has ECR pull permissions.
- **`cni plugin not initialized` / node `NotReady`**: inspect the `aws-node` Pod and VPC CNI add-on status/logs.
- **Load Balancer hostname doesn't resolve**: copy the exact hostname from `kubectl get svc`; do not construct it manually.
- **`curl` gets an empty reply**: check whether the Pods are Ready and the Service has endpoints. A created Load Balancer alone does not mean the application is healthy.

## 5. Deploy to dev or prod with Helm values

The chart includes environment-specific values files:

- `nodeHello/values-dev.yaml` sets `namespace: dev` and `CUSTOM_HEADER: "dev"`.
- `nodeHello/values-prod.yaml` sets `namespace: prod` and `CUSTOM_HEADER: "prod"`.

Helm applies values files in order, so put the common `values.yaml` first and the environment-specific file second. Use a distinct Helm release name for each environment, and deploy each release into its matching namespace.

### Deploy to dev

```bash
helm upgrade --install test-dev nodeHello/ \
  --namespace dev \
  --create-namespace \
  --values nodeHello/values.yaml \
  --values nodeHello/values-dev.yaml \
  --set image.tag=v1
```

### Deploy to prod

```bash
helm upgrade --install test-prod nodeHello/ \
  --namespace prod \
  --create-namespace \
  --values nodeHello/values.yaml \
  --values nodeHello/values-prod.yaml \
  --set image.tag=v1
```

Replace `v1` with the tag that you pushed to ECR. If your chart's `values.yaml` already points to the correct image repository, only the tag override is needed.

### Verify each environment

```bash
helm list -A
kubectl get all -n dev
kubectl get all -n prod
kubectl get configmap -n dev
kubectl get configmap -n prod
```

Get each environment's LoadBalancer hostname:

```bash
kubectl get svc -n dev
kubectl get svc -n prod
```

Use the hostname shown in the `EXTERNAL-IP` column to test each environment:

```bash
curl http://<dev-load-balancer-hostname>
curl http://<prod-load-balancer-hostname>
```

### Upgrade one environment

For example, after pushing image `v2`, update only dev:

```bash
helm upgrade test-dev nodeHello/ \
  --namespace dev \
  --values nodeHello/values.yaml \
  --values nodeHello/values-dev.yaml \
  --set image.tag=v2
```

Update prod separately when ready:

```bash
helm upgrade test-prod nodeHello/ \
  --namespace prod \
  --values nodeHello/values.yaml \
  --values nodeHello/values-prod.yaml \
  --set image.tag=v2
```

### Remove one environment

```bash
helm uninstall test-dev -n dev
# or
helm uninstall test-prod -n prod
```

Uninstalling a release removes its chart-managed resources, including its LoadBalancer Service, which should trigger deletion of that environment's AWS load balancer.

**Important:** Kubernetes resource names can be the same in different namespaces. Don't install two releases into the same namespace if the chart renders fixed resource names; Helm will report an ownership conflict. Keep `test-dev` in `dev` and `test-prod` in `prod` as shown above. The `--namespace` flag determines where Helm installs resources; the values file's `namespace` field is chart data and does not itself create a Kubernetes namespace.

## 6. Clean up to avoid ongoing AWS charges

First remove the Helm release so its LoadBalancer can be deleted:

```bash
helm uninstall test -n default
```

Then, from the Terraform root:

```bash
terraform destroy
```

Review the destroy plan and confirm. The VPC, EKS cluster/node group, NAT Gateway, ECR repository, and other Terraform-managed resources will be removed. The ECR repository is configured with `force_delete = true`, so Terraform can delete it even when it contains images; those images will be lost.

If Terraform reports dependencies still deleting, wait for the AWS load balancer to disappear and retry the destroy if necessary.

## Cost notes

The lab can incur charges while running. In particular, EKS control-plane time, EC2 worker capacity, NAT Gateway hours/data processing, load balancers, and stored ECR images may cost money. Deleting Kubernetes objects alone does **not** remove the EKS cluster or VPC. Use `terraform destroy` when finished.

## Learning objectives

- Provision VPC and EKS infrastructure with Terraform modules
- Understand EKS add-on ordering and worker-node readiness
- Build and publish a container to ECR
- Deploy and update workloads with Helm
- Trace traffic through an AWS Load Balancer, Kubernetes Service, and Pods
- Diagnose common DNS, CNI, and image-pull failures
