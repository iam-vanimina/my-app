# my-app
CI/CD Pipeline Setup — Jenkins + Docker + ECR + EKS + Helm(nodejs)

# CI/CD Pipeline Setup — Jenkins + Docker + ECR + EKS + Helm

## Prerequisites
- AWS CLI configured (`aws configure`)
- Terraform >= 1.3
- kubectl
- Helm >= 3
- Docker
- A GitHub repo with this code

---

## Step 1 — Provision AWS Infrastructure (Terraform)

```bash
cd terraform/

# First time only: create S3 bucket + DynamoDB for state
aws s3 mb s3://my-terraform-state-bucket --region ap-south-1
aws dynamodb create-table \
  --table-name terraform-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ap-south-1

# Deploy ECR + EKS + IAM
terraform init
terraform plan
terraform apply
```

After apply, note the outputs:
```
ecr_repository_url    = "123456789.dkr.ecr.ap-south-1.amazonaws.com/myapp"
configure_kubectl     = "aws eks update-kubeconfig ..."
jenkins_instance_profile = "jenkins-instance-profile"
```

---

## Step 2 — Configure kubectl

```bash
# Copy the configure_kubectl output and run it:
aws eks update-kubeconfig --region ap-south-1 --name myapp-cluster

# Verify
kubectl get nodes
```

---

## Step 3 — Install Jenkins on EC2

```bash
# Launch an EC2 (t3.medium, Amazon Linux 2)
# IMPORTANT: attach the IAM instance profile from Terraform output
#   EC2 Console → Actions → Security → Modify IAM Role → select jenkins-instance-profile

# SSH into EC2, then:
sudo yum update -y
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum install jenkins java-17-amazon-corretto docker git -y
sudo systemctl enable --now jenkins docker
sudo usermod -aG docker jenkins

# Get initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

---

## Step 4 — Configure Jenkins

1. Open `http://<EC2-IP>:8080`
2. Install suggested plugins + **GitHub**, **Docker Pipeline**, **Kubernetes CLI** plugins
3. Add Shared Library:
   - Manage Jenkins → System → Global Pipeline Libraries
   - Name: `jenkins-shared-library`
   - Source: your GitHub repo URL (jenkins/ folder)
4. Create Pipeline job:
   - New Item → Pipeline
   - GitHub Project → your repo URL
   - Build Trigger: **GitHub hook trigger for GITScm polling**
   - Pipeline: **Pipeline script from SCM** → Git → your repo → `Jenkinsfile`
5. Add GitHub webhook:
   - GitHub repo → Settings → Webhooks → Add
   - URL: `http://<EC2-IP>:8080/github-webhook/`
   - Content type: `application/json`
   - Event: **Just the push event**

---

## Step 5 — Install Helm on Jenkins EC2

```bash
sudo ssh ec2-user@<jenkins-ip>
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

## Step 6 — Update Jenkinsfile with your values

Edit `Jenkinsfile`:
```groovy
standardPipeline(
  appName:     'myapp',
  ecrRepo:     '<YOUR_AWS_ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/myapp',
  awsRegion:   'ap-south-1',
  clusterName: 'myapp-cluster',
  namespace:   'production'
)
```

---

## Step 7 — First Deploy

```bash
git add .
git commit -m "initial setup"
git push origin main
```

Watch Jenkins → your pipeline job → Console Output

Expected flow:
```
[Source]  ✓  Commit SHA captured
[Build]   ✓  Docker image built
[Test]    ✓  Tests passed
[Push]    ✓  Image pushed to ECR
[Deploy]  ✓  Helm deployed to EKS
```

---

## Verify Deployment

```bash
# Check pods
kubectl get pods -n production

# Check service
kubectl get svc -n production

# Check ingress (ALB URL)
kubectl get ingress -n production

# Test the app
curl http://<ALB-URL>/health
# Expected: {"status":"healthy","version":"a3f9c12"}
```

---

## Useful Commands

```bash
# Check Helm releases
helm list -n production

# Rollback if something goes wrong
helm rollback myapp 1 -n production

# Check rollout history
helm history myapp -n production

# View pod logs
kubectl logs -l app=myapp -n production --tail=100

# Scale manually
kubectl scale deployment myapp --replicas=3 -n production
```

---

## Cost Estimate (ap-south-1)

| Resource       | Spec           | ~Cost/month |
|----------------|----------------|-------------|
| EKS Cluster    | Control plane  | $72         |
| EC2 Nodes      | 2x t3.medium   | $60         |
| NAT Gateway    | 1x             | $35         |
| ECR            | Storage + data | ~$1         |
| Jenkins EC2    | t3.medium      | $30         |
| **Total**      |                | **~$198**   |

> Tip: Stop Jenkins EC2 and scale EKS nodes to 0 when not in use to save cost.
