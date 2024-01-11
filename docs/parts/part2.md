<!-- See https://squidfunk.github.io/mkdocs-material/reference/ -->
# Part 2: Kubernetes Deployment and Configuration

The following steps require that you be familiar with HashiCorp Terraform modules. For more info, please read [Terraform's documentation](https://developer.hashicorp.com/terraform/tutorials/modules/module). The steps below will guide you to deploy Kubernetes with a Terraform module in an automated fashion.

## Steps

### 1. Clone the terraform-equinix-kubernetes-cluster repo

For this workshop, we will clone the repository [`terraform-equinix-kubernetes-cluster`](https://github.com/equinix-labs/terraform-equinix-kubernetes-cluster), to provision a kubernetes cluster. This repository contains a Terraform module that you will run to provision the cluster.

```shell
$ git clone https://github.com/equinix-labs/terraform-equinix-kubernetes-cluster.git
$ cd terraform-equinix-kubernetes-cluster
```

### 2. Customize the Terraform Deployment

Now that we have cloned the kubernetes repo, we will need to customize our deployment. Let's first create a subfolder for our custom deployment under the `examples` folder in the repo and call it `my-deployment`.

```shell
$ cd examples && mkdir my-deployment && cd my-deployment
```

Let's build the base terraform plan with the required terraform project's files: `main.tf`, `outputs.tf`, `README.md`, `variables.tf`, `versions.tf`, and `terraform.tfvars`. For more info, see [`Part 2: Create a Terraform project`](https://equinix-labs.github.io/terraform-on-equinix-workshop/parts/create-project/) from the `Terraform on Equinix` workshop.

```shell
$ cp ../../variables.tf . && cp ../../versions.tf .
$ touch README.md main.tf terraform.tfvars
```

To instantiate the root module of the repo, we will need to create a `module` block in the `main.tf` file. We also need to add the corresponding Equinix `provider` block that is used in the root module.

```terraform
module "tfk8s" {
  source = "../.."

  metal_auth_token        = var.metal_auth_token
  kube_vip_version        = var.kube_vip_version
  kubernetes_version      = var.kubernetes_version
  metal_project_id        = var.metal_project_id
  cp_ha                   = var.cp_ha
  worker_host_count       = var.worker_host_count
  metal_metro             = var.metal_metro
  cloud_provider_external = var.cloud_provider_external
}

provider "equinix" {
  auth_token = var.metal_auth_token
}
```

Now we need to assign the variables used in the `module` block from the previous step with attributes it needs to run. There are multiple ways to accomplish that. For demonstration, we'll populate the `terraform.tfvars` file with the variable assignments. For more info on `terraform.tfvars`, see [`Assign values with a file`](https://developer.hashicorp.com/terraform/tutorials/configuration-language/variables#assign-values-with-a-file) documentation. For other options, see [`Customize Terraform configuration with variables`](https://developer.hashicorp.com/terraform/tutorials/configuration-language/variables)

Remember to assign the `metal_auth_token` and the `metal_project_id` variables with the values obtained from Part 1 of the workshop.

```terraform
metal_auth_token        = "your_token_here" #This must be a user API token
metal_project_id        = "your_project_id"
ssh_private_key_path    = "path_to_your_private_ssh_key"
kube_vip_version        = "v0.5.9"
kubernetes_version      = "v1.26.1"
metal_metro             = "da"
cloud_provider_external = false
```

Lastly, let's build populate the `outputs.tf` file. This file contains the terraform code that will generate the output values of the terraform plan:

```terraform
output "tfk8s_outputs" {
  description = "Outputs of the tfk8s module"
  value = {
    kubeip_vip       = module.tfk8s.kubeapi_vip
    cloud_init_done  = module.tfk8s.cloud_init_done
    kubeconfig_ready = module.tfk8s.kubeconfig_ready
  }
}
```

> **_Note:_** You may build custom documentation in README.md but it does not necessarily need to be populated in order to provision infrastructure.

We are ready to run Terraform to provision the kubernetes cluster!

### 3. Provision Kubernetes

Now that we have finished building the terraform plan, we need to apply it. Let's take the same steps demonstrated in [`Part 3: Apply a Terraform Plan`](https://equinix-labs.github.io/terraform-on-equinix-workshop/parts/apply-plan/) of the the `Terraform on Equinix` workshop.

```shell
$ terraform init --upgrade
$ terraform plan
$ terraform apply -auto-approve
```

Once the terraform apply execution ends, you will see something like this:

```terraform
Apply complete! Resources: 16 added, 0 changed, 0 destroyed.

Outputs:

tfk8s_outputs = {
  "cloud_init_done" = "1915224144313739338"
  "kubeconfig_ready" = "8752730457495607904"
  "kubeip_vip" = "86.109.9.236"
}
```

### 4. Install and Configure the Kubernetes Command Line Tool: kubectl

There are many [tools](https://kubernetes.io/docs/tasks/tools/) available to manage a Kubernetes cluster. For demonstration, we will focus on `kubectl` as our Kubernetes CLI manager of choice.

To install `kubectl` on MacOS, please follow [these instructions](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/). Once installed, `kubectl` sets the default config file path to `$HOME/.kube/config`. We will populate this config file with the contents of the file `kubeconfig.admin.yaml` file generate at the root of the workspace folder `my-deployment` from the previous Step 3.

```shell
$ cat kubeconfig.admin.yaml > $HOME/.kube/config
```

Alternatively, you could reference the generated config file `kubeconfig.admin.yaml` directly to manage the cluster. For instance:

```shell
$ kubectl --kubeconfig kubeconfig.admin.yaml get svc
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   68m
```

Now that you have configured `kubectl`, let's verify its configuration:

```shell
$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://86.109.9.236:6443
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: DATA+OMITTED
    client-key-data: DATA+OMITTED
```

Note that the cluster server IP `86.109.9.236` matches the `kubeip_vip` output posted in the outputs of the terraform run completed in Step 3.

## Discussion

Before proceeding to the next part let's take a few minutes to discuss what we did. Here are some questions to start the discussion.

* How many ways can a Kubernetes cluster be managed?