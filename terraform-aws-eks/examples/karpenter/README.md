# Karpenter Example

<<<<<<< HEAD
=======
Note: To change VPC with the existing one and with the existing subnets, just change the following variable values from `terraform-aws-eks/examples/karpenter/main.tf` file and 'module eks'

```bash
  vpc_id                   = module.vpc.vpc_id
  subnet_ids               = module.vpc.private_subnets
  control_plane_subnet_ids = module.vpc.intra_subnets
```

>>>>>>> 452c8584f4ac17539cc68c7c975a8b7ceb469a58

## Usage

To provision the provided configurations you need to execute:

```bash
$ git clone https://github.com/anar-shamilov/task-code-files.git
$ cd task-code-files/terraform-aws-eks/examples/karpenter
$ terraform init
$ terraform plan
$ terraform apply --auto-approve
```

Once the cluster is up and running, you can check that Karpenter is functioning as intended with the following command:

```bash
# First, make sure you have updated your local kubeconfig
aws eks --region eu-west-1 update-kubeconfig --name ex-karpenter

# Second, for test apply manifest files amd64-deployment.yaml and arm64-deployment.yaml 
cd ../../../manifests
kubectl apply -f .

# You can watch Karpenter's controller logs with
kubectl logs -f -n kube-system -l app.kubernetes.io/name=karpenter -c controller
```

# Important: If in Log from karpetner you face with bellow problem:
```bash
AuthFailure.ServiceLinkedRoleCreationNotPermitted: The provided credentials do not have permission to create the service-linked role for EC2 Spot Instances. (https://github.com/aws/amazon-vpc-cni-k8s/blob/master/docs/iam-policy.md)

# Resoultion:
aws iam create-service-linked-role --aws-service-name spot.amazonaws.com
```

Validate if the Pods are running and the  application Pods are running on Karpenter provisioned Nodes.

```bash
# wait until pods STATUS will be running
$ watch kubectl get po
NAME                         READY   STATUS    RESTARTS   AGE
amd64-app-7b7f679879-c78w4   1/1     Running   0          8m7s
arm64-app-577cd855c7-qjfht   1/1     Running   0          8m7s

# then get nodes where are that pods running
$ kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeNam 
NAME                         NODE
amd64-app-7b7f679879-c78w4   ip-10-0-25-164.eu-west-1.compute.internal
arm64-app-577cd855c7-qjfht   ip-10-0-26-43.eu-west-1.compute.internal

# check nodes which registerd by karpenter
kubectl get nodes -L karpenter.sh/registered
```

```text
NAME                                        STATUS   ROLES    AGE   VERSION               REGISTERED
ip-10-0-25-164.eu-west-1.compute.internal   Ready    <none>   28m   v1.30.0-eks-036c24b   true
ip-10-0-26-43.eu-west-1.compute.internal    Ready    <none>   28m   v1.30.0-eks-036c24b   true
ip-10-0-46-213.eu-west-1.compute.internal   Ready    <none>   75m   v1.30.0-eks-036c24b 
```
Now we check the instances to see if they match our requirements

```bash
# Use the following command to get the instance ID by filtering on the instance name.
$ aws ec2 describe-instances --filters "Name=tag:Name,Values=ip-10-0-26-43.eu-west-1.compute.internal" --query "Reservations[*].Instances[*].InstanceId" --output text
# as a result Architecture Type should be arm64 for bellow ID
i-04b9685771fcaeee3

$ aws ec2 describe-instances --filters "Name=tag:Name,Values=ip-10-0-25-164.eu-west-1.compute.internal" --query "Reservations[*].Instances[*].InstanceId" --output text
# as a result Architecture Type should be amd64 or x86_64 for bellow ID
i-0d9e8d084120e2bce

# Use the instance ID obtained in the previous step to get the architecture type of the instance. 

$ aws ec2 describe-instances --instance-ids i-04b9685771fcaeee3 --query "Reservations[*].Instances[*].Architecture" --output text
# as expected
arm64

$ aws ec2 describe-instances --instance-ids i-0d9e8d084120e2bce --query "Reservations[*].Instances[*].Architecture" --output text
#as expected
x86_64
```


### Tear Down & Clean-Up

Because Karpenter manages the state of node resources outside of Terraform, Karpenter created resources will need to be de-provisioned first before removing the remaining resources with Terraform.

1. Remove the example deployment created above and any nodes created by Karpenter

```bash
kubectl delete -f .

# wait a few minutes any nodes created by Karpenter must be deleted automatically and we can see only one node
$ kubectl get node
NAME                                        STATUS   ROLES    AGE    VERSION
ip-10-0-46-213.eu-west-1.compute.internal   Ready    <none>   100m   v1.30.0-eks-036c24b
```

2. Remove the resources created by Terraform

```bash
cd cd ../terraform-aws-eks/examples/karpenter
terraform destroy --auto-approve
```

Note that this example may create resources which cost money. Run `terraform destroy` when you don't need these resources.

<!-- BEGINNING OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
## Requirements

| Name | Version |
|------|---------|
| <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) | >= 1.3.2 |
| <a name="requirement_aws"></a> [aws](#requirement\_aws) | >= 5.40 |
| <a name="requirement_helm"></a> [helm](#requirement\_helm) | >= 2.7 |
| <a name="requirement_kubectl"></a> [kubectl](#requirement\_kubectl) | >= 2.0 |

## Providers

| Name | Version |
|------|---------|
| <a name="provider_aws"></a> [aws](#provider\_aws) | >= 5.40 |
| <a name="provider_aws.virginia"></a> [aws.virginia](#provider\_aws.virginia) | >= 5.40 |
| <a name="provider_helm"></a> [helm](#provider\_helm) | >= 2.7 |
| <a name="provider_kubectl"></a> [kubectl](#provider\_kubectl) | >= 2.0 |

## Modules

| Name | Source | Version |
|------|--------|---------|
| <a name="module_eks"></a> [eks](#module\_eks) | ../.. | n/a |
| <a name="module_karpenter"></a> [karpenter](#module\_karpenter) | ../../modules/karpenter | n/a |
| <a name="module_karpenter_disabled"></a> [karpenter\_disabled](#module\_karpenter\_disabled) | ../../modules/karpenter | n/a |
| <a name="module_vpc"></a> [vpc](#module\_vpc) | terraform-aws-modules/vpc/aws | ~> 5.0 |

## Resources

| Name | Type |
|------|------|
| [helm_release.karpenter](https://registry.terraform.io/providers/hashicorp/helm/latest/docs/resources/release) | resource |
| [kubectl_manifest.karpenter_example_deployment](https://registry.terraform.io/providers/alekc/kubectl/latest/docs/resources/manifest) | resource |
| [kubectl_manifest.karpenter_node_class](https://registry.terraform.io/providers/alekc/kubectl/latest/docs/resources/manifest) | resource |
| [kubectl_manifest.karpenter_node_pool](https://registry.terraform.io/providers/alekc/kubectl/latest/docs/resources/manifest) | resource |
| [aws_availability_zones.available](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/availability_zones) | data source |
| [aws_ecrpublic_authorization_token.token](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/ecrpublic_authorization_token) | data source |
<<<<<<< HEAD

=======
>>>>>>> 452c8584f4ac17539cc68c7c975a8b7ceb469a58
