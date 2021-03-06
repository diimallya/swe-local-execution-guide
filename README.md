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

2. Add the olm module (`github.com/cloud-native-toolkit/terraform-k8s-olm`) in `stage-1-olm.tf`, as the `terraform-tools-argocd-bootstrap` module (added in previous step) needs it.
 
 Add `stage1-olm.tf` file with following content:
 ```
 module "olm" {
    source = "github.com/cloud-native-toolkit/terraform-k8s-olm" 
    cluster_config_file = module.dev_cluster.config_file_path
    cluster_type = module.dev_cluster.platform.type_code
    cluster_version = module.dev_cluster.platform.version
  }
 ```

3. Refer all the variables being passed to all modules in all `stage-1-*.tf` files and also `stage-2-*.tf` files. Many variables are intialized with other module's reference variables. Some would need direct inputs. These variables which need input are defined in `varaibles.tf` file in the `test/stages` directory. If there is no default values initialized for a varaible in `varaibles.tf` file, or if you want to override the default values mentioned in `variables.tf` file, create a `terraform.tfvars` file in `test/stages` directory and intialize all requried variables. 

 Create `terraform.tfvars` file in `test/stages` directory. An example file will have following variables. Initialize with appropriate values required for all modules in `stage-1-*.tf` and also your module present in `stage-2-*.tf` file.
```
cluster_username="apikey"                 # default username for any OCP cluster
cluster_password="xxxxx"                  # IBM Cloud account API Key
server_url="https://xxxxxxx:xxxxxxx"      # OCP cluster URL
namespace="xxxxx"                         # name of namespace to be created for your module
cluster_type = "openshift"                # for OCP cluster value is "openshift"
git_org="testorg"                         # github org name of your account
git_repo="gitops-test-repo"               # github repo name which is not existing 
git_username="xxxxxx"                     # github user name
git_token="xxx_xxxxxx"                    # Personal Access Token for our github repo.
cp_entitlement_key="xxxxxxxxxxxxxxxxxx"   # Entitlement key value
```

While initializing `terraform.tfvars` file make sure to give all details pertaining to YOUR cluster, git repository and IBM Cloud API key of the cloud account, where target cluster is available.

4. In `stage1-cluster.tf` file you can initialize `login_token` value directly. This is the login token for target OCP cluster, obtained from `oc login` command.
```
module "dev_cluster" {
  source = "github.com/cloud-native-toolkit/terraform-ocp-login.git"

  server_url = var.server_url
  login_user = var.cluster_username
  login_password = var.cluster_password
  login_token = "xxxxxxxxxxx" # OCP cluster login token obtained from "oc login" command
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

You can choose to run execution from either of the following methods:
- Execution using **cli-tools** docker image <br/>
**OR** <br/>
- Executing locally on your system.
### Execution using cli-tools docker image ###

1. Run the following command from module's root directory (Ex: `$HOME/terraform-gitops-module`) to start running docker container:
```
docker run -it \
  -v ${PWD}:/terraform \
  -w /terraform \
  quay.io/ibmgaragecloud/cli-tools:v1.1
```
This will mount the present working directory inside the docker container at `/terraform` path. 
 
2. Inside docker run `ls` command to make sure all contents in module root directory are visible. <br/><br/>
   **Note**: <br/>
  - Sometimes ${PWD} is not mounted inside docker container, you may try giving the absolute path as below.
 ```
docker run -it \
  -v /Users/divyakamath/terraform-gitops-module:/terraform \
  -w /terraform \
quay.io/ibmgaragecloud/cli-tools:v1.1

```  
  - Sometimes due to permission issues present working directory contents are not mounted inside docker even if absolute path is given. Make sure module directory is placed in a user-created directory under $HOME rather than default directories like `$HOME/Documents` etc.
   
3. Navigate to `test/stages` direcotry and execute: 

`sudo terraform init`

This will download all required modules in .terraform directory inside test/stages. As you may encounter permission issue, to create .terraform directory, it can be executed with sudo prefix.

4. `sudo terraform plan`

This will make a dry-run and provides a list of resources to be added/destroyed and also any syntax/reference errors. If we get the output with list of resources to be added, we can run next command to actually create those resources.

5. `sudo terraform apply --auto-approve`

This command will provision resources in mentioned cluster.

The mentioned git repoository will be populated with all requried argocd config yamls and payload yamls and in OpenShift cluster we can verfiy openshift-giops, sealed-secrets, openshift-piplines etc namespaces getting created and then requried operators getting installed. 

Details like argocd password can be found in file `test/stages/.tmp/argocd-password.val` etc.

### Executing locally on your system ###

Please follow steps provided in [Troubleshooting scetion](https://github.com/diimallya/swe-local-execution-guide#troubleshooting) to download required executables compatible with your OS before proceeding with next steps. 

To be run in `test/stages` directory:

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

1. Error `yq4 is not an executable binary` while doing terraform apply.

   Steps to resovle:
    - Install yq in your system by running following commnad:
   
      `brew install yq`
    -  Find the instalaltion location of `yq` using following command:
    
       `which yq`
    - Copy the yq executable inside bin2 dir of `test/stages`.
     
      `cp /usr/local/bin/yq ./bin2/yq4`
    
    - Execute `./bin2/yq4` and verify that it is allowed to execute, change system preferences if required. If the binary launch is giving expected output then it can be considered to be ready for execution during `terraform apply`
    ```
    stages$ ./bin2/yq4
      Usage:
        yq [flags]
        yq [command]
      ....
     ```


2. The secret is not get created in OpenShift cluster due to `kubeseal` execution error.

   **Note:** This error doesn't stop the execution during terraform apply, however error gets logged and the execution continues, but secret doesn't get created.
   
   Steps to resolve:
   
    - Download the kubeseal executable compatible with your OS.
    
   Link to download kubeseal for MAC: https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.17.3/kubeseal-0.17.3-darwin-amd64.tar.gz
   
   - Extract and copy the kubeseal executable in `test/stages/bin` directory in your module folder.
    
      `cp kubeseal ./bin/`
      
   - Execute `./bin/kubeseal` and verify that it is allowed to execute, change system preferences if required.
   - Also place kubeseal executable in `/usr/local/bin/` directory.
 
 ## Authors: 
 - Gowdhaman Jayaseelan (gjayasee@in.ibm.com)
 - Divya Kamath (dimallya@in.ibm.com)
