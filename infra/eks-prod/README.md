# prod cluster — AWS EKS (Terraform)

Planned Terraform layout:
```
main.tf          # module calls: vpc, eks, ecr, irsa
variables.tf
outputs.tf
backend.tf       # S3 remote state + DynamoDB lock
environments/
  prod.tfvars
```
Suggested modules: `terraform-aws-modules/vpc/aws`,
`terraform-aws-modules/eks/aws`. Use IRSA (IAM Roles for Service Accounts)
rather than static AWS keys inside pods — External Secrets and any
in-cluster AWS access should go through this.
