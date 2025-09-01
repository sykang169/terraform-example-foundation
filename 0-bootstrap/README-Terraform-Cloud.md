# Terraform Cloud (TFC) 호환 환경 배포

아래 지침의 목표는 Terraform Example Foundation 단계(`0-bootstrap`, `1-org`, `2-environments`, `3-networks`, `4-projects`)에 대해 
Terraform Cloud를 사용하여 CI/CD 배포를 실행할 수 있게 해주는 인프라를 구성하는 것입니다.
인프라는 두 개의 Google Cloud Platform 프로젝트(`prj-b-seed`와 `prj-b-cicd-wif-tfc`)로 구성됩니다.

관심사의 분리를 위해 여기에 두 개의 별도 프로젝트(`prj-b-seed`와 `prj-b-cicd-wif-tfc`)를 갖는 것이 모범 사례입니다.
한편으로는 `prj-b-seed`가 인프라를 생성/수정할 수 있는 서비스 계정을 가지고 있습니다.
다른 한편으로는 [Workload identity federation](https://cloud.google.com/iam/docs/workload-identity-federation)을 사용한 인증 인프라가 `prj-b-cicd-wif-tfc`에서 구현됩니다. 다른 배포 방법과 달리 Terraform 상태는 GCP의 버킷이 아닌 Terraform Cloud에 저장됩니다.

참고: [Terraform Cloud with Agents](https://developer.hashicorp.com/terraform/cloud-docs/agents)를 사용하기로 선택한 경우, 개인 오토파일럿 GKE 클러스터가 Agent로 사용되기 위해 `prj-b-cicd-wif-tfc` GCP 프로젝트에 배포됩니다.

## 요구사항

이 문서에 설명된 지침을 실행하려면 다음을 설치하세요:

- [Google Cloud SDK](https://cloud.google.com/sdk/install) 버전 393.0.0 이상
    - [terraform-tools](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install) 컴포넌트
- [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) 버전 2.28.0 이상
- [Terraform](https://www.terraform.io/downloads.html) 버전 1.5.7 이상
- [jq](https://jqlang.github.io/jq/download/) 버전 1.6.0 이상

이 문서에 설명된 수동 단계의 경우, 빌드 파이프라인에서 사용되는 것과 동일한 [Terraform](https://www.terraform.io/downloads.html) 버전을 사용해야 합니다.
그렇지 않으면 Terraform 상태 스냅샷 잠금 오류가 발생할 수 있습니다.

또한 다음 사항을 확인하세요:

- 사용자 또는 [조직](https://developer.hashicorp.com/terraform/tutorials/cloud-get-started/cloud-sign-up#create-an-organization)을 위한 [Terraform Cloud 계정](https://developer.hashicorp.com/terraform/tutorials/cloud-get-started/cloud-sign-up#create-an-account).
- Terraform Cloud [조직](https://developer.hashicorp.com/terraform/cloud-docs/users-teams-organizations/organizations#creating-organizations).
- Terraform Cloud [사용자 토큰](https://developer.hashicorp.com/terraform/cloud-docs/users-teams-organizations/api-tokens#user-api-tokens) 또는 [조직 토큰](https://developer.hashicorp.com/terraform/cloud-docs/users-teams-organizations/api-tokens#organization-api-tokens).
   - 권한이 단일 TFC 조직으로 제한되므로 조직 토큰이 선호됩니다.
- Terraform Cloud 계정과 [연결된](https://developer.hashicorp.com/terraform/cloud-docs/vcs) [지원되는](https://developer.hashicorp.com/terraform/cloud-docs/vcs#supported-vcs-providers) 버전 제어 시스템(VCS) 제공자.
   - 다음 목록은 이 README에서 완전히 지원하는 TFC를 VCS 제공자에 연결하는 지원 방법을 나타냅니다. 여기에 나열되지 않은 제공자와 연결하는 것도 가능하지만 일부 조정이 필요할 수 있습니다.
      - [GitHub (OAuth)](https://developer.hashicorp.com/terraform/cloud-docs/vcs/github)
      - [GitHub Enterprise](https://developer.hashicorp.com/terraform/cloud-docs/vcs/github-enterprise)
      - [GitLab.com](https://developer.hashicorp.com/terraform/cloud-docs/vcs/gitlab-com)
      - [GitLab EE/CE](https://developer.hashicorp.com/terraform/cloud-docs/vcs/gitlab-eece)
   - VCS 제공자를 구성한 후에는 TFC 콘솔(https://app.terraform.io/app/YOUR-TFC-ORGANIZATION/settings/version-control)에서 사용 가능한 `OAuth Token ID`를 복사할 수 있어야 합니다.
- Foundation의 각 단계마다 VCS 제공자의 **개인** 리포지토리(또는 프로젝트):
   - Bootstrap
   - Organization
   - Environments
   - Networks
   - Projects
   - 자세한 내용은 [GitHub 리포지토리 생성](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-new-repository) 또는 [GitLab 프로젝트 생성](https://docs.gitlab.com/ee/user/project/index.html#create-a-blank-project)을 참조하세요.
- Google Cloud [조직](https://cloud.google.com/resource-manager/docs/creating-managing-organization).
- Google Cloud [결제 계정](https://cloud.google.com/billing/docs/how-to/manage-billing-account).
- 조직 및 결제 관리자를 위한 Cloud Identity 또는 Google Workspace 그룹.
- 이 문서의 절차를 실행할 사용자에게 다음 역할을 부여하세요:
   - Google Cloud 조직에 대한 `roles/resourcemanager.organizationAdmin` 역할.
   - Google Cloud 조직에 대한 `roles/orgpolicy.policyAdmin` 역할.
   - Google Cloud 조직에 대한 `roles/resourcemanager.projectCreator` 역할.
   - 결제 계정에 대한 `roles/billing.admin` 역할.
   - `roles/resourcemanager.folderCreator` 역할.

### 지침

1. [terraform-example-foundation](https://github.com/terraform-google-modules/terraform-example-foundation)을 로컬 환경에 복제합니다.

   ```bash
   git clone https://github.com/terraform-google-modules/terraform-example-foundation.git
   ```

1. `terraform-example-foundation` 폴더와 동일한 수준에서 생성한 모든 개인 리포지토리(또는 프로젝트)를 복제합니다.
VCS 제공자에 인증되어 있어야 합니다. 자세한 내용은 [GitHub 인증](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/about-authentication-to-github) 또는 [GitLab 인증](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/about-authentication-to-github)을 참조하세요.

   ```bash
   git clone git@<VCS-서비스-제공자>.com:<VCS-소유자>/<VCS-BOOTSTRAP-리포>.git gcp-bootstrap
   ```
   ```bash
   git clone git@<VCS-서비스-제공자>.com:<VCS-소유자>/<VCS-ORGANIZATION-리포>.git gcp-org
   ```
   ```bash
   git clone git@<VCS-서비스-제공자>.com:<VCS-소유자>/<VCS-ENVIRONMENTS-리포>.git gcp-environments
   ```
   ```bash
   git clone git@<VCS-서비스-제공자>.com:<VCS-소유자>/<VCS-NETWORKS-리포>.git gcp-networks
   ```
   ```bash
   git clone git@<VCS-서비스-제공자>.com:<VCS-소유자>/<VCS-PROJECTS-리포>.git gcp-projects
   ```

1. 레이아웃은 다음과 같아야 합니다:

   ```bash
   gcp-bootstrap/
   gcp-org/
   gcp-environments/
   gcp-networks/
   gcp-projects/
   terraform-example-foundation/
   ```

1. VCS 리포지토리(또는 프로젝트)에서 다음 브랜치가 생성되어야 합니다. 또한 이 브랜치들은 비어있으면 안 되며, 최소 하나의 파일이 필요합니다. `scripts/git_create_branches_helper.sh` 스크립트를 실행하여 각 리포지토리에 대해 시드 파일과 함께 이 브랜치들을 자동으로 생성하세요.
   - Bootstrap: `production`
   - Organization: `production`
   - Environments: `development`, `nonproduction`, `production`
   - Networks: `development`, `nonproduction`, `production`
   - Projects: `development`, `nonproduction`, `production`
   - 참고: `scripts/git_create_branches_helper.sh` 스크립트와 다음 명령어들은 모든 리포지토리가 복제된 디렉토리(이전 단계에서 설명한 레이아웃)에서 실행하는 것을 가정합니다. 다른 디렉토리에서 실행하는 경우 `scripts/git_create_branches_helper.sh`의 `BASE_PATH` 변수와 다음 명령어들을 조정하세요.

   ```bash
   chmod 755 ./terraform-example-foundation/0-bootstrap/scripts/git_create_branches_helper.sh
   ./terraform-example-foundation/0-bootstrap/scripts/git_create_branches_helper.sh
   ```

   콘솔에서 브랜치 생성과 관련된 일부 GIT 로그를 보게 되고, 스크립트 실행 끝에 `"Branch creation and push completed for all repositories"` 메시지를 보게 됩니다.

1. `login` 명령어를 실행하고 자동으로 열려야 하는 브라우저 탭에서 제공되는 지침을 따라 [Terraform CLI를 인증](https://developer.hashicorp.com/terraform/cli/commands/login)하세요.
   ```bash
   terraform login
   ```
**참고**: 사용자 토큰을 생성하기 위해 이미 [조직 토큰](https://developer.hashicorp.com/terraform/cloud-docs/users-teams-organizations/api-tokens#organization-api-tokens)이 있더라도 이 단계를 수행해야 합니다.

### 0-bootstrap 단계 배포

1. 리포지토리로 이동합니다. 모든 후속 단계는 `gcp-bootstrap` 디렉토리에서 실행한다고 가정합니다.
   다른 디렉토리에서 실행하는 경우 복사 경로를 적절히 조정하세요.

   ```bash
   cd gcp-bootstrap
   ```

1. 비프로덕션 브랜치로 변경합니다.

   ```bash
   git checkout -b plan
   ```

1. 파운데이션의 내용을 새 리포지토리로 복사합니다(현재 디렉토리에 따라 적절히 수정).

   ```bash
   mkdir -p envs/shared
   cp -RT ../terraform-example-foundation/0-bootstrap/ ./envs/shared
   cd ./envs/shared
   ```

1. versions 파일 `./versions.tf`에서 `tfe` 필수 공급자의 주석을 해제합니다
1. variables 파일 `./variables.tf`에서 `Specific to tfc_bootstrap` 섹션의 변수들의 주석을 해제합니다
1. outputs 파일 `./outputs.tf`에서 `Specific to cloudbuild_module` 섹션의 출력들을 주석 처리합니다
1. outputs 파일 `./outputs.tf`에서 `Specific to tfc_bootstrap` 섹션의 출력들의 주석을 해제합니다
    1. [Terraform Cloud with Agents](https://developer.hashicorp.com/terraform/cloud-docs/agents)를 사용하려면, `Specific to tfc_bootstrap`에 추가로 `Specific to tfc_bootstrap with Terraform Cloud Agents` 섹션의 출력들의 주석을 해제하고 `terraform.tfvars`에서 `enable_tfc_cloud_agents` 변수를 `true`로 업데이트합니다
1. 파일 `./cb.tf`를 `./cb.tf.example`로 이름을 변경합니다

   ```bash
   mv ./cb.tf ./cb.tf.example
   ```

1. 파일 `.terraform_cloud.tf.example`을 `./terraform_cloud.tf`로 이름을 변경합니다

   ```bash
   mv ./terraform_cloud.tf.example ./terraform_cloud.tf
   ```

1. 파일 `terraform.example.tfvars`를 `terraform.tfvars`로 이름을 변경합니다

   ```bash
   mv ./terraform.example.tfvars ./terraform.tfvars
   ```

1. Google Cloud 환경의 값으로 `terraform.tfvars` 파일을 업데이트합니다
1. `tfc_bootstrap` 섹션에서 VCS 리포지토리의 값으로 `terraform.tfvars` 파일을 업데이트합니다
1. `tfc_bootstrap` 섹션에서 Terraform Cloud 조직의 값으로 `terraform.tfvars` 파일을 업데이트합니다
    1. [Terraform Cloud with Agents](https://developer.hashicorp.com/terraform/cloud-docs/agents)를 사용하려면 `tfc_bootstrap` 섹션에서 `enable_tfc_cloud_agents` 변수를 업데이트합니다
1. `terraform.tfvars` 파일에 `tfc_token`을 평문으로 저장하지 않기 위해
Terraform Cloud 토큰을 환경 변수로 내보냅니다:

   ```bash
   export TF_VAR_tfc_token=YOUR-TFC-TOKEN
   ```
1. `terraform.tfvars` 파일에 `vcs_oauth_token_id`를 평문으로 저장하지 않기 위해
OAuth 토큰 ID를 환경 변수로 내보냅니다:

   ```bash
   export TF_VAR_vcs_oauth_token_id=YOUR-VCS-OAUTH-TOKEN-ID
   ```

1. TF 버전을 가져오고 환경 변수로 내보내기 위해 `terraform version`을 실행합니다. `terraform_version` 변수는 TFC 워크스페이스에서 TF 버전을 설정하기 위해 `tfe_workspace` 리소스에서 사용됩니다. 이는 상태 마이그레이션(로컬에서 TFC로)이 작동하도록 하기 위해 중요합니다.

   ```bash
   TF_VAR_tfc_terraform_version=$(terraform --version -json | jq '.terraform_version' | sed 's/"//g')
   export TF_VAR_tfc_terraform_version
   echo "TF Version = ${TF_VAR_tfc_terraform_version}"
   ```

   참고: OS에 내장되어 있지 않은 경우 `jq`를 설치해야 할 수 있습니다. 대안으로 `terraform --version`을 실행하고 숫자 버전 출력을 수동으로 복사하여 환경 변수에 설정할 수 있습니다.

1. 환경을 검증하기 위해 도우미 스크립트 [validate-requirements.sh](../scripts/validate-requirements.sh)를 사용합니다:

   ```bash
   ../../../terraform-example-foundation/scripts/validate-requirements.sh  -o <ORGANIZATION_ID> -b <BILLING_ACCOUNT_ID> -u <END_USER_EMAIL> -e
   ```

   **참고:** 스크립트는 사용자가 필수 역할을 가진 Cloud Identity 또는 Google Workspace 그룹에 속해 있는지 검증할 수 없습니다.

1. `terraform init`과 `terraform plan`을 실행하고 출력을 검토합니다.

   ```bash
   terraform init
   terraform plan -input=false -out bootstrap.tfplan
   ```

1. `terraform apply`를 실행합니다.

   ```bash
   terraform apply bootstrap.tfplan
   ```

1. [Terraform Cloud with Agents](https://developer.hashicorp.com/terraform/cloud-docs/agents)를 사용하기 위해 `terraform.tfvars`에서 `enable_tfc_cloud_agents` 변수를 `true`로 설정한 경우 다음 추가 단계를 수행해야 합니다. 그렇지 않은 경우 건너뛰어야 합니다.
    1. `provider.tf` 파일에서 kubernetes 공급자 섹션의 주석을 해제;
    1. `terraform_cloud.tf` 파일에서 `tfc_agent_gke` 모듈의 `providers` 블록의 주석을 해제;
    1. `terraform plan -input=false -out bootstrap_2.tfplan` 실행
    1. `terraform apply bootstrap_2.tfplan` 실행

1. `3-networks-svpc`, `3-networks-hub-and-spoke`, `4-projects` 단계에서 `shared` 환경에 대한 수동 단계를 실행하는 데 사용될 terraform 서비스 계정의 이메일 주소를 가져오기 위해 `terraform output`을 실행합니다.

   ```bash
   export network_step_sa=$(terraform output -raw networks_step_terraform_service_account_email)
   export projects_step_sa=$(terraform output -raw projects_step_terraform_service_account_email)

   echo "network step service account = ${network_step_sa}"
   echo "projects step service account = ${projects_step_sa}"
   ```

1. CI/CD 프로젝트의 ID를 가져오기 위해 `terraform output`을 실행합니다:

   ```bash
   export cicd_project_id=$(terraform output -raw cicd_project_id)
   echo "CI/CD Project ID = ${cicd_project_id}"
   ```

1. TFC 조직의 이름을 가져오고 환경 변수로 내보내기 위해 `terraform output`을 실행합니다. `TF_CLOUD_ORGANIZATION` 변수는 로컬 Terraform의 상태를 TFC로 이동하기 위해 `cloud` 블록에서 사용되고, `TF_VAR_tfc_org_name`은 `3-networks-svpc`, `3-networks-hub-and-spoke`, `4-projects` 단계에서 `shared` 환경에 대한 수동 단계를 실행하는 데 사용됩니다

   ```bash
   export TF_CLOUD_ORGANIZATION=$(terraform output -raw tfc_org_name)
   export TF_VAR_tfc_org_name=$TF_CLOUD_ORGANIZATION
   echo "TFC Organization = ${TF_CLOUD_ORGANIZATION}"
   ```

1. TFC에 대해 파운데이션 단계를 구성하기 위해 다음 파일들의 이름을 변경해야 합니다:
   - TFC 워크스페이스 구성을 정의하고 Terraform의 상태를 TFC에 저장하기 위해 `backend.tf`를 `backend.tf.gcs.example`로, `backend.tf.cloud.example`을 `backend.tf`로 변경.
   - TFC의 워크스페이스에서 상태 출력을 가져오기 위해 `remote.tf`를 `remote.tf.gcs.example`로, `remote.tf.cloud.example`을 `remote.tf`로 변경.
   - **참고:** 모든 단계에서 이 이름 변경을 수행해야 합니다. `scripts/set-tfc-backend-and-remote.sh` 스크립트를 실행하여 모든 단계에 대해 자동으로 이름을 변경할 수 있습니다.

   ```bash
   mv backend.tf.cloud.example backend.tf
   cd ../../../
   chmod 775 ./terraform-example-foundation/scripts/set-tfc-backend-and-remote.sh
   ./terraform-example-foundation/scripts/set-tfc-backend-and-remote.sh
   cd ./gcp-bootstrap/envs/shared
   ```

1. `terraform init`을 다시 실행합니다. 프롬프트가 표시되면 Terraform 상태를 Terraform Cloud로 복사하는 것에 동의합니다.

   ```bash
   terraform init
   ```

1. (선택 사항) 상태가 올바르게 구성되었는지 확인하기 위해 `terraform plan`을 실행합니다. 이전 상태에서 변경사항이 없어야 합니다.
1. Terraform 구성을 `gcp-bootstrap` VCS 리포지토리에 저장합니다:

   ```bash
   cd ../..
   git add .
   git commit -m 'Initialize bootstrap repo'
   git push --set-upstream origin plan
   ```

1. `plan` 브랜치에서 `production` 브랜치로 [pull request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request)(GitHub) 또는 [merge request](https://docs.gitlab.com/ee/user/project/merge_requests/creating_merge_requests.html)(GitLab)를 열고 출력을 검토합니다.
1. pull request(또는 merge request)는 `production` 환경에서 Terraform Cloud [speculative plan](https://developer.hashicorp.com/terraform/cloud-docs/run/remote-operations#speculative-plans)을 트리거합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/0-shared/runs에서 `Run List` 항목 하에 speculative plan 출력을 검토합니다. 0-bootstrap을 로컬로 적용했으므로 출력은 `Your infrastructure matches the configuration`이어야 합니다.
1. speculative plan이 성공하면 pull request를 `production` 브랜치로 병합합니다.
1. 병합은 `production` 환경에 대해 terraform 구성에 대한 plan을 다시 실행하는 Terraform Cloud `Plan and Apply` 실행을 트리거합니다.

1. 다음 단계로 이동하기 전에 상위 디렉토리로 다시 이동합니다.

   ```bash
   cd ..
   ```

**참고:** 배포 후 [문제 해결 가이드](../docs/TROUBLESHOOTING.md#project-quota-exceeded)에 설명된 프로젝트 할당량 오류를 방지하기 위해
이 단계에서 생성된 **projects step service account**에 대해 50개의 추가 프로젝트를 요청하는 것을 권장합니다.

## 1-org 단계 배포

1. 리포지토리로 이동합니다. 모든 후속 단계는 `gcp-org` 디렉토리에서 실행한다고 가정합니다.
   다른 디렉토리에서 실행하는 경우 복사 경로를 적절히 조정하세요.

   ```bash
   cd gcp-org
   ```

1. 비프로덕션 브랜치로 변경합니다.

   ```bash
   git checkout -b plan
   ```

1. 파운데이션의 내용을 새 리포지토리로 복사합니다.

   ```bash
   cp -RT ../terraform-example-foundation/1-org/ .
   ```

1. `./envs/shared/terraform.example.tfvars`를 `./envs/shared/terraform.tfvars`로 이름을 변경합니다

   ```bash
   mv ./envs/shared/terraform.example.tfvars ./envs/shared/terraform.tfvars
   ```

1. GCP 환경의 값으로 `envs/shared/terraform.tfvars` 파일을 업데이트합니다.
`terraform.tfvars` 파일의 값에 대한 추가 정보는 shared 폴더 [README.md](../1-org/envs/shared/README.md#inputs)를 참조하세요.

1. 조직에 이미 Access Context Manager 정책이 있는 경우 `create_access_context_manager_access_policy = false` 변수의 주석을 해제합니다.

    ```bash
    export ORGANIZATION_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -json common_config | jq '.org_id' --raw-output)

    export ACCESS_CONTEXT_MANAGER_ID=$(gcloud access-context-manager policies list --organization ${ORGANIZATION_ID} --format="value(name)")

    echo "access_context_manager_policy_id = ${ACCESS_CONTEXT_MANAGER_ID}"

    if [ ! -z "${ACCESS_CONTEXT_MANAGER_ID}" ]; then sed -i "s=//create_access_context_manager_access_policy=create_access_context_manager_access_policy=" ./envs/shared/terraform.tfvars; fi
    ```

1. 조직에 기본 이름인 **scc-notify**로 된 Security Command Center 알림이 이미 존재하는지 확인합니다.

   ```bash
   export ORG_STEP_SA=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw organization_step_terraform_service_account_email)

   gcloud scc notifications describe "scc-notify" --format="value(name)" --organization=${ORGANIZATION_ID} --impersonate-service-account=${ORG_STEP_SA}
   ```

1. 알림이 존재하는 경우 출력은 다음과 같습니다:

    ```text
    organizations/ORGANIZATION_ID/notificationConfigs/scc-notify
    ```

1. 알림이 존재하지 않는 경우 출력은 다음과 같습니다:

    ```text
    ERROR: (gcloud.scc.notifications.describe) NOT_FOUND: Requested entity was not found.
    ```

1. 알림이 존재하는 경우 `./envs/shared/terraform.tfvars` 파일의 `scc_notification_name` 변수에 다른 값을 선택합니다.

1. 변경사항을 커밋합니다.

   ```bash
   git add .
   git commit -m 'Initialize org repo'
   ```

1. plan 브랜치를 푸시합니다.

   ```bash
   git push --set-upstream origin plan
   ```

1. `plan` 브랜치에서 `production` 브랜치로 [pull request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request)(GitHub) 또는 [merge request](https://docs.gitlab.com/ee/user/project/merge_requests/creating_merge_requests.html)(GitLab)를 열고 출력을 검토합니다.
1. pull request(또는 merge request)는 `production` 환경에서 Terraform Cloud [speculative plan](https://developer.hashicorp.com/terraform/cloud-docs/run/remote-operations#speculative-plans)을 트리거합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/1-shared/runs에서 `Run List` 항목 하에 speculative plan 출력을 검토합니다.
1. speculative plan이 성공하면 pull request를 `production` 브랜치로 병합합니다.
1. 병합은 `production` 환경에 대해 terraform 구성을 plan하고 적용하는 Terraform Cloud `Plan and Apply` 실행을 트리거합니다. `Runs` 메뉴에서 apply를 승인해야 합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/1-shared/runs에서 `Run List` 항목 하에 apply 출력을 검토합니다.

1. 다음 단계로 이동하기 전에 상위 디렉토리로 다시 이동합니다.

   ```bash
   cd ..
   ```


## 2-environments 단계 배포

1. 리포지토리로 이동합니다. 모든 후속 단계는 `gcp-environments` 디렉토리에서 실행한다고 가정합니다.
   다른 디렉토리에서 실행하는 경우 복사 경로를 적절히 조정하세요.

   ```bash
   cd gcp-environments
   ```

1. 비프로덕션 브랜치로 변경합니다.

   ```bash
   git checkout -b plan
   ```

1. 파운데이션의 내용을 새 리포지토리로 복사합니다.

   ```bash
   cp -RT ../terraform-example-foundation/2-environments/ .
   ```

1. `terraform.example.tfvars`를 `terraform.tfvars`로 이름을 변경합니다.

   ```bash
   mv terraform.example.tfvars terraform.tfvars
   ```

1. GCP 환경의 값으로 파일을 업데이트합니다.
`terraform.tfvars` 파일의 값에 대한 추가 정보는 envs 폴더 [README.md](../2-environments/envs/production/README.md#inputs) 파일들을 참조하세요.

1. 변경사항을 커밋합니다.

   ```bash
   git add .
   git commit -m 'Initialize environments repo'
   ```

1. plan 브랜치를 푸시합니다.

   ```bash
   git push --set-upstream origin plan
   ```

1. `plan` 브랜치에서 `development` 브랜치로 [pull request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request)(GitHub) 또는 [merge request](https://docs.gitlab.com/ee/user/project/merge_requests/creating_merge_requests.html)(GitLab)를 열고 출력을 검토합니다.
1. pull request(또는 merge request)는 `development` 환경에서 Terraform Cloud [speculative plan](https://developer.hashicorp.com/terraform/cloud-docs/run/remote-operations#speculative-plans)을 트리거합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/2-development/runs에서 `Run List` 항목 하에 speculative plan 출력을 검토합니다.
1. speculative plan이 성공하면 pull request를 `development` 브랜치로 병합합니다.
1. 병합은 `development` 환경에 대해 terraform 구성을 plan하고 적용하는 Terraform Cloud `Plan and Apply` 실행을 트리거합니다. `Runs` 메뉴에서 apply를 승인해야 합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/2-development/runs에서 `Run List` 항목 하에 apply 출력을 검토합니다.
1. TFC Apply가 성공하면 다음 환경에 대한 pull request(또는 merge request)를 열 수 있습니다.

1. `development` 브랜치에서 `nonproduction` 브랜치로 [pull request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request)(GitHub) 또는 [merge request](https://docs.gitlab.com/ee/user/project/merge_requests/creating_merge_requests.html)(GitLab)를 열고 출력을 검토합니다.
1. pull request(또는 merge request)는 `nonproduction` 환경에서 Terraform Cloud [speculative plan](https://developer.hashicorp.com/terraform/cloud-docs/run/remote-operations#speculative-plans)을 트리거합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/2-nonproduction/runs에서 `Run List` 항목 하에 speculative plan 출력을 검토합니다.
1. speculative plan이 성공하면 pull request를 `nonproduction` 브랜치로 병합합니다.
1. 병합은 `nonproduction` 환경에 대해 terraform 구성을 plan하고 적용하는 Terraform Cloud `Plan and Apply` 실행을 트리거합니다. `Runs` 메뉴에서 apply를 승인해야 합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/2-nonproduction/runs에서 `Run List` 항목 하에 apply 출력을 검토합니다.
1. TFC Apply가 성공하면 다음 환경에 대한 pull request(또는 merge request)를 열 수 있습니다.

1. `nonproduction` 브랜치에서 `production` 브랜치로 [pull request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request)(GitHub) 또는 [merge request](https://docs.gitlab.com/ee/user/project/merge_requests/creating_merge_requests.html)(GitLab)를 열고 출력을 검토합니다.
1. pull request(또는 merge request)는 `production` 환경에서 Terraform Cloud [speculative plan](https://developer.hashicorp.com/terraform/cloud-docs/run/remote-operations#speculative-plans)을 트리거합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/2-production/runs에서 `Run List` 항목 하에 speculative plan 출력을 검토합니다.
1. speculative plan이 성공하면 pull request를 `production` 브랜치로 병합합니다.
1. 병합은 `production` 환경에 대해 terraform 구성을 plan하고 적용하는 Terraform Cloud `Plan and Apply` 실행을 트리거합니다. `Runs` 메뉴에서 apply를 승인해야 합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/2-production/runs에서 `Run List` 항목 하에 apply 출력을 검토합니다.

1. 이제 네트워크 단계의 지침으로 이동할 수 있습니다.
[Dual Shared VPC](https://cloud.google.com/architecture/security-foundations/networking#vpcsharedvpc-id7-1-shared-vpc-) 네트워크 모드를 사용하려면 [3-networks-svpc 단계 배포](#deploying-step-3-networks-svpc)로 이동하거나,
[Hub and Spoke](https://cloud.google.com/architecture/security-foundations/networking#hub-and-spoke) 네트워크 모드를 사용하려면 [3-networks-hub-and-spoke 단계 배포](#deploying-step-3-networks-hub-and-spoke)로 이동하세요.

1. 다음 단계로 이동하기 전에 상위 디렉토리로 다시 이동합니다.

   ```bash
   cd ..
   ```

## 3-networks-svpc 단계 배포

**참고:** `production`에 미칠 수 있는 영향 때문에 모든 목적상 `shared` 환경을 `production` 환경으로 취급합니다. 따라서 `3-production` TFC 워크스페이스는 `3-shared` TFC 워크스페이스를 소스로 하는 [Run Trigger](https://developer.hashicorp.com/terraform/cloud-docs/workspaces/settings/run-triggers)를 가지고 있는데, 이는 `3-shared` TFC 워크스페이스에서 apply 작업을 성공적으로 실행할 때마다 `3-production` TFC 워크스페이스에 대해 `Plan and apply` 작업이 자동으로 트리거된다는 의미입니다. (모든 apply는 TFC 콘솔에서 계속 수동 승인을 요구합니다).

1. 리포지토리로 이동합니다. 모든 후속 단계는 `gcp-networks` 디렉토리에서 실행한다고 가정합니다.
   다른 디렉토리에서 실행하는 경우 복사 경로를 적절히 조정하세요.

   ```bash
   cd gcp-networks
   ```

1. 비프로덕션 브랜치로 변경합니다.

   ```bash
   git checkout -b plan
   ```

1. 파운데이션의 내용을 새 리포지토리로 복사합니다.

   ```bash
   cp -RT ../terraform-example-foundation/3-networks-svpc/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. `common.auto.example.tfvars`를 `common.auto.tfvars`로, `production.auto.example.tfvars`를 `production.auto.tfvars`로, `access_context.auto.example.tfvars`를 `access_context.auto.tfvars`로 이름을 변경합니다.

   ```bash
   mv common.auto.example.tfvars common.auto.tfvars
   mv production.auto.example.tfvars production.auto.tfvars
   mv access_context.auto.example.tfvars access_context.auto.tfvars
   ```

1. `target_name_server_addresses` 값으로 `production.auto.tfvars` 파일을 업데이트합니다.
1. 조직의 `access_context_manager_policy_id`로 `access_context.auto.tfvars` 파일을 업데이트합니다.

   ```bash
   export ORGANIZATION_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -json common_config | jq '.org_id' --raw-output)

   export ACCESS_CONTEXT_MANAGER_ID=$(gcloud access-context-manager policies list --organization ${ORGANIZATION_ID} --format="value(name)")

   echo "access_context_manager_policy_id = ${ACCESS_CONTEXT_MANAGER_ID}"

   sed -i "s/ACCESS_CONTEXT_MANAGER_ID/${ACCESS_CONTEXT_MANAGER_ID}/" ./access_context.auto.tfvars
   ```

1. GCP 환경의 값으로 `common.auto.tfvars` 파일을 업데이트합니다.
`common.auto.tfvars` 파일의 값에 대한 추가 정보는 envs 폴더 [README.md](../3-networks-svpc/envs/production/README.md#inputs) 파일들을 참조하세요.
1. 프로젝트에서 생성된 리소스를 볼 수 있도록 `perimeter_additional_members` 변수에 사용자 이메일을 추가해야 합니다.

1. `development`, `nonproduction`, `production` 환경이 이에 따라 달라지므로 `shared` 환경을 (한 번만) 수동으로 plan하고 apply해야 합니다.

1. 로컬에서 shared 워크스페이스에 대한 apply를 수동으로 실행하기 위해 `envs/shared/backend.tf`를 `envs/shared/backend.tf.temporary_disabled`로 이름을 변경하여 TFC 백엔드를 일시적으로 해제해야 합니다.

   ```bash
   mv envs/shared/backend.tf envs/shared/backend.tf.temporary_disabled
   ```

1. gcp-bootstrap 출력에서 CI/CD 프로젝트 ID와 networks step Terraform 서비스 계정을 가져오기 위해 `terraform output`을 사용합니다.
1. CI/CD 프로젝트 ID는 Terraform 구성의 [검증](https://cloud.google.com/docs/terraform/policy-validation/quickstart)에 사용됩니다

   ```bash
   export CICD_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -raw cicd_project_id)
   echo ${CICD_PROJECT_ID}
   ```
1. gcp-bootstrap 출력에서 TFC 조직의 이름을 가져오고 환경 변수로 내보내기 위해 `terraform output`을 사용합니다. TFC 조직은 이전 단계의 출력을 가져오기 위해 `tfe_outputs` 리소스가 수동 apply 프로세스 동안 사용합니다.

   ```bash
   export TF_VAR_tfc_org_name=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -raw tfc_org_name)
   echo "TFC Organization = ${TF_VAR_tfc_org_name}"
   ```

1. networks step Terraform 서비스 계정은 다음 단계에서 [서비스 계정 가장](https://cloud.google.com/docs/authentication/use-service-account-impersonation)에 사용됩니다.
가장을 활성화하기 위해 Terraform 서비스 계정으로 환경 변수 `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT`가 설정됩니다.

   ```bash
   export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -raw networks_step_terraform_service_account_email)
   echo ${GOOGLE_IMPERSONATE_SERVICE_ACCOUNT}
   ```

1. shared 환경에 대해 `init`과 `plan`을 실행하고 출력을 검토합니다.

   ```bash
   ./tf-wrapper.sh init shared
   ./tf-wrapper.sh plan shared
   ```

1. `tf-wrapper.sh` 스크립트의 `validate` 옵션을 사용하려면 terraform-tools 컴포넌트를 설치하는 [지침](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install)을 따르세요.
1. `validate`를 실행하고 위반 사항을 확인합니다.

   ```bash
   ./tf-wrapper.sh validate shared $(pwd)/policy-library ${CICD_PROJECT_ID}
   ```

1. shared에 대해 `apply`를 실행합니다.

   ```bash
   ./tf-wrapper.sh apply shared
   ```

   **참고:** TFC 워크스페이스 대신 로컬에서 apply를 실행하고 있기 때문에, shared에 대한 이 apply는 `3-production` TFC 워크스페이스에 대한 [TFC Run Trigger](https://developer.hashicorp.com/terraform/cloud-docs/workspaces/settings/run-triggers)를 트리거하지 않습니다.

1. In order to set the TFC backend for shared workspace we now can rename `envs/shared/backend.tf.temporary_disabled` to `envs/shared/backend.tf` and run `terraform init`. When you're prompted, agree to copy Terraform state to Terraform Cloud.

   ```bash
   cd envs/shared/
   mv backend.tf.temporary_disabled backend.tf
   terraform init
   cd ../..
   ```

1. Commit changes

   ```bash
   git add .
   git commit -m 'Initialize networks repo'
   ```

1. Push your plan branch.

   ```bash
   git push --set-upstream origin plan
   ```

1. `plan` 브랜치에서 `production` 브랜치로 [pull request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request)(GitHub) 또는 [merge request](https://docs.gitlab.com/ee/user/project/merge_requests/creating_merge_requests.html)(GitLab)를 열고 출력을 검토합니다.
1. pull request(또는 merge request)는 `production` 환경에서 Terraform Cloud [speculative plan](https://developer.hashicorp.com/terraform/cloud-docs/run/remote-operations#speculative-plans)을 트리거합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/3-production/runs에서 `Run List` 항목 하에 speculative plan 출력을 검토합니다.
1. speculative plan이 성공하면 pull request를 `production` 브랜치로 병합합니다.
1. 병합은 `production` 환경에 대해 terraform 구성을 plan하고 적용하는 Terraform Cloud `Plan and Apply` 실행을 트리거합니다. `Runs` 메뉴에서 apply를 승인해야 합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/3-production/runs에서 `Run List` 항목 하에 apply 출력을 검토합니다.
1. TFC Apply가 성공하면 다음 환경에 대한 pull request(또는 merge request)를 열 수 있습니다.

1. `plan` 브랜치에서 `development` 브랜치로 [pull request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request)(GitHub) 또는 [merge request](https://docs.gitlab.com/ee/user/project/merge_requests/creating_merge_requests.html)(GitLab)를 열고 출력을 검토합니다.
1. pull request(또는 merge request)는 `development` 환경에서 Terraform Cloud [speculative plan](https://developer.hashicorp.com/terraform/cloud-docs/run/remote-operations#speculative-plans)을 트리거합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/3-development/runs에서 `Run List` 항목 하에 speculative plan 출력을 검토합니다.
1. speculative plan이 성공하면 pull request를 `development` 브랜치로 병합합니다.
1. 병합은 `development` 환경에 대해 terraform 구성을 plan하고 적용하는 Terraform Cloud `Plan and Apply` 실행을 트리거합니다. `Runs` 메뉴에서 apply를 승인해야 합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/3-development/runs에서 `Run List` 항목 하에 apply 출력을 검토합니다.
1. TFC Apply가 성공하면 다음 환경에 대한 pull request(또는 merge request)를 열 수 있습니다.

1. `development` 브랜치에서 `nonproduction` 브랜치로 [pull request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request)(GitHub) 또는 [merge request](https://docs.gitlab.com/ee/user/project/merge_requests/creating_merge_requests.html)(GitLab)를 열고 출력을 검토합니다.
1. pull request(또는 merge request)는 `nonproduction` 환경에서 Terraform Cloud [speculative plan](https://developer.hashicorp.com/terraform/cloud-docs/run/remote-operations#speculative-plans)을 트리거합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/3-nonproduction/runs에서 `Run List` 항목 하에 speculative plan 출력을 검토합니다.
1. speculative plan이 성공하면 pull request를 `nonproduction` 브랜치로 병합합니다.
1. 병합은 `nonproduction` 환경에 대해 terraform 구성을 plan하고 적용하는 Terraform Cloud `Plan and Apply` 실행을 트리거합니다. `Runs` 메뉴에서 apply를 승인해야 합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/3-nonproduction/runs에서 `Run List` 항목 하에 apply 출력을 검토합니다.
1. TFC Apply가 성공하면 다음 환경에 대한 pull request(또는 merge request)를 열 수 있습니다.

1. 다음 단계를 실행하기 전에 `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` 환경 변수를 해제합니다.

   ```bash
   unset GOOGLE_IMPERSONATE_SERVICE_ACCOUNT
   ```

1. 다음 단계로 이동하기 전에 상위 디렉토리로 다시 이동합니다.

   ```bash
   cd ..
   ```

1. 이제 [4-projects](#deploying-step-4-projects) 단계의 지침으로 이동할 수 있습니다.

## 3-networks-hub-and-spoke 단계 배포

**참고:** `production`에 미칠 수 있는 영향 때문에 모든 목적상 `shared` 환경을 `production` 환경으로 취급합니다. 따라서 `3-production` TFC 워크스페이스는 `3-shared` TFC 워크스페이스를 소스로 하는 [Run Trigger](https://developer.hashicorp.com/terraform/cloud-docs/workspaces/settings/run-triggers)를 가지고 있는데, 이는 `3-shared` TFC 워크스페이스에서 apply 작업을 성공적으로 실행할 때마다 `3-production` TFC 워크스페이스에 대해 `Plan and apply` 작업이 자동으로 트리거된다는 의미입니다. (모든 apply는 TFC 콘솔에서 계속 수동 승인을 요구합니다).

1. 리포지토리로 이동합니다. 모든 후속 단계는 `gcp-networks` 디렉토리에서 실행한다고 가정합니다.
   다른 디렉토리에서 실행하는 경우 복사 경로를 적절히 조정하세요.

   ```bash
   cd gcp-networks
   ```

1. 비프로덕션 브랜치로 변경합니다.

   ```bash
   git checkout -b plan
   ```

1. 파운데이션의 내용을 새 리포지토리로 복사합니다.

   ```bash
   cp -RT ../terraform-example-foundation/3-networks-hub-and-spoke/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. `common.auto.example.tfvars`를 `common.auto.tfvars`로, `shared.auto.example.tfvars`를 `shared.auto.tfvars`로, `access_context.auto.example.tfvars`를 `access_context.auto.tfvars`로 이름을 변경합니다.

   ```bash
   mv common.auto.example.tfvars common.auto.tfvars
   mv shared.auto.example.tfvars shared.auto.tfvars
   mv access_context.auto.example.tfvars access_context.auto.tfvars
   ```

1. GCP 환경의 값으로 `common.auto.tfvars` 파일을 업데이트합니다.
`common.auto.tfvars` 파일의 값에 대한 추가 정보는 envs 폴더 [README.md](../3-networks-hub-and-spoke/envs/production/README.md#inputs) 파일들을 참조하세요.
1. 프로젝트에서 생성된 리소스를 보기 위해 `perimeter_additional_members` 변수에 사용자 이메일을 추가해야 합니다.

1. `development`, `nonproduction`, `production` 환경이 이에 의존하므로 `shared` 환경을 (한 번만) 수동으로 plan하고 apply해야 합니다.

1. 로컬에서 shared 워크스페이스에 대한 apply를 수동으로 실행하기 위해 `envs/shared/backend.tf`를 `envs/shared/backend.tf.temporary_disabled`로 이름을 변경하여 TFC 백엔드를 일시적으로 해제해야 합니다.

   ```bash
   mv envs/shared/backend.tf envs/shared/backend.tf.temporary_disabled
   ```

1. gcp-bootstrap 출력에서 CI/CD 프로젝트 ID와 networks step Terraform 서비스 계정을 가져오기 위해 `terraform output`을 사용합니다.
1. CI/CD 프로젝트 ID는 Terraform 구성의 [검증](https://cloud.google.com/docs/terraform/policy-validation/quickstart)에 사용됩니다

   ```bash
   export CICD_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -raw cicd_project_id)
   echo ${CICD_PROJECT_ID}
   ```

1. gcp-bootstrap 출력에서 TFC 조직의 이름을 가져오고 환경 변수로 내보내기 위해 `terraform output`을 사용합니다. TFC 조직은 이전 단계의 출력을 가져오기 위해 `tfe_outputs` 리소스가 수동 apply 프로세스 동안 사용합니다.

   ```bash
   export TF_VAR_tfc_org_name=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -raw tfc_org_name)
   echo "TFC Organization = ${TF_VAR_tfc_org_name}"
   ```

1. networks step Terraform 서비스 계정은 다음 단계에서 [서비스 계정 가장](https://cloud.google.com/docs/authentication/use-service-account-impersonation)에 사용됩니다.
가장을 활성화하기 위해 Terraform 서비스 계정으로 환경 변수 `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT`가 설정됩니다.

   ```bash
   export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -raw networks_step_terraform_service_account_email)
   echo ${GOOGLE_IMPERSONATE_SERVICE_ACCOUNT}
   ```

1. shared 환경에 대해 `init`과 `plan`을 실행하고 출력을 검토합니다.

   ```bash
   ./tf-wrapper.sh init shared
   ./tf-wrapper.sh plan shared
   ```

1. `tf-wrapper.sh` 스크립트의 `validate` 옵션을 사용하려면 terraform-tools 컴포넌트를 설치하는 [지침](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install)을 따르세요.
1. `validate`를 실행하고 위반 사항을 확인합니다.

   ```bash
   ./tf-wrapper.sh validate shared $(pwd)/policy-library ${CICD_PROJECT_ID}
   ```

1. shared에 대해 `apply`를 실행합니다.

   ```bash
   ./tf-wrapper.sh apply shared
   ```
   **참고:** TFC 워크스페이스 대신 로컬에서 apply를 실행하고 있기 때문에, shared에 대한 이 apply는 `3-production` TFC 워크스페이스에 대한 [TFC Run Trigger](https://developer.hashicorp.com/terraform/cloud-docs/workspaces/settings/run-triggers)를 트리거하지 않습니다.

1. shared 워크스페이스에 대한 TFC 백엔드를 설정하기 위해 이제 `envs/shared/backend.tf.temporary_disabled`를 `envs/shared/backend.tf`로 이름을 변경하고 `terraform init`을 실행할 수 있습니다. 프롬프트가 표시되면 Terraform 상태를 Terraform Cloud로 복사하는 것에 동의합니다.

   ```bash
   cd envs/shared/
   mv backend.tf.temporary_disabled backend.tf
   terraform init
   cd ../..
   ```

1. 변경사항을 커밋합니다

   ```bash
   git add .
   git commit -m 'Initialize networks repo'
   ```

1. plan 브랜치를 푸시합니다.

   ```bash
   git push --set-upstream origin plan
   ```

1. `plan` 브랜치에서 `development` 브랜치로 [pull request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request)(GitHub) 또는 [merge request](https://docs.gitlab.com/ee/user/project/merge_requests/creating_merge_requests.html)(GitLab)를 열고 출력을 검토합니다.
1. pull request(또는 merge request)는 `development` 환경에서 Terraform Cloud [speculative plan](https://developer.hashicorp.com/terraform/cloud-docs/run/remote-operations#speculative-plans)을 트리거합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/3-development/runs에서 `Run List` 항목 하에 speculative plan 출력을 검토합니다.
1. speculative plan이 성공하면 pull request를 `development` 브랜치로 병합합니다.
1. 병합은 `development` 환경에 대해 terraform 구성을 plan하고 적용하는 Terraform Cloud `Plan and Apply` 실행을 트리거합니다. `Runs` 메뉴에서 apply를 승인해야 합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/3-development/runs에서 `Run List` 항목 하에 apply 출력을 검토합니다.
1. TFC Apply가 성공하면 다음 환경에 대한 pull request(또는 merge request)를 열 수 있습니다.

1. `development` 브랜치에서 `nonproduction` 브랜치로 [pull request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request)(GitHub) 또는 [merge request](https://docs.gitlab.com/ee/user/project/merge_requests/creating_merge_requests.html)(GitLab)를 열고 출력을 검토합니다.
1. pull request(또는 merge request)는 `nonproduction` 환경에서 Terraform Cloud [speculative plan](https://developer.hashicorp.com/terraform/cloud-docs/run/remote-operations#speculative-plans)을 트리거합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/3-nonproduction/runs에서 `Run List` 항목 하에 speculative plan 출력을 검토합니다.
1. speculative plan이 성공하면 pull request를 `nonproduction` 브랜치로 병합합니다.
1. 병합은 `nonproduction` 환경에 대해 terraform 구성을 plan하고 적용하는 Terraform Cloud `Plan and Apply` 실행을 트리거합니다. `Runs` 메뉴에서 apply를 승인해야 합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/3-nonproduction/runs에서 `Run List` 항목 하에 apply 출력을 검토합니다.
1. TFC Apply가 성공하면 다음 환경에 대한 pull request(또는 merge request)를 열 수 있습니다.

1. `nonproduction` 브랜치에서 `production` 브랜치로 [pull request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request)(GitHub) 또는 [merge request](https://docs.gitlab.com/ee/user/project/merge_requests/creating_merge_requests.html)(GitLab)를 열고 출력을 검토합니다.
1. pull request(또는 merge request)는 `production` 환경에서 Terraform Cloud [speculative plan](https://developer.hashicorp.com/terraform/cloud-docs/run/remote-operations#speculative-plans)을 트리거합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/3-production/runs에서 `Run List` 항목 하에 speculative plan 출력을 검토합니다.
1. speculative plan이 성공하면 pull request를 `production` 브랜치로 병합합니다.
1. 병합은 `production` 환경에 대해 terraform 구성을 plan하고 적용하는 Terraform Cloud `Plan and Apply` 실행을 트리거합니다. `Runs` 메뉴에서 apply를 승인해야 합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/3-production/runs에서 `Run List` 항목 하에 apply 출력을 검토합니다.


1. 다음 단계를 실행하기 전에 `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` 환경 변수를 해제합니다.

   ```bash
   unset GOOGLE_IMPERSONATE_SERVICE_ACCOUNT
   ```

1. 다음 단계로 이동하기 전에 상위 디렉토리로 다시 이동합니다.

   ```bash
   cd ..
   ```

1. 이제 [4-projects](#deploying-step-4-projects) 단계의 지침으로 이동할 수 있습니다.

## 4-projects 단계 배포

**참고:** `production`에 미칠 수 있는 영향 때문에 모든 목적상 `shared` 환경을 `production` 환경으로 취급합니다. 따라서 `4-<business_unit>-production` TFC 워크스페이스는 `4-<business_unit>-shared` TFC 워크스페이스를 소스로 하는 [Run Trigger](https://developer.hashicorp.com/terraform/cloud-docs/workspaces/settings/run-triggers)를 가지고 있는데, 이는 `4-<business_unit>-shared` TFC 워크스페이스에서 apply 작업을 성공적으로 실행할 때마다 `4-<business_unit>-production` TFC 워크스페이스에 대해 `Plan and apply` 작업이 자동으로 트리거된다는 의미입니다. (모든 apply는 TFC 콘솔에서 계속 수동 승인을 요구합니다).

1. 리포지토리로 이동합니다. 모든 후속 단계는 `gcp-projects` 디렉토리에서 실행한다고 가정합니다.
   다른 디렉토리에서 실행하는 경우 복사 경로를 적절히 조정하세요.

   ```bash
   cd gcp-projects
   ```

1. 비프로덕션 브랜치로 변경합니다.

   ```bash
   git checkout -b plan
   ```

1. 파운데이션의 내용을 새 리포지토리로 복사합니다.

   ```bash
   cp -RT ../terraform-example-foundation/4-projects/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. `auto.example.tfvars` 파일들의 이름을 `auto.tfvars`로 변경합니다.

   ```bash
   mv common.auto.example.tfvars common.auto.tfvars
   mv shared.auto.example.tfvars shared.auto.tfvars
   mv development.auto.example.tfvars development.auto.tfvars
   mv nonproduction.auto.example.tfvars nonproduction.auto.tfvars
   mv production.auto.example.tfvars production.auto.tfvars
   ```

1. `common.auto.tfvars`, `development.auto.tfvars`, `nonproduction.auto.tfvars`, `production.auto.tfvars` 파일의 값에 대한 추가 정보는 envs 폴더 [README.md](../4-projects/business_unit_1/production/README.md#inputs) 파일들을 참조하세요.
1. `shared.auto.tfvars` 파일의 값에 대한 추가 정보는 shared 폴더 [README.md](../4-projects/business_unit_1/shared/README.md#inputs) 파일들을 참조하세요.

1. `development`, `nonproduction`, `production`이 이에 의존하므로 `business_unit_1/shared`와 `business_unit_2/shared` 환경을 한 번만 수동으로 plan하고 apply해야 합니다.

1. 로컬에서 shared 워크스페이스에 대한 apply를 수동으로 실행하기 위해 `envs/shared/backend.tf`를 `envs/shared/backend.tf.temporary_disabled`로 이름을 변경하여 TFC 백엔드를 일시적으로 해제해야 합니다.

   ```bash
   mv business_unit_1/shared/backend.tf business_unit_1/shared/backend.tf.temporary_disabled
   mv business_unit_2/shared/backend.tf business_unit_2/shared/backend.tf.temporary_disabled
   ```

1. gcp-bootstrap 출력에서 CI/CD 프로젝트 ID와 projects step Terraform 서비스 계정을 가져오기 위해 `terraform output`을 사용합니다.
1. CI/CD 프로젝트 ID는 Terraform 구성의 [검증](https://cloud.google.com/docs/terraform/policy-validation/quickstart)에 사용됩니다

   ```bash
   export CICD_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -raw cicd_project_id)
   echo ${CICD_PROJECT_ID}
   ```

1. gcp-bootstrap 출력에서 TFC 조직의 이름을 가져오고 환경 변수로 내보내기 위해 `terraform output`을 사용합니다. TFC 조직은 이전 단계의 출력을 가져오기 위해 `tfe_outputs` 리소스가 수동 apply 프로세스 동안 사용합니다.

   ```bash
   export TF_VAR_tfc_org_name=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -raw tfc_org_name)
   echo "TFC Organization = ${TF_VAR_tfc_org_name}"
   ```

1. projects step Terraform 서비스 계정은 다음 단계에서 [서비스 계정 가장](https://cloud.google.com/docs/authentication/use-service-account-impersonation)에 사용됩니다.
가장을 활성화하기 위해 Terraform 서비스 계정으로 환경 변수 `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT`가 설정됩니다.

   ```bash
   export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -raw projects_step_terraform_service_account_email)
   echo ${GOOGLE_IMPERSONATE_SERVICE_ACCOUNT}
   ```

1. shared 환경에 대해 `init`과 `plan`을 실행하고 출력을 검토합니다.

   ```bash
   ./tf-wrapper.sh init shared
   ./tf-wrapper.sh plan shared
   ```

1. `tf-wrapper.sh` 스크립트의 `validate` 옵션을 사용하려면 terraform-tools 컴포넌트를 설치하는 [지침](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install)을 따르세요.
1. `validate`를 실행하고 위반 사항을 확인합니다.

   ```bash
   ./tf-wrapper.sh validate shared $(pwd)/policy-library ${CICD_PROJECT_ID}
   ```

1. shared에 대해 `apply`를 실행합니다.

   ```bash
   ./tf-wrapper.sh apply shared
   ```
   **참고:** TFC 워크스페이스 대신 로컬에서 apply를 실행하고 있기 때문에, shared에 대한 이 apply는 `4-<business_unit>-production` TFC 워크스페이스에 대한 [TFC Run Trigger](https://developer.hashicorp.com/terraform/cloud-docs/workspaces/settings/run-triggers)를 트리거하지 않습니다.


1. shared 워크스페이스에 대한 TFC 백엔드를 설정하기 위해 이제 `envs/shared/backend.tf.temporary_disabled`를 `envs/shared/backend.tf`로 이름을 변경하고 `terraform init`을 실행할 수 있습니다. 프롬프트가 표시되면 Terraform 상태를 Terraform Cloud로 복사하는 것에 동의합니다.

   ```bash
   mv business_unit_1/shared/backend.tf.temporary_disabled business_unit_1/shared/backend.tf
   mv business_unit_2/shared/backend.tf.temporary_disabled business_unit_2/shared/backend.tf
   terraform -chdir="business_unit_1/shared/" init
   terraform -chdir="business_unit_2/shared/" init
   ```

1. (선택 사항) 별도의 비즈니스 단위 또는 엔티티에 대한 추가 하위 폴더를 원하는 경우, `business_unit_1` 폴더의 추가 복사본을 만들고 `business_code`, `business_unit` 또는 `subnet_ip_range`와 같이 비즈니스 단위마다 달라지는 값들을 수정하세요.

예를 들어, business_unit_1과 비슷한 새 비즈니스 단위를 만들려면 다음을 실행하세요:

   ```bash
   #business_unit_1 폴더와 그 내용을 새 폴더 business_unit_2로 복사
   cp -r  business_unit_1 business_unit_2

   # `business_unit_2` 폴더 하에서 모든 파일을 검색하고 business_unit_1에 대한 문자열을 business_unit_2에 대한 문자열로 교체
   grep -rl bu1 business_unit_2/ | xargs sed -i 's/bu1/bu2/g'
   grep -rl business_unit_1 business_unit_2/ | xargs sed -i 's/business_unit_1/business_unit_2/g'
   ```


1. 변경사항을 커밋합니다

   ```bash
   git add .
   git commit -m 'Initialize networks repo'
   ```

1. plan 브랜치를 푸시합니다.

   ```bash
   git push --set-upstream origin plan
   ```

1. `plan` 브랜치에서 `development` 브랜치로 [pull request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request)(GitHub) 또는 [merge request](https://docs.gitlab.com/ee/user/project/merge_requests/creating_merge_requests.html)(GitLab)를 열고 출력을 검토합니다.
1. pull request(또는 merge request)는 `development` 환경에서 Terraform Cloud [speculative plan](https://developer.hashicorp.com/terraform/cloud-docs/run/remote-operations#speculative-plans)을 트리거합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/4-development/runs에서 `Run List` 항목 하에 speculative plan 출력을 검토합니다.
1. speculative plan이 성공하면 pull request를 `development` 브랜치로 병합합니다.
1. 병합은 `development` 환경에 대해 terraform 구성을 plan하고 적용하는 Terraform Cloud `Plan and Apply` 실행을 트리거합니다. `Runs` 메뉴에서 apply를 승인해야 합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/4-development/runs에서 `Run List` 항목 하에 apply 출력을 검토합니다.
1. TFC Apply가 성공하면 다음 환경에 대한 pull request(또는 merge request)를 열 수 있습니다.

1. `development` 브랜치에서 `nonproduction` 브랜치로 [pull request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request)(GitHub) 또는 [merge request](https://docs.gitlab.com/ee/user/project/merge_requests/creating_merge_requests.html)(GitLab)를 열고 출력을 검토합니다.
1. pull request(또는 merge request)는 `nonproduction` 환경에서 Terraform Cloud [speculative plan](https://developer.hashicorp.com/terraform/cloud-docs/run/remote-operations#speculative-plans)을 트리거합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/4-nonproduction/runs에서 `Run List` 항목 하에 speculative plan 출력을 검토합니다.
1. speculative plan이 성공하면 pull request를 `nonproduction` 브랜치로 병합합니다.
1. 병합은 `nonproduction` 환경에 대해 terraform 구성을 plan하고 적용하는 Terraform Cloud `Plan and Apply` 실행을 트리거합니다. `Runs` 메뉴에서 apply를 승인해야 합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/4-nonproduction/runs에서 `Run List` 항목 하에 apply 출력을 검토합니다.
1. TFC Apply가 성공하면 다음 환경에 대한 pull request(또는 merge request)를 열 수 있습니다.

1. `nonproduction` 브랜치에서 `production` 브랜치로 [pull request](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/creating-a-pull-request)(GitHub) 또는 [merge request](https://docs.gitlab.com/ee/user/project/merge_requests/creating_merge_requests.html)(GitLab)를 열고 출력을 검토합니다.
1. pull request(또는 merge request)는 `production` 환경에서 Terraform Cloud [speculative plan](https://developer.hashicorp.com/terraform/cloud-docs/run/remote-operations#speculative-plans)을 트리거합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/4-production/runs에서 `Run List` 항목 하에 speculative plan 출력을 검토합니다.
1. speculative plan이 성공하면 pull request를 `production` 브랜치로 병합합니다.
1. 병합은 `production` 환경에 대해 terraform 구성을 plan하고 적용하는 Terraform Cloud `Plan and Apply` 실행을 트리거합니다. `Runs` 메뉴에서 apply를 승인해야 합니다.
1. Terraform Cloud https://app.terraform.io/app/TFC-ORGANIZATION-NAME/workspaces/4-production/runs에서 `Run List` 항목 하에 apply 출력을 검토합니다.

1. `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` 환경 변수를 해제합니다.

   ```bash
   unset GOOGLE_IMPERSONATE_SERVICE_ACCOUNT
   ```
