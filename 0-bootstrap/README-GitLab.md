# GitLab CI/CD 호환 환경 배포

다음 지침의 목표는 Terraform Example Foundation 단계(`0-bootstrap`, `1-org`, `2-environments`, `3-networks`, `4-projects`)의 배포를 위해 GitLab CI/CD를 사용하여 CI/CD 배포를 허용하는 인프라를 구성하는 것입니다.
이 인프라는 두 개의 Google Cloud Platform 프로젝트(`prj-b-seed` 및 `prj-b-cicd-wif-gl`)로 구성됩니다.

관심사 분리를 위해 여기서 두 개의 별도 프로젝트(`prj-b-seed` 및 `prj-b-cicd-wif-gl`)를 갖는 것이 모범 사례입니다.
한편으로, `prj-b-seed`는 terraform 상태를 저장하고 인프라를 생성/수정할 수 있는 서비스 계정을 보유합니다.
반면에, [워크로드 ID 페더레이션](https://cloud.google.com/iam/docs/workload-identity-federation)을 사용한 인증 인프라는 `prj-b-cicd-wif-gl`에서 구현됩니다.

이 문서에 설명된 지침을 실행하려면 다음을 설치하십시오:

- [Google Cloud SDK](https://cloud.google.com/sdk/install) 버전 393.0.0 이상
    - [terraform-tools](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install) 구성 요소
- [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) 버전 2.28.0 이상
- [Terraform](https://www.terraform.io/downloads.html) 버전 1.5.7 이상
- [jq](https://jqlang.github.io/jq/) 버전 1.6 이상

이 문서에 설명된 수동 단계의 경우, 빌드 파이프라인에서 사용되는 것과 동일한 [Terraform](https://www.terraform.io/downloads.html) 버전을 사용해야 합니다.
그렇지 않으면 Terraform 상태 스냅샷 잠금 오류가 발생할 수 있습니다.

버전 1.5.7은 라이선스 모델 변경 이전의 마지막 버전입니다. 최신 버전의 Terraform을 사용하려면, `3-networks` 및 `4-projects` 단계의 일부를 수동으로 실행하는 데 사용되는 운영 체제의 Terraform 버전이 다음 코드에 구성된 것과 동일한 버전인지 확인하십시오

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

또한 다음 사항을 확인하십시오:

- 사용자 또는 그룹의 [GitLab](https://docs.gitlab.com/ee/user/profile/account/create_accounts.html) 계정
- Foundation의 각 단계와 GitLab 러너 이미지를 위한 비공개 GitLab 프로젝트(저장소):
    - Bootstrap
    - Organization
    - Environments
    - Networks
    - Projects
    - CI/CD Runner
- 다음 [범위](https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html#personal-access-token-scopes)로 구성된 [개인](https://docs.gitlab.com/ee/user/profile/personal_access_tokens.html) 액세스 토큰 또는 [그룹](https://docs.gitlab.com/ee/user/group/settings/group_access_tokens.html) 액세스 토큰:
    - api
    - read_api
    - create_runner
    - read_repository
    - write_repository
    - read_registry
    - write_registry
- Google Cloud [조직](https://cloud.google.com/resource-manager/docs/creating-managing-organization)
- Google Cloud [결제 계정](https://cloud.google.com/billing/docs/how-to/manage-billing-account)
- 조직 및 결제 관리자를 위한 Cloud Identity 또는 Google Workspace 그룹
- 이 문서의 절차를 실행할 사용자에게 다음 역할을 부여:
   - Google Cloud 조직의 `roles/resourcemanager.organizationAdmin` 역할
   - Google Cloud 조직의 `roles/orgpolicy.policyAdmin` 역할
   - Google Cloud 조직의 `roles/resourcemanager.projectCreator` 역할
   - 결제 계정의 `roles/billing.admin` 역할
   - `roles/resourcemanager.folderCreator` 역할

## 지침

### Terraform Example Foundation 저장소 복제

1. 로컬 환경에 [terraform-example-foundation](https://github.com/terraform-google-modules/terraform-example-foundation)을 복제합니다.

   ```bash
   git clone https://github.com/terraform-google-modules/terraform-example-foundation.git
   ```

### git 브랜치 생성

1. 이 문서에 설명된 지침은 Terraform Example Foundation 배포에 사용되는 브랜치가 존재하고 [보호](https://docs.gitlab.com/ee/user/project/protected_branches.html)되어 있어야 합니다. 모든 브랜치를 생성하려면 이 섹션의 지침을 따르십시오.
1. `terraform-example-foundation` 폴더와 같은 수준에서 생성한 모든 비공개 프로젝트를 복제합니다.
[인증](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/about-authentication-to-github)을 위해 GitLab에 [SSH 키](https://docs.gitlab.com/ee/user/ssh.html)가 구성되어 있어야 합니다.

   ```bash
   git clone git@gitlab.com:<GITLAB-OWNER>/<GITLAB-BOOTSTRAP-REPO>.git gcp-bootstrap
   ```
   ```bash
   git clone git@gitlab.com:<GITLAB-OWNER>/<GITLAB-ORGANIZATION-REPO>.git gcp-org
   ```
   ```bash
   git clone git@gitlab.com:<GITLAB-OWNER>/<GITLAB-ENVIRONMENTS-REPO>.git gcp-environments
   ```
   ```bash
   git clone git@gitlab.com:<GITLAB-OWNER>/<GITLAB-NETWORKS-REPO>.git gcp-networks
   ```
   ```bash
   git clone git@gitlab.com:<GITLAB-OWNER>/<GITLAB-PROJECTS-REPO>.git gcp-projects
   ```
   ```bash
   git clone git@gitlab.com:<GITLAB-OWNER>/<GITLAB-RUNNER-REPO>.git gcp-cicd-runner
   ```

1. 레이아웃은 다음과 같아야 합니다:

   ```bash
   gcp-bootstrap/
   gcp-cicd-runner/
   gcp-environments/
   gcp-networks/
   gcp-org/
   gcp-projects/
   terraform-example-foundation/
   ```

1. git 프로젝트에서 다음 브랜치들이 생성되어야 합니다.

   - CI/CD Runner: `image`
   - Bootstrap: `plan` 및 `production`
   - Organization: `plan` 및 `production`
   - Environments: `plan`, `development`, `nonproduction`, `production`
   - Networks: `plan`, `development`, `nonproduction`, `production`
   - Projects: `plan`, `development`, `nonproduction`, `production`

1. 이러한 브랜치들은 마지 요청의 대상 브랜치가 될 수 있도록 비어 있으면 안 됩니다.
`0-bootstrap/scripts/git_create_branches_helper.sh` 스크립트를 실행하여 각 저장소에 대해 시드 파일과 함께 이러한 브랜치들을 자동으로 생성합니다.

   ```bash
   ./terraform-example-foundation/0-bootstrap/scripts/git_create_branches_helper.sh GITLAB
   ```

1. 스크립트는 콘솔에 브랜치 생성과 관련된 로그를 출력하고 스크립트 실행 끝에 
`"Branch creation and push completed for all repositories"` 메시지를 출력합니다.
1. `0-bootstrap` 단계의 Terraform 구성은 이러한 브랜치들을 **보호된** 브랜치로 만듭니다

### CI/CD 러너 이미지 빌드

1. CI/CD 러너 저장소로 이동합니다. 이후의 모든 단계는 `gcp-cicd-runner` 디렉토리에서 실행한다고 가정합니다.
   다른 디렉토리에서 실행하는 경우 복사 경로를 적절히 조정하십시오.

   ```bash
   cd gcp-cicd-runner
   ```

1. `image` 브랜치로 체크아웃합니다. 이 브랜치에는 GitLab 파이프라인에서 사용되는 Docker 이미지가 포함됩니다.

   ```bash
   git checkout image
   ```

1. Copy contents of foundation to cloned project (modify accordingly based on your current directory).

   ```bash
   cp ../terraform-example-foundation/build/gitlab-ci.yml ./.gitlab-ci.yml
   cp ../terraform-example-foundation/0-bootstrap/Dockerfile ./Dockerfile
   ```

1. Save the CI/CD runner configuration to `gcp-cicd-runner` GitLab project:

   ```bash
   git add .
   git commit -m 'Initialize CI/CD runner project'
   git push
   ```

1. Review the CI/CD Job output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-RUNNER-REPO/-/jobs under name `build-image`.
1. If the CI/CD Job is successful proceed with the next steps.

### Provide access to the CI/CD runner image

1. Go to https://gitlab.com/GITLAB-OWNER/GITLAB-RUNNER-REPO/-/settings/ci_cd#js-token-access
1. Add all the repositories: Bootstrap, Organization, Environments, Networks, and Projects to the allow list tha allow access to the CI/CD runner image.
1. In "Allow CI job tokens from the following projects to access this project" add the other projects/repositories. Format is `<GITLAB-OWNER>/<GITLAB-REPO>`

### Deploying step 0-bootstrap

1. Navigate into the Bootstrap repo. All subsequent
   steps assume you are running them from the `gcp-bootstrap` directory.
   If you run them from another directory, adjust your copy paths accordingly.

   ```bash
   cd gcp-bootstrap
   ```

1. change to a nonproduction branch.

   ```bash
   git checkout plan
   ```

1. Copy contents of foundation to cloned project (modify accordingly based on your current directory).

   ```bash
   mkdir -p envs/shared

   cp -RT ../terraform-example-foundation/0-bootstrap/ ./envs/shared
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   cp ../terraform-example-foundation/build/gitlab-ci.yml ./.gitlab-ci.yml
   cp ../terraform-example-foundation/build/run_gcp_auth.sh .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./*.sh
   cd ./envs/shared
   ```

1. In the versions file `./versions.tf` un-comment the `gitlab` required provider
1. In the variables file `./variables.tf` un-comment variables in the section `Specific to gitlab_bootstrap`
1. In the outputs file `./outputs.tf` Comment-out outputs in the section `Specific to cloudbuild_module`
1. In the outputs file `./outputs.tf` un-comment outputs in the section `Specific to gitlab_bootstrap`
1. Rename file `./cb.tf` to `./cb.tf.example`

   ```bash
   mv ./cb.tf ./cb.tf.example
   ```

1. Rename file `./gitlab.tf.example` to `./gitlab.tf`

   ```bash
   mv ./gitlab.tf.example ./gitlab.tf
   ```

1. Rename file `terraform.example.tfvars` to `terraform.tfvars`

   ```bash
   mv ./terraform.example.tfvars ./terraform.tfvars
   ```

1. Update the file `terraform.tfvars` with values from your Google Cloud environment
1. Update the file `terraform.tfvars` with values from your GitLab projects
1. To prevent saving the `gitlab_token` in plain text in the `terraform.tfvars` file,
export the GitLab personal or group access token as an environment variable:

   ```bash
   export TF_VAR_gitlab_token="YOUR-PERSONAL-OR-GROUP-ACCESS-TOKEN"
   ```

1. Use the helper script [validate-requirements.sh](../scripts/validate-requirements.sh) to validate your environment:

   ```bash
   ../../../terraform-example-foundation/scripts/validate-requirements.sh  -o <ORGANIZATION_ID> -b <BILLING_ACCOUNT_ID> -u <END_USER_EMAIL> -e
   ```

   **Note:** The script is not able to validate if the user is in a Cloud Identity or Google Workspace group with the required roles.

1. Run `terraform init` and `terraform plan` and review the output.

   ```bash
   terraform init
   terraform plan -input=false -out bootstrap.tfplan
   ```

1. To  validate your policies, run `gcloud beta terraform vet`. For installation instructions, see [Validate policies](https://cloud.google.com/docs/terraform/policy-validation/validate-policies) instructions for the Google Cloud CLI.

1. Run the following commands and check for violations:

   ```bash
   export VET_PROJECT_ID=A-VALID-PROJECT-ID

   terraform show -json bootstrap.tfplan > bootstrap.json
   gcloud beta terraform vet bootstrap.json --policy-library="../../policy-library" --project ${VET_PROJECT_ID}
   ```

   *`A-VALID-PROJECT-ID`* must be an existing project you have access to. This is necessary because `gcloud beta terraform vet` needs to link resources to a valid Google Cloud Platform project.

1. No violations and an output with `done` means the validation was successful.

1. Run `terraform apply`.

   ```bash
   terraform apply bootstrap.tfplan
   ```

1. Run `terraform output` to get the email address of the terraform service accounts that will be used to run manual steps for `shared` environments in steps `3-networks-svpc`, `3-networks-hub-and-spoke`, and `4-projects`.

   ```bash
   export network_step_sa=$(terraform output -raw networks_step_terraform_service_account_email)
   export projects_step_sa=$(terraform output -raw projects_step_terraform_service_account_email)

   echo "network step service account = ${network_step_sa}"
   echo "projects step service account = ${projects_step_sa}"
   ```

1. Run `terraform output` to get the ID of your CI/CD project:

   ```bash
   export cicd_project_id=$(terraform output -raw cicd_project_id)
   echo "CI/CD Project ID = ${cicd_project_id}"
   ```

1. Copy the backend and update `backend.tf` with the name of your Google Cloud Storage bucket for Terraform's state. Also update the `backend.tf` of all steps.

   ```bash
   export backend_bucket=$(terraform output -raw gcs_bucket_tfstate)
   echo "backend_bucket = ${backend_bucket}"

   cp backend.tf.example backend.tf
   cd ../../../

   for i in `find . -name 'backend.tf'`; do sed -i'' -e "s/UPDATE_ME/${backend_bucket}/" $i; done
   for i in `find . -name 'backend.tf'`; do sed -i'' -e "s/UPDATE_PROJECTS_BACKEND/${backend_bucket}/" $i; done

   cd gcp-bootstrap/envs/shared
   ```

1. Re-run `terraform init`. When you're prompted, agree to copy Terraform state to Cloud Storage.

   ```bash
   terraform init
   ```

1. (Optional) Run `terraform plan` to verify that state is configured correctly. You should see no changes from the previous state.
1. Save the Terraform configuration to `gcp-bootstrap` GitLab project:

   ```bash
   cd ../..
   git add .
   git commit -m 'Initialize bootstrap project'
   git push
   ```

1. Open a merge request in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-BOOTSTRAP-REPO/-/merge_requests?scope=all&state=opened from the `plan` branch to the `production` branch and review the output.
1. The merge request will trigger a GitLab pipelines that will run Terraform `init`/`plan`/`validate` in the `production` environment.
1. Review the GitLab pipelines output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-BOOTSTRAP-REPO/-/pipelines.
1. If the GitLab pipelines is successful, merge the merge request in to the `production` branch.
1. The merge will trigger a GitLab pipelines that will apply the terraform configuration for the `production` environment.
1. Review merge output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-BOOTSTRAP-REPO/-/pipelines under `tf-apply`.
1. If the GitLab pipelines is successful, apply the next environment.

1. Before moving to the next step, go back to the parent directory.

   ```bash
   cd ..
   ```

**Note 1:** The stages after `0-bootstrap` use `terraform_remote_state` data source to read common configuration like the organization ID from the output of the `0-bootstrap` stage.
They will [fail](../docs/TROUBLESHOOTING.md#error-unsupported-attribute) if the state is not copied to the Cloud Storage bucket.

**Note 2:** After the deploy, to prevent the project quota error described in the [Troubleshooting guide](../docs/TROUBLESHOOTING.md#project-quota-exceeded),
we recommend that you request 50 additional projects for the **projects step service account** created in this step.

**Note 3:** In case GitLab variables are not being created on GitLab projects, it may be related to the existence of an Access Token in both User/Group profile and in the repo settings. The deploy will [fail](../docs/TROUBLESHOOTING.md#terraform-deploy-fails-due-to-gitlab-repositories-not-found).

**Note 4:** If the GitLab pipelines fail in 0-bootstrap or next steps with access denied in the runner image, you may need to add all projects to the [Token Access allow list in the CI/CD Runner repository](../docs/TROUBLESHOOTING.md#gitlab-pipelines-access-denied).

## Deploying step 1-org

1. Navigate into the repo. All subsequent steps assume you are running them from the `gcp-org` directory.
   If you run them from another directory, adjust your copy paths accordingly.

   ```bash
   cd gcp-org
   ```

1. change to a nonproduction branch.

   ```bash
   git checkout plan
   ```

1. Copy contents of foundation to new repo.

   ```bash
   cp -RT ../terraform-example-foundation/1-org/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   cp ../terraform-example-foundation/build/gitlab-ci.yml ./.gitlab-ci.yml
   cp ../terraform-example-foundation/build/run_gcp_auth.sh .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./*.sh
   ```

1. Rename `./envs/shared/terraform.example.tfvars` to `./envs/shared/terraform.tfvars`

   ```bash
   mv ./envs/shared/terraform.example.tfvars ./envs/shared/terraform.tfvars
   ```

1. Update the file `envs/shared/terraform.tfvars` with values from your GCP environment.
See the shared folder [README.md](../1-org/envs/shared/README.md#inputs) for additional information on the values in the `terraform.tfvars` file.

1. Un-comment the variable `create_access_context_manager_access_policy = false` if your organization already has an Access Context Manager Policy.

    ```bash
    export ORGANIZATION_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -json common_config | jq '.org_id' --raw-output)

    export ACCESS_CONTEXT_MANAGER_ID=$(gcloud access-context-manager policies list --organization ${ORGANIZATION_ID} --format="value(name)")

    echo "access_context_manager_policy_id = ${ACCESS_CONTEXT_MANAGER_ID}"

    if [ ! -z "${ACCESS_CONTEXT_MANAGER_ID}" ]; then sed -i'' -e "s=//create_access_context_manager_access_policy=create_access_context_manager_access_policy=" ./envs/shared/terraform.tfvars; fi
    ```

1. Update the `remote_state_bucket` variable with the backend bucket from step Bootstrap.

   ```bash
   export backend_bucket=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw gcs_bucket_tfstate)

   echo "remote_state_bucket = ${backend_bucket}"

   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" ./envs/shared/terraform.tfvars
   ```

1. Check if a Security Command Center Notification with the default name, **scc-notify**, already exists in your organization.

   ```bash
   export ORG_STEP_SA=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw organization_step_terraform_service_account_email)

   gcloud scc notifications describe "scc-notify" --format="value(name)" --organization=${ORGANIZATION_ID} --impersonate-service-account=${ORG_STEP_SA}
   ```

1. If the notification exists the output will be:

    ```text
    organizations/ORGANIZATION_ID/notificationConfigs/scc-notify
    ```

1. If the notification does not exist the output will be:

    ```text
    ERROR: (gcloud.scc.notifications.describe) NOT_FOUND: Requested entity was not found.
    ```

1. If the notification exists, choose a different value for the `scc_notification_name` variable in the `./envs/shared/terraform.tfvars` file.

1. Commit changes and Push your plan branch.

   ```bash
   git add .
   git commit -m 'Initialize org repo'
   git push
   ```

1. Open a merge request in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-ORGANIZATION-REPO/-/merge_requests?scope=all&state=opened from the `plan` branch to the `production` branch and review the output.
1. The merge request will trigger a GitLab pipelines that will run Terraform `init`/`plan`/`validate` in the `production` environment.
1. Review the GitLab pipelines output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-ORGANIZATION-REPO/-/pipelines.
1. If the GitLab pipelines is successful, merge the merge request in to the `production` branch.
1. The merge will trigger a GitLab pipelines that will apply the terraform configuration for the `production` environment.
1. Review merge output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-ORGANIZATION-REPO/-/pipelines under `tf-apply`.
1. If the GitLab pipelines is successful, apply the next environment.

1. Before moving to the next step, go back to the parent directory.

   ```bash
   cd ..
   ```

## Deploying step 2-environments

1. Navigate into the repo. All subsequent
   steps assume you are running them from the `gcp-environments` directory.
   If you run them from another directory, adjust your copy paths accordingly.

   ```bash
   cd gcp-environments
   ```

1. change to a nonproduction branch.

   ```bash
   git checkout plan
   ```

1. Copy contents of foundation to new repo.

   ```bash
   cp -RT ../terraform-example-foundation/2-environments/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   cp ../terraform-example-foundation/build/gitlab-ci.yml ./.gitlab-ci.yml
   cp ../terraform-example-foundation/build/run_gcp_auth.sh .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./*.sh
   ```

1. Rename `terraform.example.tfvars` to `terraform.tfvars`.

   ```bash
   mv terraform.example.tfvars terraform.tfvars
   ```

1. Update the file with values from your GCP environment.
See any of the envs folder [README.md](../2-environments/envs/production/README.md#inputs) files for additional information on the values in the `terraform.tfvars` file.

1. Update the `remote_state_bucket` variable with the backend bucket from step Bootstrap.

   ```bash
   export backend_bucket=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${backend_bucket}"

   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" terraform.tfvars
   ```

1. Commit changes and push your plan branch.

   ```bash
   git add .
   git commit -m 'Initialize environments repo'
   git push
   ```

1. Open a merge request in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-ENVIRONMENTS-REPO/-/merge_requests?scope=all&state=opened from the `plan` branch to the `development` branch and review the output.
1. The merge request will trigger a GitLab pipelines that will run Terraform `init`/`plan`/`validate` in the `development` environment.
1. Review the GitLab pipelines output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-ENVIRONMENTS-REPO/-/pipelines.
1. If the GitLab pipelines is successful, merge the merge request in to the `development` branch.
1. The merge will trigger a GitLab pipelines that will apply the terraform configuration for the `development` environment.
1. Review merge output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-ENVIRONMENTS-REPO/-/pipelines under `tf-apply`.
1. If the GitLab pipelines is successful, apply the next environment.

1. Open a merge request in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-ENVIRONMENTS-REPO/-/merge_requests?scope=all&state=opened from the `development` branch to the `nonproduction` branch and review the output.
1. The merge request will trigger a GitLab pipelines that will run Terraform `init`/`plan`/`validate` in the `nonproduction` environment.
1. Review the GitLab pipelines output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-ENVIRONMENTS-REPO/-/pipelines.
1. If the GitLab pipelines is successful, merge the merge request in to the `nonproduction` branch.
1. The merge will trigger a GitLab pipelines that will apply the terraform configuration for the `nonproduction` environment.
1. Review merge output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-ENVIRONMENTS-REPO/-/pipelines under `tf-apply`.
1. If the GitLab pipelines is successful, apply the next environment.

1. Open a merge request in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-ENVIRONMENTS-REPO/-/merge_requests?scope=all&state=opened from the `nonproduction` branch to the `production` branch and review the output.
1. The merge request will trigger a GitLab pipelines that will run Terraform `init`/`plan`/`validate` in the `production` environment.
1. Review the GitLab pipelines output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-ENVIRONMENTS-REPO/-/pipelines.
1. If the GitLab pipelines is successful, merge the merge request in to the `production` branch.
1. The merge will trigger a GitLab pipelines that will apply the terraform configuration for the `production` environment.
1. Review merge output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-ENVIRONMENTS-REPO/-/pipelines under `tf-apply`.

1. Before moving to the next step, go back to the parent directory.

   ```bash
   cd ..
   ```

1. You can now move to the instructions in the network stage.
To use the [Dual Shared VPC](https://cloud.google.com/architecture/security-foundations/networking#vpcsharedvpc-id7-1-shared-vpc-) network mode go to [Deploying step 3-networks-svpc](#deploying-step-3-networks-svpc),
or go to [Deploying step 3-networks-hub-and-spoke](#deploying-step-3-networks-hub-and-spoke) to use the [Hub and Spoke](https://cloud.google.com/architecture/security-foundations/networking#hub-and-spoke) network mode.

## Deploying step 3-networks-svpc

1. Navigate into the repo. All subsequent steps assume you are running them from the `gcp-networks` directory.
   If you run them from another directory, adjust your copy paths accordingly.

   ```bash
   cd gcp-networks
   ```

1. change to a nonproduction branch.

   ```bash
   git checkout plan
   ```

1. Copy contents of foundation to new repo.

   ```bash
   cp -RT ../terraform-example-foundation/3-networks-svpc/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   cp ../terraform-example-foundation/build/gitlab-ci.yml ./.gitlab-ci.yml
   cp ../terraform-example-foundation/build/run_gcp_auth.sh .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./*.sh
   ```

1. Rename `common.auto.example.tfvars` to `common.auto.tfvars`, rename `production.auto.example.tfvars` to `production.auto.tfvars` and rename `access_context.auto.example.tfvars` to `access_context.auto.tfvars`.

   ```bash
   mv common.auto.example.tfvars common.auto.tfvars
   mv production.auto.example.tfvars production.auto.tfvars
   mv access_context.auto.example.tfvars access_context.auto.tfvars
   ```

1. Update the file `production.auto.tfvars` with the values for the `target_name_server_addresses`.
1. Update the file `access_context.auto.tfvars` with the organization's `access_context_manager_policy_id`.

   ```bash
   export ORGANIZATION_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -json common_config | jq '.org_id' --raw-output)

   export ACCESS_CONTEXT_MANAGER_ID=$(gcloud access-context-manager policies list --organization ${ORGANIZATION_ID} --format="value(name)")

   echo "access_context_manager_policy_id = ${ACCESS_CONTEXT_MANAGER_ID}"

   sed -i'' -e "s/ACCESS_CONTEXT_MANAGER_ID/${ACCESS_CONTEXT_MANAGER_ID}/" ./access_context.auto.tfvars
   ```

1. Update `common.auto.tfvars` file with values from your GCP environment.
See any of the envs folder [README.md](../3-networks-svpc/envs/production/README.md#inputs) files for additional information on the values in the `common.auto.tfvars` file.
1. You must add your user email in the variable `perimeter_additional_members` to be able to see the resources created in the project.
1. Update the `remote_state_bucket` variable with the backend bucket from step Bootstrap in the `common.auto.tfvars` file.

   ```bash
   export backend_bucket=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -raw gcs_bucket_tfstate)

   echo "remote_state_bucket = ${backend_bucket}"

   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" ./common.auto.tfvars
   ```

1. Commit changes

   ```bash
   git add .
   git commit -m 'Initialize networks repo'
   ```

1. You must manually plan and apply the `shared` environment (only once) since the `development`, `nonproduction` and `production` environments depend on it.
1. Use `terraform output` to get the CI/CD project ID and the networks step Terraform Service Account from gcp-bootstrap output.
1. The CI/CD project ID will be used in the [validation](https://cloud.google.com/docs/terraform/policy-validation/quickstart) of the Terraform configuration

   ```bash
   export CICD_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -raw cicd_project_id)
   echo ${CICD_PROJECT_ID}
   ```

1. The networks step Terraform Service Account will be used for [Service Account impersonation](https://cloud.google.com/docs/authentication/use-service-account-impersonation) in the following steps.
An environment variable `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` will be set with the Terraform Service Account to enable impersonation.

   ```bash
   export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -raw networks_step_terraform_service_account_email)
   echo ${GOOGLE_IMPERSONATE_SERVICE_ACCOUNT}
   ```

1. Run `init` and `plan` and review output for environment shared.

   ```bash
   ./tf-wrapper.sh init shared
   ./tf-wrapper.sh plan shared
   ```

1. To use the `validate` option of the `tf-wrapper.sh` script, please follow the [instructions](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install) to install the terraform-tools component.
1. Run `validate` and check for violations.

   ```bash
   ./tf-wrapper.sh validate shared $(pwd)/policy-library ${CICD_PROJECT_ID}
   ```

1. Run `apply` shared.

   ```bash
   ./tf-wrapper.sh apply shared
   ```

1. You must manually plan and apply the `production` environment since the `development`, `nonproduction` and `plan` environments depend on it.

   ```bash
   git checkout production
   git merge plan
   ```

1. Run `init` and `plan` and review output for environment production.

   ```bash
   ./tf-wrapper.sh init production
   ./tf-wrapper.sh plan production
   ```

1. Run `apply` production.

   ```bash
   ./tf-wrapper.sh apply production
   ```

   1. Push your production branch since development and nonproduction depends it.

*Note:** The Production envrionment must be the first branch to be pushed as it includes the DNS Hub communication that will be used by other environments.

   ```bash
   git add .
   git commit -m 'Initialize networks repo - production'
   git push
   ```

1. Open a merge request in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-NETWORKS-REPO/-/merge_requests?scope=all&state=opened from the `production` branch to the `plan` branch and review the output.

1. Push your plan branch.

   ```bash
   git checkout plan
   git push
   ```

1. Open a merge request in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-NETWORKS-REPO/-/merge_requests?scope=all&state=opened from the `production` branch to the `development` branch and review the output.

   > NOTE: Development and Non-production branches depends on the production branch to be deployed first, for the `3-networks-dual-svpc`.

1. The merge request will trigger a GitLab pipelines that will run Terraform `init`/`plan`/`validate` in the `development` environment.
1. Review the GitLab pipelines output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-NETWORKS-REPO/-/pipelines.
1. If the GitLab pipelines is successful, merge the merge request in to the `development` branch.
1. The merge will trigger a GitLab pipelines that will apply the terraform configuration for the `development` environment.
1. Review merge output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-NETWORKS-REPO/-/pipelines under `tf-apply`.

1. Open a merge request in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-NETWORKS-REPO/-/merge_requests?scope=all&state=opened from the `production` branch to the `nonproduction` branch and review the output.
1. The merge request will trigger a GitLab pipelines that will run Terraform `init`/`plan`/`validate` in the `nonproduction` environment.
1. Review the GitLab pipelines output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-NETWORKS-REPO/-/pipelines.
1. If the GitLab pipelines is successful, merge the merge request in to the `nonproduction` branch.
1. The merge will trigger a GitLab pipelines that will apply the terraform configuration for the `nonproduction` environment.
1. Review merge output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-NETWORKS-REPO/-/pipelines under `tf-apply`.

1. Before executing the next steps, unset the `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` environment variable.

   ```bash
   unset GOOGLE_IMPERSONATE_SERVICE_ACCOUNT
   ```

1. Before moving to the next step, go back to the parent directory.

   ```bash
   cd ..
   ```

1. You can now move to the instructions in the [4-projects](#deploying-step-4-projects) stage.

## Deploying step 3-networks-hub-and-spoke

1. Navigate into the repo. All subsequent steps assume you are running them from the `gcp-networks` directory.
   If you run them from another directory, adjust your copy paths accordingly.

   ```bash
   cd gcp-networks
   ```

1. change to a nonproduction branch.

   ```bash
   git checkout plan
   ```

1. Copy contents of foundation to new repo.

   ```bash
   cp -RT ../terraform-example-foundation/3-networks-hub-and-spoke/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   cp ../terraform-example-foundation/build/gitlab-ci.yml ./.gitlab-ci.yml
   cp ../terraform-example-foundation/build/run_gcp_auth.sh .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./*.sh
   ```

1. Rename `common.auto.example.tfvars` to `common.auto.tfvars`, rename `shared.auto.example.tfvars` to `shared.auto.tfvars` and rename `access_context.auto.example.tfvars` to `access_context.auto.tfvars`.

   ```bash
   mv common.auto.example.tfvars common.auto.tfvars
   mv shared.auto.example.tfvars shared.auto.tfvars
   mv access_context.auto.example.tfvars access_context.auto.tfvars
   ```

1. Update `common.auto.tfvars` file with values from your GCP environment.
See any of the envs folder [README.md](../3-networks-hub-and-spoke/envs/production/README.md#inputs) files for additional information on the values in the `common.auto.tfvars` file.
1. You must add your user email in the variable `perimeter_additional_members` to be able to see the resources created in the project.
1. Update the `remote_state_bucket` variable with the backend bucket from step Bootstrap in the `common.auto.tfvars` file.

   ```bash
   export backend_bucket=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -raw gcs_bucket_tfstate)

   echo "remote_state_bucket = ${backend_bucket}"

   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" ./common.auto.tfvars
   ```

1. Commit changes

   ```bash
   git add .
   git commit -m 'Initialize networks repo'
   ```

1. You must manually plan and apply the `shared` environment (only once) since the `development`, `nonproduction` and `production` environments depend on it.
1. Use `terraform output` to get the CI/CD project ID and the networks step Terraform Service Account from gcp-bootstrap output.
1. The CI/CD project ID will be used in the [validation](https://cloud.google.com/docs/terraform/policy-validation/quickstart) of the Terraform configuration

   ```bash
   export CICD_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -raw cicd_project_id)
   echo ${CICD_PROJECT_ID}
   ```

1. The networks step Terraform Service Account will be used for [Service Account impersonation](https://cloud.google.com/docs/authentication/use-service-account-impersonation) in the following steps.
An environment variable `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` will be set with the Terraform Service Account to enable impersonation.

   ```bash
   export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -raw networks_step_terraform_service_account_email)
   echo ${GOOGLE_IMPERSONATE_SERVICE_ACCOUNT}
   ```

1. Run `init` and `plan` and review output for environment shared.

   ```bash
   ./tf-wrapper.sh init shared
   ./tf-wrapper.sh plan shared
   ```

1. To use the `validate` option of the `tf-wrapper.sh` script, please follow the [instructions](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install) to install the terraform-tools component.
1. Run `validate` and check for violations.

   ```bash
   ./tf-wrapper.sh validate shared $(pwd)/policy-library ${CICD_PROJECT_ID}
   ```

1. Run `apply` shared.

   ```bash
   ./tf-wrapper.sh apply shared
   ```

1. Push your plan branch.

   ```bash
   git push
   ```

1. Open a merge request in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-NETWORKS-REPO/-/merge_requests?scope=all&state=opened from the `plan` branch to the `development` branch and review the output.
1. The merge request will trigger a GitLab pipelines that will run Terraform `init`/`plan`/`validate` in the `development` environment.
1. Review the GitLab pipelines output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-NETWORKS-REPO/-/pipelines.
1. If the GitLab pipelines is successful, merge the merge request in to the `development` branch.
1. The merge will trigger a GitLab pipelines that will apply the terraform configuration for the `development` environment.
1. Review merge output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-NETWORKS-REPO/-/pipelines under `tf-apply`.
1. If the GitLab pipelines is successful, apply the next environment.

1. Open a merge request in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-NETWORKS-REPO/-/merge_requests?scope=all&state=opened from the `development` branch to the `nonproduction` branch and review the output.
1. The merge request will trigger a GitLab pipelines that will run Terraform `init`/`plan`/`validate` in the `nonproduction` environment.
1. Review the GitLab pipelines output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-NETWORKS-REPO/-/pipelines.
1. If the GitLab pipelines is successful, merge the merge request in to the `nonproduction` branch.
1. The merge will trigger a GitLab pipelines that will apply the terraform configuration for the `nonproduction` environment.
1. Review merge output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-NETWORKS-REPO/-/pipelines under `tf-apply`.
1. If the GitLab pipelines is successful, apply the next environment.

1. Open a merge request in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-NETWORKS-REPO/-/merge_requests?scope=all&state=opened from the `nonproduction` branch to the `production` branch and review the output.
1. The merge request will trigger a GitLab pipelines that will run Terraform `init`/`plan`/`validate` in the `production` environment.
1. Review the GitLab pipelines output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-NETWORKS-REPO/-/pipelines.
1. If the GitLab pipelines is successful, merge the merge request in to the `production` branch.
1. The merge will trigger a GitLab pipelines that will apply the terraform configuration for the `production` environment.
1. Review merge output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-NETWORKS-REPO/-/pipelines under `tf-apply`.


1. Before executing the next steps, unset the `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` environment variable.

   ```bash
   unset GOOGLE_IMPERSONATE_SERVICE_ACCOUNT
   ```

1. Before moving to the next step, go back to the parent directory.

   ```bash
   cd ..
   ```

1. You can now move to the instructions in the [4-projects](#deploying-step-4-projects) stage.

## Deploying step 4-projects

1. Navigate into the repo. All subsequent
   steps assume you are running them from the `gcp-projects` directory.
   If you run them from another directory, adjust your copy paths accordingly.

   ```bash
   cd gcp-projects
   ```

1. change to a nonproduction branch.

   ```bash
   git checkout plan
   ```

1. Copy contents of foundation to new repo.

   ```bash
   cp -RT ../terraform-example-foundation/4-projects/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   cp ../terraform-example-foundation/build/gitlab-ci.yml ./.gitlab-ci.yml
   cp ../terraform-example-foundation/build/run_gcp_auth.sh .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./*.sh
   ```

1. Rename `auto.example.tfvars` files to `auto.tfvars`.

   ```bash
   mv common.auto.example.tfvars common.auto.tfvars
   mv shared.auto.example.tfvars shared.auto.tfvars
   mv development.auto.example.tfvars development.auto.tfvars
   mv nonproduction.auto.example.tfvars nonproduction.auto.tfvars
   mv production.auto.example.tfvars production.auto.tfvars
   ```

1. See any of the envs folder [README.md](../4-projects/business_unit_1/production/README.md#inputs) files for additional information on the values in the `common.auto.tfvars`, `development.auto.tfvars`, `nonproduction.auto.tfvars`, and `production.auto.tfvars` files.
1. See any of the shared folder [README.md](../4-projects/business_unit_1/shared/README.md#inputs) files for additional information on the values in the `shared.auto.tfvars` file.

1. Use `terraform output` to get the backend bucket value from bootstrap output.

   ```bash
   export remote_state_bucket=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${remote_state_bucket}"

   sed -i'' -e "s/REMOTE_STATE_BUCKET/${remote_state_bucket}/" ./common.auto.tfvars
   ```

1. (Optional) If you want additional subfolders for separate business units or entities, make additional copies of the folder `business_unit_1` and modify any values that vary across business unit like `business_code`, `business_unit`, or `subnet_ip_range`.

For example, to create a new business unit similar to business_unit_1, run the following:

   ```bash
   #copy the business_unit_1 folder and it's contents to a new folder business_unit_2
   cp -r  business_unit_1 business_unit_2

   # search all files under the folder `business_unit_2` and replace strings for business_unit_1 with strings for business_unit_2
   grep -rl bu1 business_unit_2/ | xargs sed -i 's/bu1/bu2/g'
   grep -rl business_unit_1 business_unit_2/ | xargs sed -i 's/business_unit_1/business_unit_2/g'
   ```


1. Commit changes.

   ```bash
   git add .
   git commit -m 'Initialize projects repo'
   ```

1. You need to manually plan and apply only once the `business_unit_1/shared` and `business_unit_2/shared` environments since `development`, `nonproduction`, and `production` depend on them.

1. Use `terraform output` to get the CI/CD project ID and the projects step Terraform Service Account from gcp-bootstrap output.
1. The CI/CD project ID will be used in the [validation](https://cloud.google.com/docs/terraform/policy-validation/quickstart) of the Terraform configuration

   ```bash
   export CICD_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -raw cicd_project_id)
   echo ${CICD_PROJECT_ID}
   ```

1. The projects step Terraform Service Account will be used for [Service Account impersonation](https://cloud.google.com/docs/authentication/use-service-account-impersonation) in the following steps.
An environment variable `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` will be set with the Terraform Service Account to enable impersonation.

   ```bash
   export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -raw projects_step_terraform_service_account_email)
   echo ${GOOGLE_IMPERSONATE_SERVICE_ACCOUNT}
   ```

1. Run `init` and `plan` and review output for environment shared.

   ```bash
   ./tf-wrapper.sh init shared
   ./tf-wrapper.sh plan shared
   ```

1. To use the `validate` option of the `tf-wrapper.sh` script, please follow the [instructions](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install) to install the terraform-tools component.
1. Run `validate` and check for violations.

   ```bash
   ./tf-wrapper.sh validate shared $(pwd)/policy-library ${CICD_PROJECT_ID}
   ```

1. Run `apply` shared.

   ```bash
   ./tf-wrapper.sh apply shared
   ```

1. Push your plan branch.

   ```bash
   git push
   ```

1. Open a merge request in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-PROJECTS-REPO/-/merge_requests?scope=all&state=opened from the `plan` branch to the `development` branch and review the output.
1. The merge request will trigger a GitLab pipelines that will run Terraform `init`/`plan`/`validate` in the `development` environment.
1. Review the GitLab pipelines output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-PROJECTS-REPO/-/pipelines.
1. If the GitLab pipelines is successful, merge the merge request in to the `development` branch.
1. The merge will trigger a GitLab pipelines that will apply the terraform configuration for the `development` environment.
1. Review merge output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-PROJECTS-REPO/-/pipelines under `tf-apply`.
1. If the GitLab pipelines is successful, apply the next environment.

1. Open a merge request in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-PROJECTS-REPO/-/merge_requests?scope=all&state=opened from the `development` branch to the `nonproduction` branch and review the output.
1. The merge request will trigger a GitLab pipelines that will run Terraform `init`/`plan`/`validate` in the `nonproduction` environment.
1. Review the GitLab pipelines output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-PROJECTS-REPO/-/pipelines.
1. If the GitLab pipelines is successful, merge the merge request in to the `nonproduction` branch.
1. The merge will trigger a GitLab pipelines that will apply the terraform configuration for the `nonproduction` environment.
1. Review merge output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-PROJECTS-REPO/-/pipelines under `tf-apply`.
1. If the GitLab pipelines is successful, apply the next environment.

1. Open a merge request in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-PROJECTS-REPO/-/merge_requests?scope=all&state=opened from the `nonproduction` branch to the `production` branch and review the output.
1. The merge request will trigger a GitLab pipelines that will run Terraform `init`/`plan`/`validate` in the `production` environment.
1. Review the GitLab pipelines output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-PROJECTS-REPO/-/pipelines.
1. If the GitLab pipelines is successful, merge the merge request in to the `production` branch.
1. The merge will trigger a GitLab pipelines that will apply the terraform configuration for the `production` environment.
1. Review merge output in GitLab https://gitlab.com/GITLAB-OWNER/GITLAB-PROJECTS-REPO/-/pipelines under `tf-apply`.

1. Unset the `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` environment variable.

   ```bash
   unset GOOGLE_IMPERSONATE_SERVICE_ACCOUNT
   ```
