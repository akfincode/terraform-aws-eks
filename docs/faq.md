# Frequently Asked Questions

- [How do I manage the `aws-auth` configmap?](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/docs/faq.md#how-do-i-manage-the-aws-auth-configmap)
- [I received an error: `Error: Invalid for_each argument ...`](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/docs/faq.md#i-received-an-error-error-invalid-for_each-argument-)
- [Why are nodes not being registered?](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/docs/faq.md#why-are-nodes-not-being-registered)
- [Why are there no changes when a node group's `desired_size` is modified?](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/docs/faq.md#why-are-there-no-changes-when-a-node-groups-desired_size-is-modified)
- [How can I deploy Windows based nodes?](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/docs/faq.md#how-can-i-deploy-windows-based-nodes)
- [How do I access compute resource attributes?](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/master/docs/faq.md#how-do-i-access-compute-resource-attributes)

### How do I manage the `aws-auth` configmap?

TL;DR - https://github.com/terraform-aws-modules/terraform-aws-eks/issues/1901

- Users can roll their own equivalent of `kubectl patch ...` using the [`null_resource`](https://github.com/terraform-aws-modules/terraform-aws-eks/blob/9a99689cc13147f4afc426b34ba009875a28614e/examples/complete/main.tf#L301-L336)
- There is a module that was created to fill this gap that provides a Kubernetes based approach to provision: https://github.com/aidanmelen/terraform-aws-eks-auth
- Ideally, one of the following issues are resolved upstream for a more native experience for users:
  - https://github.com/aws/containers-roadmap/issues/185
  - https://github.com/hashicorp/terraform-provider-kubernetes/issues/723

### I received an error: `Error: Invalid for_each argument ...`

Users may encounter an error such as `Error: Invalid for_each argument - The "for_each" value depends on resource attributes that cannot be determined until apply, so Terraform cannot predict how many instances will be created. To work around this, use the -target argument to first apply ...`

This error is due to an upstream issue with [Terraform core](https://github.com/hashicorp/terraform/issues/4149). There are two potential options you can take to help mitigate this issue:

1. Create the dependent resources before the cluster => `terraform apply -target <your policy or your security group>` and then `terraform apply` for the cluster (or other similar means to just ensure the referenced resources exist before creating the cluster)

- Note: this is the route users will have to take for adding additional security groups to nodes since there isn't a separate "security group attachment" resource

2. For additional IAM policies, users can attach the policies outside of the cluster definition as demonstrated below

```hcl
resource "aws_iam_role_policy_attachment" "additional" {
  for_each = module.eks.eks_managed_node_groups
  # you could also do the following or any combination:
  # for_each = merge(
  #   module.eks.eks_managed_node_groups,
  #   module.eks.self_managed_node_group,
  #   module.eks.fargate_profile,
  # )

  #            This policy does not have to exist at the time of cluster creation. Terraform can
  #            deduce the proper order of its creation to avoid errors during creation
  policy_arn = aws_iam_policy.node_additional.arn
  role       = each.value.iam_role_name
}
```

TL;DR - Terraform resource passed into the modules map definition _must_ be known before you can apply the EKS module. The variables this potentially affects are:

- `cluster_security_group_additional_rules` (i.e. - referencing an external security group resource in a rule)
- `node_security_group_additional_rules` (i.e. - referencing an external security group resource in a rule)
- `iam_role_additional_policies` (i.e. - referencing an external policy resource)

- Setting `instance_refresh_enabled = true` will recreate your worker nodes without draining them first. It is recommended to install [aws-node-termination-handler](https://github.com/aws/aws-node-termination-handler) for proper node draining. See the [instance_refresh](https://github.com/terraform-aws-modules/terraform-aws-eks/tree/master/examples/irsa_autoscale_refresh) example provided.

### Why are nodes not being registered?

Nodes not being able to register with the EKS control plane is generally due to networking mis-configurations.

1. At least one of the cluster endpoints (public or private) must be enabled.

If you require a public endpoint, setting up both (public and private) and restricting the public endpoint via setting `cluster_endpoint_public_access_cidrs` is recommended. More info regarding communication with an endpoint is available [here](https://docs.aws.amazon.com/eks/latest/userguide/cluster-endpoint.html).

2. Nodes need to be able to contact the EKS cluster endpoint. By default, the module only creates a public endpoint. To access the endpoint, the nodes need outgoing internet access:

- Nodes in private subnets: via a NAT gateway or instance along with the appropriate routing rules
- Nodes in public subnets: ensure that nodes are launched with public IPs (enable through either the module here or your subnet setting defaults)

**Important: If you apply only the public endpoint and configure the `cluster_endpoint_public_access_cidrs` to restrict access, know that EKS nodes will also use the public endpoint and you must allow access to the endpoint. If not, then your nodes will fail to work correctly.**

3. The private endpoint can also be enabled by setting `cluster_endpoint_private_access = true`. Ensure that VPC DNS resolution and hostnames are also enabled for your VPC when the private endpoint is enabled.

4. Nodes need to be able to connect to other AWS services to function (download container images, make API calls to assume roles, etc.). If for some reason you cannot enable public internet access for nodes you can add VPC endpoints to the relevant services: EC2 API, ECR API, ECR DKR and S3.

### Why are there no changes when a node group's `desired_size` is modified?

The module is configured to ignore this value. Unfortunately, Terraform does not support variables within the `lifecycle` block. The setting is ignored to allow autoscaling via controllers such as cluster autoscaler or Karpenter to work properly and without interference by Terraform. Changing the desired count must be handled outside of Terraform once the node group is created.

### How can I deploy Windows based nodes?

To enable Windows support for your EKS cluster, you will need to apply some configuration manually. See the [Enabling Windows Support (Windows/MacOS/Linux)](https://docs.aws.amazon.com/eks/latest/userguide/windows-support.html#enable-windows-support).

In addition, Windows based nodes require an additional cluster RBAC role (`eks:kube-proxy-windows`).

Note: Windows based node support is limited to a default user data template that is provided due to the lack of Windows support and manual steps required to provision Windows based EKS nodes.

### How do I access compute resource attributes?

Examples of accessing the attributes of the compute resource(s) created by the root module are shown below. Note - the assumption is that your cluster module definition is named `eks` as in `module "eks" { ... }`:

````hcl

- EKS Managed Node Group attributes

```hcl
eks_managed_role_arns = [for group in module.eks_managed_node_group : group.iam_role_arn]
````

- Self Managed Node Group attributes

```hcl
self_managed_role_arns = [for group in module.self_managed_node_group : group.iam_role_arn]
```

- Fargate Profile attributes

```hcl
fargate_profile_pod_execution_role_arns = [for group in module.fargate_profile : group.fargate_profile_pod_execution_role_arn]
```
