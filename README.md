# SWE execution in local environment

This documentation demonstrates the steps to test the developed IBM SWE gitops module locally by pointing to desired OpenShift cluster and GitHub repository.

## Setup

Clone your module repository. Go to `test/stages` directory:

1. Use `github.com/cloud-native-toolkit/terraform-tools-argocd-bootstrap.git` module for gitops-bootstrap purpose. 
   
**Note**: In the forked template code `github.com/cloud-native-toolkit/terraform-util-gitops-bootstrap` is used as they are using existing ArgoCD setup and not installing ArgoCD for every execution. This change will be helpful to automate ArgoCD installation in our target cluster and to test our developed module. **Please don't commit this change back to repo**.

   Ex: Change `stage1-gitops-bootstrap.tf` file content to
   ```
   module "gitops-bootstrap" {
    source = "github.com/cloud-native-toolkit/terraform-tools-argocd-bootstrap.git" 
    cluster_type = module.dev_cluster.platform.type_code
    ingress_subdomain = module.dev_cluster.platform.ingress
    cluster_config_file = module.dev_cluster.config_file_path
    olm_namespace = module.olm.olm_namespace
    operator_namespace = module.olm.target_namespace
    gitops_repo_url = module.gitops.config_repo_url
    git_username = module.gitops.config_username
    git_token = module.gitops.config_token
    bootstrap_path = module.gitops.bootstrap_path
    sealed_secret_cert = module.cert.cert
    sealed_secret_private_key = module.cert.private_key
    bootstrap_prefix = var.bootstrap_prefix
    create_webhook = true
  }
 ```

2. Add the olm module (`github.com/cloud-native-toolkit/terraform-k8s-olm`) as `stage-1-olm.tf` the  `terraform-tools-argocd-bootstrap` module needs it.
 
 Add `stage1-olm.tf` file with following content:
 ```
 module "olm" {
    source = "github.com/cloud-native-toolkit/terraform-k8s-olm" 
    cluster_config_file = module.dev_cluster.config_file_path
    cluster_type = module.dev_cluster.platform.type_code
    cluster_version = module.dev_cluster.platform.version
  }
 ```

3. Refer all the variables being passed to all modules in all `stage-1-*.tf` files and also `stage-2-*.tf` files. Many variables are intialized with other module's reference variables. Some would need direct inputs. These variables which need input are defined in `varaibles.tf` file in that directory. If there is no default values initialized for a varaible in varaibles.tf file, or if you want to override the default values mentioned in `variables.tf` file, create a terraform.tfvars file and intialize all requried variables. 

 Create `terraform.tfvars` file in test/stages directory. An example file will have following variables. Initialize with appropriate values required for your module setup.
```
ibmcloud_api_key=""
server_url=""
namespace=""
login_token=""
git_org=""
git_repo=""
git_username=""
git_token=""
cp_entitlement_key=""
```

In `terraform.tfvars` file make sure to give all details pertaining to YOUR cluster, git repository and IBM Cloud API key of the target cloud account.

4. In `stage1-cluster.tf` file you may initialize `login_token` value directly or initialize it with `var.login_token` and define `login_token` variable in variable.tf and then initialize it in `terraform.tfvars` 
```
module "dev_cluster" {
  source = "github.com/cloud-native-toolkit/terraform-ocp-login.git"

  server_url = var.server_url
  login_user = "apikey"
  login_password = var.ibmcloud_api_key
  login_token = var.login_token # added login_token in variable.tf and initialized in terraform.tfvars file.
}
```

5. Create a symbolic link to current module code execution (stage2-mymodule.tf) by running following command in command prompt in test/stages directory:
```
ln -s ../.. module
```

6. If this is second time execution, remove `.terraform` `.tmp` `.tmpgitops` etc directories and `terraform.tfstate` `terraform.tfstate.backup` files. 

**Note**: By default the github repo mentioned (`git_repo` variable in `terraform.tfvars`) will be created hence should not exist. Only github org should exist.

Now we are set to execute!!

## Execution Steps: 

To be run in test/stages directory:

1. `terraform init`

This will download all required modules in .terraform directory inside test/stages.

2. `terraform plan`

This will make a dry-run and provides a list of resources to be added/destroyed and also any syntax/reference errors.

If we get the output with list of resources to be added, we can run next command to actually create those resources.

3. `terraform apply --auto-approve`

This command will provision resources in mentioned cluster.

The mentioned git repoository will be populated with all requried argocd config yamls and payload yamls and in OpenShift cluster we can verfiy openshift-giops, sealed-secrets, openshift-piplines etc namespaces getting created and then requried operators getting installed. 

Details like argocd password can be found in file `test/stages/.tmp/argocd-password.val` etc.

## Destroy 

`terraform destroy` 

This will remove all the ArgoCD deployments and github repo created by the automation. After `terraform destroy` is completed, to make OpenShift cluster reusable, further cleaning can be done using following steps in OpenShift console:

1. Delete the namespaces created by script.

2. Uninstall all the operators installed by automation Ex: openshift-pipeline, openshift-gitops etc.

3. Go to Administration -> CRDs -> namespacescopes -> Instances Tab -> Click on each item and in yaml remove the entry under finalizers

Ex:
```
  finalizers:
    - finalizer.nss.operator.ibm.com 
```
 In the above example remove line `- finalizer.nss.operator.ibm.com`

 4. Make sure namespaces get deleted and not in "Terminating" state anymore. If it continues to be in "Terminating" state, examine namespace yaml and check the finalizer holiding that.

## Troubleshooting

 If facing following error 

 `yq4 is not an executable binary`

 Install yq in your system by running `brew install yq`
 Then run `which mq` to find the instalaltion location.
 Then copy the yq executable inside bin2 dir of test/stages. `cp /usr/local/bin/yq bin2/`
 Execute `./bin2/yq4` and verify that it is allowed to execute, change system preferences if required.
 
 ## Authors: 
 - Gowdhaman Jayaseelan (gjayasee@in.ibm.com)
 - Divya Kamath (dimallya@in.ibm.com
