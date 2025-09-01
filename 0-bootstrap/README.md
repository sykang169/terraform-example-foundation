# 0-bootstrap

이 리포지토리는 [Google Cloud 보안 기반 가이드](https://cloud.google.com/architecture/security-foundations)에서 설명된
example.com 참조 아키텍처를 구성하고 배포하는 방법을 보여주는 다중 파트 가이드의 일부입니다. 다음 표는 이 배포의 단계들을 나열합니다.

<table>
<tbody>
<tr>
<td>0-bootstrap (이 파일)</td>
<td>Google Cloud 조직을 부트스트래핑하여 Cloud Foundation Toolkit(CFT) 사용을 시작하는 데 필요한 모든 리소스와
권한을 생성합니다. 이 단계는 또한 후속 단계에서 기반 코드를 위한 <a href="../docs/GLOSSARY.md#foundation-cicd-pipeline">CI/CD 파이프라인</a>을 구성합니다.</td>
</tr>
<tr>
<td><a href="../1-org">1-org</a></td>
<td>최상위 수준 공유 폴더, 네트워킹 프로젝트 및 조직 수준 로깅을 설정하고, 
조직 정책을 통해 기준선 보안 설정을 구성합니다.</td>
</tr>
<tr>
<td><a href="../2-environments"><span style="white-space: nowrap;">2-environments</span></a></td>
<td>생성한 Google Cloud 조직 내에서 개발, 비운영, 운영 환경을 설정합니다.</td>
</tr>
<tr>
<td><a href="../3-networks-svpc">3-networks-svpc</a></td>
<td>각 환경에 대해 기본 DNS, NAT(선택사항), 
Private Service 네트워킹, VPC 서비스 컨트롤, 온프레미스 Dedicated
Interconnect, 기준선 방화벽 규칙이 포함된 공유 VPC를 설정합니다. 또한
글로벌 DNS 허브를 설정합니다.</td>
</tr>
<tr>
<td><a href="../3-networks-hub-and-spoke">3-networks-hub-and-spoke</a></td>
<td>3-networks-svpc 단계에서 찾을 수 있는 모든 기본 구성이 포함된 공유 VPC를 설정하지만,
여기서는 아키텍처가 Hub and Spoke 네트워크 모델을 기반으로 합니다. 또한
글로벌 DNS 허브를 설정합니다.</td>
</tr>
</tr>
<tr>
<td><a href="../4-projects">4-projects</a></td>
<td>이전 단계에서 생성된 공유 VPC에 서비스 프로젝트로 연결되는 애플리케이션을 위한
폴더 구조, 프로젝트 및 애플리케이션 인프라 파이프라인을 설정합니다.</td>
</tr>
<tr>
<td><a href="../5-app-infra">5-app-infra</a></td>
<td>4-projects에서 설정한 인프라 파이프라인을 사용하여 비즈니스 유닛 프로젝트 중 하나에 <a href="https://cloud.google.com/compute/">Compute Engine</a> 인스턴스를 배포합니다.</td>
</tr>
</tbody>
</table>

아키텍처와 부분들에 대한 개요는
[terraform-example-foundation README](https://github.com/terraform-google-modules/terraform-example-foundation)
파일을 참조하세요.

## 목적

이 단계의 목적은 Google Cloud 조직을 부트스트래핑하여 Cloud Foundation Toolkit(CFT) 사용을 시작하는 데 필요한 모든 리소스와 권한을 생성하는 것입니다. 이 단계는 또한 후속 단계에서 기반 코드를 위한 [CI/CD 파이프라인](/docs/GLOSSARY.md#foundation-cicd-pipeline)을 구성합니다. [CI/CD 파이프라인](/docs/GLOSSARY.md#foundation-cicd-pipeline)은 Cloud Build와 Cloud Source Repos 또는 Jenkins와 사용자 자체 Git 리포지토리(온프레미스에 있을 수 있음) 중 하나를 사용할 수 있습니다.

## 사용 목적 및 지원

이 리포지토리는 사용자의 자체 버전 제어 시스템에서 포크, 수정 및 유지 관리할 수 있는 예제로 제공됩니다. 이 리포지토리 내의 모듈들은 원격 참조로 사용하기 위한 것이 아닙니다.
이 블루프린트가 기반 설계 및 구축을 가속화하는 데 도움이 될 수 있지만, 사용자는 자신의 요구 사항에 따라 자체 기반을 배포하고 사용자 정의할 수 있는 엔지니어링 기술과 팀을 갖추고 있다고 가정합니다.

지원 항목:

- 코드는 의미적으로 유효하고, 알려진 좋은 버전에 고정되며, terraform validate 및 lint 검사를 통과합니다
- 이 리포지토리에 대한 모든 PR은 병합되기 전에 테스트 환경에 모든 리소스를 배포하는 통합 테스트를 통과해야 합니다
- 코드 사용 편의성에 대한 기능 요청이나 모든 사용자에게 일반적으로 적용되는 기능 요청은 환영합니다

지원하지 않는 항목:

- 이전 버전으로 배포된 기반에서 최신 버전으로의 제자리 업그레이드는 마이너 버전 변경의 경우에도 실현 가능하지 않을 수 있습니다. 리포지토리 관리자는 사용자가 기반 위에 배포하는 리소스나 배포 시 기반이 어떻게 사용자 정의되었는지 알 수 없으므로, 주요 변경 사항을 피한다는 보장을 하지 않습니다.
- 단일 사용자의 요구 사항에 특화되고 일반적인 모범 사례를 대표하지 않는 기능 요청

## 전제 조건

이 문서에서 설명하는 명령을 실행하려면 다음을 설치하세요:

- [Google Cloud SDK](https://cloud.google.com/sdk/install) 버전 393.0.0 이상
- [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) 버전 2.28.0 이상
- [Terraform](https://www.terraform.io/downloads.html) 버전 1.5.7
- [jq](https://jqlang.github.io/jq/download/) 버전 1.6.0 이상

**참고:** 이 시리즈 전체에서 동일한 [Terraform](https://www.terraform.io/downloads.html) 버전을 사용해야 합니다. 그렇지 않으면 Terraform 상태 스냅샷 잠금 오류가 발생할 수 있습니다.

버전 1.5.7은 라이선스 모델 변경 이전의 마지막 버전입니다. Terraform의 최신 버전을 사용하려면 `3-networks`와 `4-projects`의 일부 단계를 수동으로 실행하는 운영 체제에서 사용되는 Terraform 버전이 다음 코드에서 구성된 버전과 동일한지 확인하세요

- 0-bootstrap/modules/jenkins-agent/variables.tf

   ```
   default     = "1.5.7"
   ```

- 0-bootstrap/cb.tf

   ```
   terraform_version = "1.5.7"
   ```

- scripts/validate-requirements.sh

   ```
   TF_VERSION="1.5.7"
   ```

- build/github-tf-apply.yaml

   ```
   terraform_version: '1.5.7'
   ```

- github-tf-pull-request.yaml

   ```
   terraform_version: "1.5.7"
   ```

- 0-bootstrap/Dockerfile

   ```
   ARG TERRAFORM_VERSION=1.5.7
   ```

또한 다음 사항을 완료했는지 확인하세요:

1. Google Cloud
   [조직](https://cloud.google.com/resource-manager/docs/creating-managing-organization)을 설정합니다.
1. Google Cloud
   [청구 계정](https://cloud.google.com/billing/docs/how-to/manage-billing-account)을 설정합니다.
1. [액세스 제어를 위한 그룹](https://cloud.google.com/architecture/security-foundations/authentication-authorization#groups_for_access_control)에서 정의한 대로 Cloud Identity 또는 Google Workspace 그룹을 생성합니다.
생성한 특정 그룹 이름을 사용하도록 **terraform.tfvars**의 변수(`groups` 블록)를 설정합니다.
1. 이 문서의 절차를 실행할 사용자에게 다음 역할을 부여합니다:
   - Google Cloud 조직에서 `roles/resourcemanager.organizationAdmin` 역할
   - Google Cloud 조직에서 `roles/orgpolicy.policyAdmin` 역할
   - Google Cloud 조직에서 `roles/resourcemanager.projectCreator` 역할
   - 청구 계정에서 `roles/billing.admin` 역할
   - `roles/resourcemanager.folderCreator` 역할
   - `roles/securitycenter.admin` 역할

     ```bash
     # example:
     gcloud organizations add-iam-policy-binding ${ORG_ID}  --member=user:$SUPER_ADMIN_EMAIL --role=roles/securitycenter.admin --quiet > /dev/null 1>&1
     ```

1. Enable the following additional services on your current bootstrap project:

     ```bash
     gcloud services enable cloudresourcemanager.googleapis.com
     gcloud services enable cloudbilling.googleapis.com
     gcloud services enable iam.googleapis.com
     gcloud services enable cloudkms.googleapis.com
     gcloud services enable servicenetworking.googleapis.com
     ```

### Optional - Automatic creation of Google Cloud Identity groups

In the foundation, Google Cloud Identity groups are used for [authentication and access management](https://cloud.google.com/architecture/security-foundations/authentication-authorization) .

To enable automatic creation of the [groups](https://cloud.google.com/architecture/security-foundations/authentication-authorization#groups_for_access_control), complete the following actions:

- Have an existing project for Cloud Identity API billing.
- Enable the Cloud Identity API (`cloudidentity.googleapis.com`) on the billing project.
- Grant role `roles/serviceusage.serviceUsageConsumer` to the user running Terraform on the billing project.
- Change the field `groups.create_required_groups` to **true** to create the required groups.
- Change the field `groups.create_optional_groups` to **true** and fill the `groups.optional_groups` with the emails to be created.

### Optional - Cloud Build access to on-prem

See [onprem](./onprem.md) for instructions on how to configure Cloud Build access to your on-premises environment.

### Troubleshooting

See [troubleshooting](../docs/TROUBLESHOOTING.md) if you run into issues during this step.

## Deploying with Jenkins

*Warning: the guidance for deploying with Jenkins is no longer actively tested or maintained. While we have left the guidance available for users who prefer Jenkins, we make no guarantees about its quality, and you might be responsible for troubleshooting and modifying the directions.*

If you are using the `jenkins_bootstrap` sub-module, see [README-Jenkins](./README-Jenkins.md)
for requirements and instructions on how to run the 0-bootstrap step. Using
Jenkins requires a few manual steps, including configuring connectivity with
your current Jenkins manager (controller) environment.

## Deploying with GitHub Actions

If you are deploying using [GitHub Actions](https://docs.github.com/en/actions), see [README-GitHub.md](./README-GitHub.md)
for requirements and instructions on how to run the 0-bootstrap step.
Using GitHub Actions requires manual creation of the GitHub repositories used in each stage.

## Deploying with GitLab Pipelines

If you are deploying using [GitLab Pipelines](https://docs.gitlab.com/ee/ci/pipelines/), see [README-GitLab.md](./README-GitLab.md)
for requirements and instructions on how to run the 0-bootstrap step.
Using GitLab Pipeline requires manual creation of the GitLab projects (repositories) used in each stage.

## Deploying with Terraform Cloud

If you are deploying using [Terraform Cloud](https://developer.hashicorp.com/terraform/cloud-docs), see [README-Terraform-Cloud.md](./README-Terraform-Cloud.md)
for requirements and instructions on how to run the 0-bootstrap step.
Using Terraform Cloud requires manual creation of the GitHub repositories or GitLab projects used in each stage.

## Deploying with Cloud Build

*Warning: This method has a dependency on Cloud Source Repositories, which is [no longer available to new customers](https://cloud.google.com/source-repositories/docs). If you have previously used the CSR API in your organization then you can use this method, but a newly created organization will not be able to enable CSR and cannot use this deployment method. In that case, we recommend that you follow the directions for deploying locally, Github, Gitlab, or Terraform Cloud instead.*

The following steps introduce the steps to deploy with Cloud Build Alternatively, use the [helper script](../helpers/foundation-deployer/README.md) to automate all the stages of. Use the helper script when you want to rapidly create and destroy the entire organization for demonstration or testing purposes, without much customization at each stage.

1. Clone [terraform-example-foundation](https://github.com/terraform-google-modules/terraform-example-foundation) into your local environment and navigate to the `0-bootstrap` folder.

   ```bash
   git clone https://github.com/terraform-google-modules/terraform-example-foundation.git

   cd terraform-example-foundation/0-bootstrap
   ```

1. Rename `terraform.example.tfvars` to `terraform.tfvars` and update the file with values from your environment:

   ```bash
   mv terraform.example.tfvars terraform.tfvars
   ```

1. Use the helper script [validate-requirements.sh](../scripts/validate-requirements.sh) to validate your environment:

   ```bash
   ../scripts/validate-requirements.sh -o <ORGANIZATION_ID> -b <BILLING_ACCOUNT_ID> -u <END_USER_EMAIL>
   ```

   **Note:** The script is not able to validate if the user is in a Cloud Identity or Google Workspace group with the required roles.

1. Run `terraform init` and `terraform plan` and review the output.

   ```bash
   terraform init
   terraform plan -input=false -out bootstrap.tfplan
   ```

1. To  validate your policies, run `gcloud beta terraform vet`. For installation instructions, see [Install Google Cloud CLI](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install).

1. Run the following commands and check for violations:

   ```bash
   export VET_PROJECT_ID=A-VALID-PROJECT-ID
   terraform show -json bootstrap.tfplan > bootstrap.json
   gcloud beta terraform vet bootstrap.json --policy-library="../policy-library" --project ${VET_PROJECT_ID}
   ```

   *`A-VALID-PROJECT-ID`* must be an existing project you have access to. This is necessary because `gcloud beta terraform vet` needs to link resources to a valid Google Cloud Platform project.

1. Run `terraform apply`.

   ```bash
   terraform apply bootstrap.tfplan
   ```

1. Run `terraform output` to get the email address of the terraform service accounts that will be used to run manual steps for `shared` environments in steps `3-networks-svpc`, `3-networks-hub-and-spoke`, and `4-projects` and the state bucket that will be used by step 4-projects.

   ```bash
   export network_step_sa=$(terraform output -raw networks_step_terraform_service_account_email)
   export projects_step_sa=$(terraform output -raw projects_step_terraform_service_account_email)
   export projects_gcs_bucket_tfstate=$(terraform output -raw projects_gcs_bucket_tfstate)

   echo "network step service account = ${network_step_sa}"
   echo "projects step service account = ${projects_step_sa}"
   echo "projects gcs bucket tfstate = ${projects_gcs_bucket_tfstate}"
   ```

1. Run `terraform output` to get the ID of your Cloud Build project:

   ```bash
   export cloudbuild_project_id=$(terraform output -raw cloudbuild_project_id)
   echo "cloud build project ID = ${cloudbuild_project_id}"
   ```

1. Copy the backend and update `backend.tf` with the name of your Google Cloud Storage bucket for Terraform's state. Also update the `backend.tf` of all steps.

   ```bash
   export backend_bucket=$(terraform output -raw gcs_bucket_tfstate)
   echo "backend_bucket = ${backend_bucket}"

   export backend_bucket_projects=$(terraform output -raw projects_gcs_bucket_tfstate)
   echo "backend_bucket_projects = ${backend_bucket_projects}"

   cp backend.tf.example backend.tf
   cd ..

   for i in `find . -name 'backend.tf'`; do sed -i'' -e "s/UPDATE_ME/${backend_bucket}/" $i; done
   for i in `find . -name 'backend.tf'`; do sed -i'' -e "s/UPDATE_PROJECTS_BACKEND/${backend_bucket_projects}/" $i; done

   cd 0-bootstrap
   ```

1. Re-run `terraform init`. When you're prompted, agree to copy Terraform state to Cloud Storage.

   ```bash
   terraform init
   ```

1. (Optional) Run `terraform plan` to verify that state is configured correctly. You should see no changes from the previous state.
1. Clone the policy repo and copy contents of policy-library to new repo. Clone the repo at the same level of the `terraform-example-foundation` folder.

   ```bash
   cd ../..

   gcloud source repos clone gcp-policies --project=${cloudbuild_project_id}

   cd gcp-policies
   git checkout -b main
   cp -RT ../terraform-example-foundation/policy-library/ .
   ```

1. Commit changes and push your main branch to the policy repo.

   ```bash
   git add .
   git commit -m 'Initialize policy library repo'
   git push --set-upstream origin main
   ```

1. Navigate out of the repo.

   ```bash
   cd ..
   ```

1. Save `0-bootstrap` Terraform configuration to `gcp-bootstrap` source repository:

   ```bash
   gcloud source repos clone gcp-bootstrap --project=${cloudbuild_project_id}

   cd gcp-bootstrap
   git checkout -b plan
   mkdir -p envs/shared

   cp -RT ../terraform-example-foundation/0-bootstrap/ ./envs/shared
   cp ../terraform-example-foundation/build/cloudbuild-tf-* .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh

   git add .
   git commit -m 'Initialize bootstrap repo'
   git push --set-upstream origin plan
   ```

1. Continue with the instructions in the [1-org](../1-org/README.md) step.

**Note 1:** The stages after `0-bootstrap` use `terraform_remote_state` data source to read common configuration like the organization ID from the output of the `0-bootstrap` stage. They will [fail](../docs/TROUBLESHOOTING.md#error-unsupported-attribute) if the state is not copied to the Cloud Storage bucket.

**Note 2:** After the deploy, even if you did not receive the project quota error described in the [Troubleshooting guide](../docs/TROUBLESHOOTING.md#project-quota-exceeded), we recommend that you request 50 additional projects for the **projects step service account** created in this step.

## Running Terraform locally

The following steps will guide you through deploying without using Cloud Build.

1. Clone [terraform-example-foundation](https://github.com/terraform-google-modules/terraform-example-foundation) into your local environment and create to the `gcp-bootstrap` folder at the same level. Copy the `0-bootstrap` content and `.gitignore` to `gcp-bootstrap`.

   ```bash
   git clone https://github.com/terraform-google-modules/terraform-example-foundation.git

   mkdir gcp-bootstrap

   cp -R terraform-example-foundation/0-bootstrap/* gcp-bootstrap/

   cp terraform-example-foundation/.gitignore gcp-bootstrap
   ```

1. Navigate to `gcp-bootstrap` and initialize a local Git repository to manage versions locally. Then, Create the environment branches.

   ```bash
   cd gcp-bootstrap

   git init
   git commit -m "initialize empty directory" --allow-empty
   git checkout -b plan

   git checkout -b shared
   ```

1. Rename `terraform.example.tfvars` to `terraform.tfvars` and update the file with values from your environment:

   ```bash
   mv terraform.example.tfvars terraform.tfvars
   ```

1. Rename `cb.tf` to `cb.tf.example`:

   ```bash
   mv cb.tf cb.tf.example
   ```

1. Comment Cloud Build related outputs at `outputs.tf`.

1. In `sa.tf` file, comment out lines related to Cloud Build. Specifically, search for `cicd_project_iam_member` and comment out the corresponding module, as well as the "depends_on" meta-argument in any modules that depend on the commented module.

1. In `sa.tf` file, search for `local.cicd_project_id` and comment out the corresponding code.

1. Use the helper script [validate-requirements.sh](../scripts/validate-requirements.sh) to validate your environment:

   ```bash
   ../terraform-example-foundation/scripts/validate-requirements.sh -o <ORGANIZATION_ID> -b <BILLING_ACCOUNT_ID> -u <END_USER_EMAIL>
   ```

   **Note:** The script is not able to validate if the user is in a Cloud Identity or Google Workspace group with the required roles.

1. Run `terraform init` and `terraform plan` and review the output.

   ```bash
   git checkout plan
   terraform init
   terraform plan -input=false -out bootstrap.tfplan
   ```

1. Create a new folder called gcp-policies at the same directory level as the `terraform-example-foundation` folder. Initialize a Git repository, create a branch called `main`, and copy the contents of the `policy-library` directory from the `terraform-example-foundation` folder into the gcp-policies folder.

   ```bash
   cd ../

   mkdir gcp-policies

   cd gcp-policies
   git init
   git checkout -b main
   cp -RT ../terraform-example-foundation/policy-library/ .
   ```

1. Commit changes to the main branch of the policy repo. This way you can manage versions locally.

   ```bash
   git add .
   git commit -m 'Initialize policy library repo'
   ```

1. Navigate back to `gcp-bootstrap` repo.

   ```bash
   cd ../gcp-bootstrap
   ```

1. To  validate your policies, run `gcloud beta terraform vet`. For installation instructions, see [Install Google Cloud CLI](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install).

1. Run the following commands and check for violations:

   ```bash
   export VET_PROJECT_ID=A-VALID-PROJECT-ID
   terraform show -json bootstrap.tfplan > bootstrap.json
   gcloud beta terraform vet bootstrap.json --policy-library="$(pwd)/../gcp-policies" --project ${VET_PROJECT_ID}
   ```

   *`A-VALID-PROJECT-ID`* must be an existing project you have access to. This is necessary because `gcloud beta terraform vet` needs to link resources to a valid Google Cloud Platform project.

1. Commit validated code in plan branch.

   ```bash
   git add .
   git commit -m "Initial version os gcp-bootstrap."
   ```

1. Checkout `shared` branch and merge the `plan` branch into it. Then, Run `terraform apply`.

   ```bash
   git checkout shared
   git merge plan

   terraform apply bootstrap.tfplan
   ```

1. Run `terraform output` to get the email address of the terraform service accounts that will be used to run steps manually and the state bucket that will be used by step `4-projects`.

   ```bash
   export network_step_sa=$(terraform output -raw networks_step_terraform_service_account_email)
   export projects_step_sa=$(terraform output -raw projects_step_terraform_service_account_email)
   export projects_gcs_bucket_tfstate=$(terraform output -raw projects_gcs_bucket_tfstate)

   echo "network step service account = ${network_step_sa}"
   echo "projects step service account = ${projects_step_sa}"
   echo "projects gcs bucket tfstate = ${projects_gcs_bucket_tfstate}"
   ```

1. Copy the backend and update `backend.tf` with the name of your Google Cloud Storage bucket for Terraform's state. Also update the `backend.tf` of all steps.

   ```bash
   export backend_bucket=$(terraform output -raw gcs_bucket_tfstate)
   echo "backend_bucket = ${backend_bucket}"

   export backend_bucket_projects=$(terraform output -raw projects_gcs_bucket_tfstate)
   echo "backend_bucket_projects = ${backend_bucket_projects}"

   cp backend.tf.example backend.tf

   cd ../

   for i in `find . -name 'backend.tf'`; do sed -i'' -e "s/UPDATE_ME/${backend_bucket}/" $i; done
   for i in `find . -name 'backend.tf'`; do sed -i'' -e "s/UPDATE_PROJECTS_BACKEND/${backend_bucket_projects}/" $i; done

   cd gcp-bootstrap
   ```

1. Re-run `terraform init`. When you're prompted, agree to copy Terraform state to Cloud Storage.

   ```bash
   terraform init
   ```

1. Commit the new code version, so you can manage versions locally.

   ```sh
   git add backend.tf
   git commit -m "Init gcs backend."
   cd ../
   ```

<!-- BEGINNING OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| attribute\_condition | Workload Identity Pool Provider attribute condition expression. [More info](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/iam_workload_identity_pool_provider#attribute_condition) | `string` | `null` | no |
| billing\_account | The ID of the billing account to associate projects with. | `string` | n/a | yes |
| bucket\_force\_destroy | When deleting a bucket, this boolean option will delete all contained objects. If false, Terraform will fail to delete buckets which contain objects. | `bool` | `false` | no |
| bucket\_prefix | Name prefix to use for state bucket created. | `string` | `"bkt"` | no |
| bucket\_tfstate\_kms\_force\_destroy | When deleting a bucket, this boolean option will delete the KMS keys used for the Terraform state bucket. | `bool` | `false` | no |
| default\_region | Default region to create resources where applicable. | `string` | `"us-central1"` | no |
| default\_region\_2 | Secondary default region to create resources where applicable. | `string` | `"us-west1"` | no |
| default\_region\_gcs | Case-Sensitive default region to create gcs resources where applicable. | `string` | `"US"` | no |
| default\_region\_kms | Secondary default region to create kms resources where applicable. | `string` | `"us"` | no |
| folder\_deletion\_protection | Prevent Terraform from destroying or recreating the folder. | `string` | `true` | no |
| folder\_prefix | Name prefix to use for folders created. Should be the same in all steps. | `string` | `"fldr"` | no |
| groups | Contain the details of the Groups to be created. | <pre>object({<br>    create_required_groups = optional(bool, false)<br>    create_optional_groups = optional(bool, false)<br>    billing_project        = optional(string, null)<br>    required_groups = object({<br>      group_org_admins     = string<br>      group_billing_admins = string<br>      billing_data_users   = string<br>      audit_data_users     = string<br>    })<br>    optional_groups = optional(object({<br>      gcp_security_reviewer    = optional(string, "")<br>      gcp_network_viewer       = optional(string, "")<br>      gcp_scc_admin            = optional(string, "")<br>      gcp_global_secrets_admin = optional(string, "")<br>      gcp_kms_admin            = optional(string, "")<br>    }), {})<br>  })</pre> | n/a | yes |
| initial\_group\_config | Define the group configuration when it is initialized. Valid values are: WITH\_INITIAL\_OWNER, EMPTY and INITIAL\_GROUP\_CONFIG\_UNSPECIFIED. | `string` | `"WITH_INITIAL_OWNER"` | no |
| org\_id | GCP Organization ID | `string` | n/a | yes |
| org\_policy\_admin\_role | Additional Org Policy Admin role for admin group. You can use this for testing purposes. | `bool` | `false` | no |
| parent\_folder | Optional - for an organization with existing projects or for development/validation. It will place all the example foundation resources under the provided folder instead of the root organization. The value is the numeric folder ID. The folder must already exist. | `string` | `""` | no |
| project\_deletion\_policy | The deletion policy for the project created. | `string` | `"PREVENT"` | no |
| project\_prefix | Name prefix to use for projects created. Should be the same in all steps. Max size is 3 characters. | `string` | `"prj"` | no |
| workflow\_deletion\_protection | Whether Terraform will be prevented from destroying a workflow. When the field is set to true or unset in Terraform state, a `terraform apply` or `terraform destroy` that would delete the workflow will fail. When the field is set to false, deleting the workflow is allowed. | `bool` | `true` | no |

## Outputs

| Name | Description |
|------|-------------|
| bootstrap\_step\_terraform\_service\_account\_email | Bootstrap Step Terraform Account |
| cloud\_build\_peered\_network\_id | The ID of the Cloud Build peered network. |
| cloud\_build\_private\_worker\_pool\_id | ID of the Cloud Build private worker pool. |
| cloud\_build\_worker\_peered\_ip\_range | The IP range of the peered service network. |
| cloud\_build\_worker\_range\_id | The Cloud Build private worker IP range ID. |
| cloud\_builder\_artifact\_repo | Artifact Registry (AR) Repository created to store TF Cloud Builder images. |
| cloudbuild\_project\_id | Project where Cloud Build configuration and terraform container image will reside. |
| common\_config | Common configuration data to be used in other steps. |
| csr\_repos | List of Cloud Source Repos created by the module, linked to Cloud Build triggers. |
| environment\_step\_terraform\_service\_account\_email | Environment Step Terraform Account |
| gcs\_bucket\_cloudbuild\_artifacts | Bucket used to store Cloud Build artifacts in cicd project. |
| gcs\_bucket\_cloudbuild\_logs | Bucket used to store Cloud Build logs in cicd project. |
| gcs\_bucket\_tfstate | Bucket used for storing terraform state for Foundations Pipelines in Seed Project. |
| networks\_step\_terraform\_service\_account\_email | Networks Step Terraform Account |
| optional\_groups | List of Google Groups created that are optional to the Example Foundation steps. |
| organization\_step\_terraform\_service\_account\_email | Organization Step Terraform Account |
| projects\_gcs\_bucket\_tfstate | Bucket used for storing terraform state for stage 4-projects foundations pipelines in seed project. |
| projects\_step\_terraform\_service\_account\_email | Projects Step Terraform Account |
| required\_groups | List of Google Groups created that are required by the Example Foundation steps. |
| seed\_project\_id | Project where service accounts and core APIs will be enabled. |

<!-- END OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
