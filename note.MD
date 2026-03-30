complete real-world setup — Full CI/CD pipeline on AWS with a sample Node.js app. Every file is production-ready.

Folder Structure
```
project/
├── app/                        # Sample Node.js app
│   ├── app.js
│   ├── package.json
│   └── Dockerfile
├── helm/                       # Helm chart
│   └── myapp/
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           └── ingress.yaml
├── terraform/                  # AWS infra (ECR + EKS)
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── jenkins/                    # Jenkins shared library
│   └── vars/
│       └── standardPipeline.groovy
└── Jenkinsfile                 # Per-repo pipeline entry point

```

# STEP 1 — Sample Node.js AppCreated 4 filesCreated 4 files
# STEP 2 — Terraform (ECR + EKS on AWS)Created 3 filesCreated 3 files
# STEP 3 — Helm ChartCreated 5 filesCreated 5 files
# STEP 4 — Jenkins Shared Library + JenkinsfileCreated 2 filesCreated 2 files
# STEP 5 — README with exact commands to run everythingCreated a file, ran a command, read a fileCreated a file, ran a command, read a fileHere's everything you need. The only 3 things you must change before running:
```
In terraform/main.tf → replace my-terraform-state-bucket with your S3 bucket name
```
```
In Jenkinsfile → replace 123456789 with your actual AWS account ID
```
```
In helm/myapp/values.yaml → replace myapp.example.com with your domain
```
```

Order to run:
terraform apply   →   Jenkins setup   →   git push   →   watch pipeline fire
```

# The --atomic flag in the Helm deploy means if anything goes wrong during deployment, it auto-rolls back to the previous working release — so production is never left in a broken state
