# The purpose of this repository is to deploy an EKS cluster using Terraform. `kubectl` used Terraform output to deploy a Kubernetes dashboard on the cluster.

[HashiCorp Learn Guide](https://learn.hashicorp.com/tutorials/terraform/eks?in=terraform/kubernetes)

The Amazon Elastic Kubernetes Service (EKS) is the AWS service for deploying, managing, and scaling containerized applications with Kubernetes.

***Warning! AWS charges $0.10 per hour for each EKS cluster. 


# Prerequisites

1. an [AWS account](https://signin.aws.amazon.com/signin?redirect_uri=https%3A%2F%2Fportal.aws.amazon.com%2Fbilling%2Fsignup%2Fresume&client_id=signup&code_challenge_method=SHA-256&code_challenge=_lZzlGtCepKXO4raf3pqyiG4Ju41ZrDE1dlZYkzRzBw#/start) with the IAM permissions listed on the [EKS module documentation](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/latest)

2. Configured [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
Fro Mac OS:
```
$ brew install awscli
```
For Windows:
```
$ choco install awscli
```

3. AWS IAM Authenticator
Fro Mac OS:
```
$ brew install aws-iam-authenticator
```
For Windows:
```
$ choco install aws-iam-authenticator
```

4. [kubectl](https://learn.hashicorp.com/tutorials/terraform/eks?in=terraform/kubernetes#kubectl)
Fro Mac OS:
```
$ brew install kubernetes-cli
```
For Windows:
```
$ choco install kubernetes-cli
```


5. wget (required for the eks module)
Fro Mac OS:
```
$ brew install wget
```
For Windows:
```
$ choco install wget
```

6. [Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli) -version used 


# Set up and initialize your Terraform workspace
clone the repo:
```
$ git clone https://github.com/krasteki/terraform-provision-eks-cluster.git
$ cd terraform-provision-eks-cluster
```

1. `vpc.tf` - provisions a VPC, subnets and availability zones using the [AWS VPC Module](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/2.32.0). A new VPC will be created.
2. `security-groups.tf` - provisions the security groups used by the EKS cluster.
3. `eks-cluster.tf` - provisions all the resources (AutoScaling Groups, etc...) required to set up an EKS cluster using the [AWS EKS Module](https://registry.terraform.io/modules/terraform-aws-modules/eks/aws/11.0.0).

4. `outputs.tf` - defines the output configuration.
5. `versions.tf` - sets the Terraform version to at least 0.14. It also sets versions for the providers used in this sample.

# Initialize Terraform workspace and Provision the EKS cluster
```
$ terraform init
$ terraform apply
```

# Configure kubectl
```
$ aws eks --region $(terraform output -raw region) update-kubeconfig --name $(terraform output -raw cluster_name)
```

The `Kubernetes cluster name` and `region` correspond to the output variables showed after the successful Terraform run.

# Deploy and access Kubernetes Dashboard

1. Deploy Kubernetes Metrics Server
The Kubernetes Metrics Server, used to gather metrics such as cluster CPU and memory usage over time, is not deployed by default in EKS clusters.
```
$ wget -O v0.3.6.tar.gz https://codeload.github.com/kubernetes-sigs/metrics-server/tar.gz/v0.3.6 && tar -xzf v0.3.6.tar.gz
```

Deploy the metrics server to the cluster by running the following command.
```
$ kubectl apply -f metrics-server-0.3.6/deploy/1.8+/
```

Verify that the metrics server has been deployed
```
$ kubectl get deployment metrics-server -n kube-system
```

2. Deploy Kubernetes Dashboard
The following command will schedule the resources necessary for the dashboard:
```
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml
```

Proxy is needed:
```
$ kubectl proxy
```

The k8s dashboard should be accessible:
http://127.0.0.1:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

3. Authenticate the dashboard
To use the Kubernetes dashboard, you need to create a `ClusterRoleBinding` and provide an authorization token. This gives the `cluster-admin` permission to access the `kubernetes-dashboard`

In another terminal (do not close the `kubectl proxy` process), create the `ClusterRoleBinding` resource.
```
$ kubectl apply -f https://raw.githubusercontent.com/krasteki/terraform-provision-eks-cluster/main/kubernetes-dashboard-admin.rbac.yaml
```
Generate the authorization token:
```
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep service-controller-token | awk '{print $1}')
```