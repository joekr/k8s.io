# Provisioninig EKS clusters

## Prod vs Canary

These scripts support provisioning two types of EKS clusters. One is meant for hosting prow jobs
on production and the other one is for testing infrastructure changes before promoting them to
production.

Here are some differences between canary and production setups:
* cluster name,
* cluster admin IAM role name,
* secrets-manager IAM policy name,
* canary is missing k8s prow OIDC provider and corresponding role,
* subnet setup is different,
* instance type and autoscaling paramethers (mainly for saving),

## Provisioning Cluster

Running installation from scratch is different than consecutive invocations of Terraform.
First run creates a role that can be later assumed by other users. Becasue of that additional
variable has to be set:

```bash
# For provisioning Prod:
export WORKSPACE_NAME=prod
# For provisioning Canary:
export WORKSPACE_NAME=canary

# Just making sure we don't have state cached locally.
ASSUME_ROLE=false make init
ASSUME_ROLE=false make apply
```

Once the infrastructure is provisioned, next step is RBAC setup:

```bash
# Fetch & update kubeconfig.
# For Prod:
aws eks update-kubeconfig --region us-east-2 --name prow-build-cluster
# For Canary:
aws eks update-kubeconfig --region us-east-2 --name prow-build-canary-cluster

# create cluster role bindings
kubectl apply -f ./resources/rbac
```

Lastly, run Terraform script again without additinal variable. This time, it will implicitly assume
previously created role and provision resources on top of EKS cluster.

```bash
make apply
```

From here, all consecutive runs should be possible with command from above.

## Using cluster

### Fetch kubeconfig

```bash
# Prod:
aws eks update-kubeconfig --region us-east-2 --name prow-build-cluster
# Canary:
aws eks update-kubeconfig --region us-east-2 --name prow-build-canary-cluster
```

### Open kubeconfig and add assume role argument

For Prod:
```yaml
args:
  - --region
  - us-east-2
  - eks
  - get-token
  - --cluster-name
  - prow-build-cluster
  - --role-arn
  - arn:aws:iam::468814281478:role/Prow-Cluster-Admin
```

For Canary:
```yaml
args:
  - --region
  - us-east-2
  - eks
  - get-token
  - --cluster-name
  - prow-build-canary-cluster
  - --role-arn
  - arn:aws:iam::468814281478:role/canary-Prow-Cluster-Admin
```

## Removing cluster

Same as for installation, cluster removal requires running Terraform twice.
**IMPORTANT**: It's possible only for users with assigned `AdministratorAccess` policy.

```bash
export WORKSPACE_NAME= # choose between canary/prod

# First remove resources running on the cluster and IAM role. This fails once assumed role gets deleted.
make destroy

# Clean up the rest. 
ASSUME_ROLE=false make destroy
```

