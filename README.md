# EKS Practice Cluster with Terraform

Terraform code to create an Amazon EKS cluster on AWS for hands-on practice. It uses the community `terraform-aws-modules` VPC and EKS modules and is tuned to be low cost (Spot nodes, single NAT gateway).

> **Cost warning:** EKS control plane, NAT gateway and EC2 nodes are billed hourly. Always run `terraform destroy` when you finish practising.

---

## 1. What gets created

| Resource | Details |
|---|---|
| VPC | CIDR `10.0.0.0/16`, 2 AZs |
| Subnets | 2 public + 2 private (tagged for Kubernetes load balancers) |
| NAT Gateway | 1 (single, to save cost) |
| EKS cluster | Public API endpoint, Kubernetes version set by `kubernetes_version` |
| Add-ons | CoreDNS, kube-proxy, VPC CNI |
| Node group | Managed node group, `t3.medium` **Spot**, desired 2 (min 1, max 3) |
| Access | The IAM identity running Terraform gets cluster admin access |

Estimated time to create: **12-15 minutes**.

---

## 2. Project structure

```
eks-terraform/
├── versions.tf     # Terraform + AWS provider versions, provider config
├── variables.tf    # Input variables (region, cluster name, node sizes...)
├── main.tf         # VPC and EKS modules
├── outputs.tf      # Cluster name, endpoint, kubectl command
└── README.md
```

---

## 3. Prerequisites

Install these tools (Windows examples use PowerShell and `winget`/`choco`):

| Tool | Minimum version | Install |
|---|---|---|
| Terraform | 1.5.0 | `winget install Hashicorp.Terraform` |
| AWS CLI v2 | 2.x | `winget install Amazon.AWSCLI` |
| kubectl | Close to your cluster version | `winget install Kubernetes.kubectl` |
| Git | any | `winget install Git.Git` |

Verify:

```powershell
terraform -version
aws --version
kubectl version --client
git --version
```

### AWS account and permissions

- An AWS account with an IAM user or role that can create VPC, EC2, EKS, IAM, KMS and CloudWatch Logs resources. For practice, `AdministratorAccess` is the simplest option.
- Make sure your account has enough quota for the instances and Elastic IPs used.

---

## 4. Configure AWS credentials

```powershell
aws configure
```

Enter your Access Key ID, Secret Access Key, default region (for example `ap-south-1`) and output format (`json`).

Confirm which identity Terraform will use:

```powershell
aws sts get-caller-identity
```

> If you use AWS SSO, run `aws configure sso`, then `aws sso login --profile <profile>` and `$env:AWS_PROFILE="<profile>"`.

---

## 5. Configuration

Defaults are defined in `variables.tf`:

| Variable | Default | Description |
|---|---|---|
| `region` | `ap-south-1` | AWS region |
| `cluster_name` | `eks-practice` | EKS cluster name |
| `kubernetes_version` | `1.33` | EKS Kubernetes version (use a version EKS currently supports) |
| `vpc_cidr` | `10.0.0.0/16` | VPC CIDR block |
| `node_instance_types` | `["t3.medium"]` | Node instance types |
| `node_desired_size` | `2` | Desired node count |
| `node_min_size` | `1` | Minimum node count |
| `node_max_size` | `3` | Maximum node count |

To override values without editing files, create `terraform.tfvars`:

```hcl
region             = "ap-south-1"
cluster_name       = "my-eks"
kubernetes_version = "1.33"
node_desired_size  = 2
```

Or pass them on the command line: `terraform apply -var="cluster_name=my-eks"`.

---

## 6. Deploy

Run all commands from the project folder.

```powershell
# 1. Download providers and modules
terraform init

# 2. Check formatting and syntax (optional but good practice)
terraform fmt
terraform validate

# 3. Preview what will be created
terraform plan

# 4. Create the infrastructure (type 'yes' when asked)
terraform apply
```

When it finishes, Terraform prints the outputs, including the `configure_kubectl` command.

---

## 7. Connect to the cluster

```powershell
aws eks update-kubeconfig --region ap-south-1 --name eks-practice
```

(Use your own region and cluster name if you changed them.)

Verify:

```powershell
kubectl get nodes
kubectl get pods -A
kubectl cluster-info
```

You should see 2 nodes in `Ready` state and the CoreDNS, kube-proxy and aws-node pods running in `kube-system`.

---

## 8. Test with a sample app

Create `nginx.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
```

Deploy and check:

```powershell
kubectl apply -f nginx.yaml
kubectl get pods
kubectl get svc nginx --watch
```

When `EXTERNAL-IP` shows a load balancer hostname (this can take a few minutes), open `http://<EXTERNAL-IP>` in your browser to see the nginx welcome page.

Clean up the app:

```powershell
kubectl delete -f nginx.yaml
```

---

## 9. Destroy (important)

Delete any Kubernetes `LoadBalancer` services and PVCs first. They create AWS resources (ELBs, EBS volumes) that Terraform does not manage and that can block VPC deletion.

```powershell
kubectl delete -f nginx.yaml
kubectl get svc -A      # confirm no LoadBalancer services remain

terraform destroy       # type 'yes' when asked
```

Afterwards, check the AWS console (EC2 > Load Balancers, EC2 > Volumes, VPC) for leftovers.

---

## 10. Troubleshooting

| Problem | Fix |
|---|---|
| `Error: No valid credential sources found` | Run `aws configure` or set `AWS_PROFILE`; check `aws sts get-caller-identity` |
| `UnsupportedAvailabilityZone` or version error | Change `kubernetes_version` to one EKS supports, or check AZ availability in your region |
| `kubectl` says `Unauthorized` or `You must be logged in` | Re-run `aws eks update-kubeconfig`; make sure you use the same IAM identity that ran `terraform apply` |
| Nodes stuck in `NotReady` or node group creation fails | Check Spot capacity; change `capacity_type` to `"ON_DEMAND"` in `main.tf` or try another instance type |
| `terraform destroy` hangs on VPC or subnets | Delete leftover load balancers, ENIs and EBS volumes created by Kubernetes, then retry |
| `Error acquiring the state lock` | Another run is in progress; wait, or `terraform force-unlock <ID>` if it crashed |
| `LimitExceeded` for Elastic IPs or vCPUs | Request a quota increase or clean up unused resources |

---

## 11. Push to GitHub

Create a `.gitignore` so state and provider files are never committed:

```gitignore
.terraform/
*.tfstate
*.tfstate.*
crash.log
*.tfvars
*.tfplan
```

Commit `.terraform.lock.hcl` so provider versions stay consistent.

```powershell
git add .
git commit -m "Add EKS Terraform code and README"
git push -u origin main
```

> Never commit `terraform.tfstate`. State files can contain sensitive values.

---


