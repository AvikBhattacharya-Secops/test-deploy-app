# Multi Stage Docker Build



| Tool                        | Description                          |
| --------------------------- | ------------------------------------ |
| Jenkins                     | Running on master node or external   |
| ArgoCD                      | Installed in cluster                 |
| self-managed cluster | Already set up                       |
| Helm                        | Installed on your system or Jenkins  |
| AWS CLI                     | Set up with IAM user with ECR access |
| Docker Hub account          | For pushing public/private image     |


#Final Pipeline Summary
========================

Jenkins builds Docker image from calculator.go

Pushes to DockerHub and AWS ECR

Updates Helm values.yaml with new image tag

Commits and pushes change to GitHub

ArgoCD detects Git change and deploys via Helm

K8s exposes the service via LoadBalancer

Optionally scales based on traffic
