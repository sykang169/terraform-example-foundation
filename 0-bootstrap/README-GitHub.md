# GitHub Actions 호환 환경 배포

아래 지침의 목표는 Terraform Example Foundation 단계(`0-bootstrap`, `1-org`, `2-environments`, `3-networks`, `4-projects`)에 대해 GitHub Actions를 사용하여 CI/CD 배포를 실행할 수 있는 인프라를 구성하는 것입니다.
이 인프라는 두 개의 Google Cloud Platform 프로젝트(`prj-b-seed` 및 `prj-b-cicd-wif-gh`)로 구성됩니다.

관심사 분리를 위해 여기서 두 개의 별도 프로젝트(`prj-b-seed` 및 `prj-b-cicd-wif-gh`)를 갖는 것이 모범 사례입니다.
한편으로, `prj-b-seed`는 terraform 상태를 저장하고 인프라를 생성/수정할 수 있는 서비스 계정을 보유합니다.
반면에, [워크로드 ID 페더레이션](https://cloud.google.com/iam/docs/workload-identity-federation)을 사용한 인증 인프라는 `prj-b-cicd-wif-gh`에서 구현됩니다.

## 요구사항

이 문서에 설명된 지침을 실행하려면 다음을 설치하십시오:

- [Google Cloud SDK](https://cloud.google.com/sdk/install) 버전 393.0.0 이상
    - [terraform-tools](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install) 구성 요소
- [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) 버전 2.28.0 이상
- [Terraform](https://www.terraform.io/downloads.html) 버전 1.5.7 이상

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

- 사용자 또는 [조직](https://docs.github.com/en/organizations/collaborating-with-groups-in-organizations/creating-a-new-organization-from-scratch)의 [GitHub 계정](https://docs.github.com/en/get-started/onboarding/getting-started-with-your-github-account)
- Foundation의 각 단계에 대한 **비공개** [GitHub 저장소](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-new-repository):
    - Bootstrap
    - Organization
    - Environments
    - Networks
    - Projects
- 다음과 같이 구성된 [세분화된](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token#creating-a-fine-grained-personal-access-token) 개인 액세스 토큰:
    - 이전 항목의 모든 저장소를 포함하여 선택한 저장소에만 저장소 액세스
    - 권한:
        - Actions: 읽기 및 쓰기
        - Metadata: 읽기 전용
        - Secrets: 읽기 및 쓰기
        - Variables: 읽기 및 쓰기
        - Workflows: 읽기 및 쓰기
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

[첫 번째 커밋 푸시가 2개의 워크플로우 실행을 트리거함](https://github.com/orgs/community/discussions/50356) 문제로 인해,
생성된 저장소는 초기 커밋으로 초기화되어야 하며,
이는 `plan` 브랜치에 대한 초기 푸시가 두 개의 워크플로우 실행을 트리거하지 않도록 하기 위함입니다.
다음 단계의 지침은 각 저장소에 대한 초기 커밋 생성을 보장합니다.

### 0-bootstrap 단계 배포

1. 로컬 환경에 [terraform-example-foundation](https://github.com/terraform-google-modules/terraform-example-foundation)을 복제합니다.

   ```bash
   git clone https://github.com/terraform-google-modules/terraform-example-foundation.git
   ```

1. `terraform-example-foundation` 폴더와 같은 수준에서 `0-bootstrap` terraform 구성을 호스팅하기 위해 생성한 비공개 저장소를 복제합니다.
[GitHub에 인증](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/about-authentication-to-github)되어 있어야 합니다.

   ```bash
   git clone git@github.com:<GITHUB-OWNER>/<GITHUB-BOOTSTRAP-REPO>.git gcp-bootstrap
   ```

1. 레이아웃은 다음과 같아야 합니다:

   ```bash
   gcp-bootstrap/
   terraform-example-foundation/
   ```

1. 저장소로 이동합니다. 이후의 모든 단계는
   `gcp-bootstrap` 디렉토리에서 실행한다고 가정합니다.
   다른 디렉토리에서 실행하는 경우 복사 경로를 적절히 조정하십시오.

   ```bash
   cd gcp-bootstrap
   ```

1. 아직 초기화되지 않은 경우 저장소를 시드합니다.

   ```bash
   git commit --allow-empty -m 'repository seed'
   git push --set-upstream origin main

   git checkout -b production
   git push --set-upstream origin production
   ```

1. 비프로덕션 브랜치로 변경합니다.

   ```bash
   git checkout -b plan
   ```

1. foundation의 내용을 새 저장소로 복사합니다 (현재 디렉토리에 따라 적절히 수정).

   ```bash
   mkdir -p envs/shared

   cp -RT ../terraform-example-foundation/0-bootstrap/ ./envs/shared
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   mkdir -p .github/workflows
   cp ../terraform-example-foundation/build/github-tf-* ./.github/workflows/
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   cd ./envs/shared
   ```

1. 버전 파일 `./versions.tf`에서 `github` 필수 프로바이더의 주석을 해제합니다
1. 변수 파일 `./variables.tf`에서 `Specific to github_bootstrap` 섹션의 변수들의 주석을 해제합니다
1. 출력 파일 `./outputs.tf`에서 `Specific to cloudbuild_module` 섹션의 출력들을 주석 처리합니다
1. 출력 파일 `./outputs.tf`에서 `Specific to github_bootstrap` 섹션의 출력들의 주석을 해제합니다
1. `./cb.tf` 파일을 `./cb.tf.example`로 이름을 변경합니다

   ```bash
   mv ./cb.tf ./cb.tf.example
   ```

1. `./github.tf.example` 파일을 `./github.tf`로 이름을 변경합니다

   ```bash
   mv ./github.tf.example ./github.tf
   ```

1. `terraform.example.tfvars` 파일을 `terraform.tfvars`로 이름을 변경합니다

   ```bash
   mv ./terraform.example.tfvars ./terraform.tfvars
   ```

1. Google Cloud 환경의 값으로 `terraform.tfvars` 파일을 업데이트합니다
1. GitHub 저장소의 값으로 `terraform.tfvars` 파일을 업데이트합니다
1. `terraform.tfvars` 파일에 `gh_token`을 평문으로 저장하는 것을 방지하기 위해,
GitHub 세분화된 액세스 토큰을 환경 변수로 내보냅니다:

   ```bash
   export TF_VAR_gh_token="YOUR-FINE-GRAINED-ACCESS-TOKEN"
   ```

1. 헬퍼 스크립트 [validate-requirements.sh](../scripts/validate-requirements.sh)를 사용하여 환경을 검증합니다:

   ```bash
   ../../../terraform-example-foundation/scripts/validate-requirements.sh  -o <ORGANIZATION_ID> -b <BILLING_ACCOUNT_ID> -u <END_USER_EMAIL> -e
   ```

   **참고:** 스크립트는 사용자가 필요한 역할을 가진 Cloud Identity 또는 Google Workspace 그룹에 있는지 검증할 수 없습니다.

1. `terraform init`과 `terraform plan`을 실행하고 출력을 검토합니다.

   ```bash
   terraform init
   terraform plan -input=false -out bootstrap.tfplan
   ```

1. 정책을 검증하려면 `gcloud beta terraform vet`를 실행합니다. 설치 지침은 Google Cloud CLI용 [정책 검증](https://cloud.google.com/docs/terraform/policy-validation/validate-policies) 지침을 참조하십시오.

1. 다음 명령을 실행하고 위반 사항을 확인합니다:

   ```bash
   export VET_PROJECT_ID=A-VALID-PROJECT-ID

   terraform show -json bootstrap.tfplan > bootstrap.json
   gcloud beta terraform vet bootstrap.json --policy-library="../../policy-library" --project ${VET_PROJECT_ID}
   ```

   *`A-VALID-PROJECT-ID`*는 액세스 권한이 있는 기존 프로젝트여야 합니다. Terraform-validator가 리소스를 유효한 Google Cloud Platform 프로젝트에 연결해야 하므로 이는 필요합니다.

1. 위반 사항이 없고 `done`이라는 출력이 있으면 검증이 성공한 것입니다.

1. `terraform apply`를 실행합니다.

   ```bash
   terraform apply bootstrap.tfplan
   ```

1. `terraform output`을 실행하여 `3-networks-svpc`, `3-networks-hub-and-spoke`, `4-projects` 단계에서 `shared` 환경에 대한 수동 단계를 실행하는 데 사용될 terraform 서비스 계정의 이메일 주소를 가져옵니다.

   ```bash
   export network_step_sa=$(terraform output -raw networks_step_terraform_service_account_email)
   export projects_step_sa=$(terraform output -raw projects_step_terraform_service_account_email)

   echo "network step service account = ${network_step_sa}"
   echo "projects step service account = ${projects_step_sa}"
   ```

1. `terraform output`을 실행하여 CI/CD 프로젝트의 ID를 가져옵니다:

   ```bash
   export cicd_project_id=$(terraform output -raw cicd_project_id)
   echo "CI/CD Project ID = ${cicd_project_id}"
   ```

1. 백엔드를 복사하고 Terraform 상태를 위한 Google Cloud Storage 버킷 이름으로 `backend.tf`를 업데이트합니다. 또한 모든 단계의 `backend.tf`도 업데이트합니다.

   ```bash
   export backend_bucket=$(terraform output -raw gcs_bucket_tfstate)
   echo "backend_bucket = ${backend_bucket}"

   cp backend.tf.example backend.tf
   cd ../../../

   for i in `find . -name 'backend.tf'`; do sed -i'' -e "s/UPDATE_ME/${backend_bucket}/" $i; done
   for i in `find . -name 'backend.tf'`; do sed -i'' -e "s/UPDATE_PROJECTS_BACKEND/${backend_bucket}/" $i; done

   cd gcp-bootstrap/envs/shared
   ```

1. `terraform init`을 다시 실행합니다. 메시지가 표시되면 Terraform 상태를 Cloud Storage로 복사하는 것에 동의합니다.

   ```bash
   terraform init
   ```

1. (선택사항) `terraform plan`을 실행하여 상태가 올바르게 구성되었는지 확인합니다. 이전 상태에서 변경 사항이 없어야 합니다.
1. Terraform 구성을 `gcp-bootstrap` GitHub 저장소에 저장합니다:

   ```bash
   cd ../..
   git add .
   git commit -m 'Initialize bootstrap repo'
   git push --set-upstream origin plan
   ```

1. GitHub에서 https://github.com/GITHUB-OWNER/GITHUB-BOOTSTRAP-REPO/pull/new/plan에서 `plan` 브랜치에서 `production` 브랜치로 풀 리퀄스트를 열고 출력을 검토합니다.
1. 풀 리퀄스트는 `production` 환경에서 Terraform `init`/`plan`/`validate`를 실행하는 GitHub Action을 트리거합니다.
1. GitHub https://github.com/GITHUB-OWNER/GITHUB-BOOTSTRAP-REPO/actions의 `tf-pull-request`에서 GitHub Action 출력을 검토합니다.
1. GitHub 액션이 성공하면 풀 리퀄스트를 `production` 브랜치로 병합합니다.
1. 병합은 `production` 환경에 terraform 구성을 적용하는 GitHub Action을 트리거합니다.
1. GitHub https://github.com/GITHUB-OWNER/GITHUB-BOOTSTRAP-REPO/actions의 `tf-apply`에서 병합 출력을 검토합니다.

1. 다음 단계로 이동하기 전에 상위 디렉토리로 돌아갑니다.

   ```bash
   cd ..
   ```

**참고 1:** `0-bootstrap` 이후의 단계들은 `terraform_remote_state` 데이터 소스를 사용하여 `0-bootstrap` 단계의 출력에서 조직 ID와 같은 공통 구성을 읽습니다.
상태가 Cloud Storage 버킷에 복사되지 않으면 [실패](../docs/TROUBLESHOOTING.md#error-unsupported-attribute)합니다.

**참고 2:** 배포 후 [문제 해결 가이드](../docs/TROUBLESHOOTING.md#project-quota-exceeded)에 설명된 프로젝트 할당량 오류를 방지하기 위해,
이 단계에서 생성된 **프로젝트 단계 서비스 계정**에 대해 추가로 50개의 프로젝트를 요청하는 것을 권장합니다.

## 1-org 단계 배포

1. `terraform-example-foundation` 폴더와 같은 수준에서 `1-org` terraform 구성을 호스팅하기 위해 생성한 저장소를 복제합니다.

   ```bash
   git clone git@github.com:<GITHUB-OWNER>/<GITHUB-ORGANIZATION-REPO>.git gcp-org
   ```

1. 저장소로 이동합니다. 이후의 모든 단계는 `gcp-org` 디렉토리에서 실행한다고 가정합니다.
   다른 디렉토리에서 실행하는 경우 복사 경로를 적절히 조정하십시오.

   ```bash
   cd gcp-org
   ```

1. 아직 초기화되지 않은 경우 저장소를 시드합니다.

   ```bash
   git commit --allow-empty -m 'repository seed'
   git push --set-upstream origin main

   git checkout -b production
   git push --set-upstream origin production
   ```

1. 비프로덕션 브랜치로 변경합니다.

   ```bash
   git checkout -b plan
   ```

1. foundation의 내용을 새 저장소로 복사합니다.

   ```bash
   cp -RT ../terraform-example-foundation/1-org/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   mkdir -p .github/workflows
   cp ../terraform-example-foundation/build/github-tf-* ./.github/workflows/
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. `./envs/shared/terraform.example.tfvars`를 `./envs/shared/terraform.tfvars`로 이름을 변경합니다

   ```bash
   mv ./envs/shared/terraform.example.tfvars ./envs/shared/terraform.tfvars
   ```

1. GCP 환경의 값으로 `envs/shared/terraform.tfvars` 파일을 업데이트합니다.
`terraform.tfvars` 파일의 값에 대한 추가 정보는 shared 폴더 [README.md](../1-org/envs/shared/README.md#inputs)를 참조하십시오.

1. 조직에 이미 Access Context Manager 정책이 있는 경우 `create_access_context_manager_access_policy = false` 변수의 주석을 해제합니다.

    ```bash
    export ORGANIZATION_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -json common_config | jq '.org_id' --raw-output)

    export ACCESS_CONTEXT_MANAGER_ID=$(gcloud access-context-manager policies list --organization ${ORGANIZATION_ID} --format="value(name)")

    echo "access_context_manager_policy_id = ${ACCESS_CONTEXT_MANAGER_ID}"

    if [ ! -z "${ACCESS_CONTEXT_MANAGER_ID}" ]; then sed -i'' -e "s=//create_access_context_manager_access_policy=create_access_context_manager_access_policy=" ./envs/shared/terraform.tfvars; fi
    ```

1. Bootstrap 단계의 백엔드 버킷으로 `remote_state_bucket` 변수를 업데이트합니다.

   ```bash
   export backend_bucket=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw gcs_bucket_tfstate)

   echo "remote_state_bucket = ${backend_bucket}"

   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" ./envs/shared/terraform.tfvars
   ```

1. 조직에 기본 이름인 **scc-notify**를 가진 Security Command Center 알림이 이미 있는지 확인합니다.

   ```bash
   export ORG_STEP_SA=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw organization_step_terraform_service_account_email)

   gcloud scc notifications describe "scc-notify" --format="value(name)" --organization=${ORGANIZATION_ID} --impersonate-service-account=${ORG_STEP_SA}
   ```

1. 알림이 존재하면 출력은 다음과 같습니다:

    ```text
    organizations/ORGANIZATION_ID/notificationConfigs/scc-notify
    ```

1. 알림이 존재하지 않으면 출력은 다음과 같습니다:

    ```text
    ERROR: (gcloud.scc.notifications.describe) NOT_FOUND: Requested entity was not found.
    ```

1. 알림이 존재하면 `./envs/shared/terraform.tfvars` 파일의 `scc_notification_name` 변수에 대해 다른 값을 선택합니다.

1. 변경 사항을 커밋합니다.

   ```bash
   git add .
   git commit -m 'Initialize org repo'
   ```

1. plan 브랜치를 푸시합니다.

   ```bash
   git push --set-upstream origin plan
   ```

1. GitHub에서 https://github.com/GITHUB-OWNER/GITHUB-ORGANIZATION-REPO/pull/new/plan에서 `plan` 브랜치에서 `production` 브랜치로 풀 리퀘스트를 열고 출력을 검토합니다.
1. 풀 리퀘스트는 `production` 환경에서 Terraform `init`/`plan`/`validate`를 실행하는 GitHub Action을 트리거합니다.
1. GitHub https://github.com/GITHUB-OWNER/GITHUB-ORGANIZATION-REPO/actions의 `tf-pull-request`에서 GitHub Action 출력을 검토합니다.
1. GitHub 액션이 성공하면 풀 리퀘스트를 `production` 브랜치로 병합합니다.
1. 병합은 `production` 환경에 terraform 구성을 적용하는 GitHub Action을 트리거합니다.
1. GitHub https://github.com/GITHUB-OWNER/GITHUB-ORGANIZATION-REPO/actions의 `tf-apply`에서 병합 출력을 검토합니다.

1. 다음 단계로 이동하기 전에 상위 디렉토리로 돌아갑니다.

   ```bash
   cd ..
   ```


## 2-environments 단계 배포

1. `terraform-example-foundation` 폴더와 같은 수준에서 `2-environments` terraform 구성을 호스팅하기 위해 생성한 저장소를 복제합니다.

   ```bash
   git clone git@github.com:<GITHUB-OWNER>/<GITHUB-ENVIRONMENTS-REPO>.git gcp-environments
   ```

1. 저장소로 이동합니다. 이후의 모든
   단계는 `gcp-environments` 디렉토리에서 실행한다고 가정합니다.
   다른 디렉토리에서 실행하는 경우 복사 경로를 적절히 조정하십시오.

   ```bash
   cd gcp-environments
   ```

1. 아직 초기화되지 않은 경우 저장소를 시드합니다.

   ```bash
   git commit --allow-empty -m 'repository seed'
   git push --set-upstream origin main

   git checkout -b production
   git push --set-upstream origin production

   git checkout -b nonproduction
   git push --set-upstream origin nonproduction

   git checkout -b development
   git push --set-upstream origin development
   ```

1. 비프로덕션 브랜치로 변경합니다.

   ```bash
   git checkout -b plan
   ```

1. foundation의 내용을 새 저장소로 복사합니다.

   ```bash
   cp -RT ../terraform-example-foundation/2-environments/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   mkdir -p .github/workflows
   cp ../terraform-example-foundation/build/github-tf-* ./.github/workflows/
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. `terraform.example.tfvars`를 `terraform.tfvars`로 이름을 변경합니다.

   ```bash
   mv terraform.example.tfvars terraform.tfvars
   ```

1. GCP 환경의 값으로 파일을 업데이트합니다.
`terraform.tfvars` 파일의 값에 대한 추가 정보는 envs 폴더의 [README.md](../2-environments/envs/production/README.md#inputs) 파일을 참조하십시오.

1. Bootstrap 단계의 백엔드 버킷으로 `remote_state_bucket` 변수를 업데이트합니다.

   ```bash
   export backend_bucket=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${backend_bucket}"

   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" terraform.tfvars
   ```

1. 변경 사항을 커밋합니다.

   ```bash
   git add .
   git commit -m 'Initialize environments repo'
   ```

1. plan 브랜치를 푸시합니다.

   ```bash
   git push --set-upstream origin plan
   ```

1. GitHub에서 https://github.com/GITHUB-OWNER/GITHUB-ENVIRONMENTS-REPO/pull/new/plan에서 `plan` 브랜치에서 `development` 브랜치로 풀 리퀘스트를 열고 출력을 검토합니다.
1. 풀 리퀘스트는 `development` 환경에서 Terraform `init`/`plan`/`validate`를 실행하는 GitHub Action을 트리거합니다.
1. GitHub https://github.com/GITHUB-OWNER/GITHUB-ENVIRONMENTS-REPO/actions의 `tf-pull-request`에서 GitHub Action 출력을 검토합니다.
1. GitHub 액션이 성공하면 풀 리퀘스트를 `development` 브랜치로 병합합니다.
1. 병합은 `development` 환경에 terraform 구성을 적용하는 GitHub Action을 트리거합니다.
1. GitHub https://github.com/GITHUB-OWNER/GITHUB-ENVIRONMENTS-REPO/actions의 `tf-apply`에서 병합 출력을 검토합니다.
1. GitHub 액션이 성공하면 다음 환경을 적용합니다.

1. Open a pull request in GitHub https://github.com/GITHUB-OWNER/GITHUB-ENVIRONMENTS-REPO/pull/new/development from the `development` branch to the `nonproduction` branch and review the output.
1. The Pull request will trigger a GitHub Action that will run Terraform `init`/`plan`/`validate` in the `nonproduction` environment.
1. GitHub https://github.com/GITHUB-OWNER/GITHUB-ENVIRONMENTS-REPO/actions의 `tf-pull-request`에서 GitHub Action 출력을 검토합니다.
1. If the GitHub action is successful, merge the pull request in to the `nonproduction` branch.
1. The merge will trigger a GitHub Action that will apply the terraform configuration for the `nonproduction` environment.
1. GitHub https://github.com/GITHUB-OWNER/GITHUB-ENVIRONMENTS-REPO/actions의 `tf-apply`에서 병합 출력을 검토합니다.
1. GitHub 액션이 성공하면 다음 환경을 적용합니다.

1. Open a pull request in GitHub https://github.com/GITHUB-OWNER/GITHUB-ENVIRONMENTS-REPO/pull/new/nonproduction from the `nonproduction` branch to the `production` branch and review the output.
1. The Pull request will trigger a GitHub Action that will run Terraform `init`/`plan`/`validate` in the `production` environment.
1. GitHub https://github.com/GITHUB-OWNER/GITHUB-ENVIRONMENTS-REPO/actions의 `tf-pull-request`에서 GitHub Action 출력을 검토합니다.
1. If the GitHub action is successful, merge the pull request in to the `production` branch.
1. The merge will trigger a GitHub Action that will apply the terraform configuration for the `production` environment.
1. GitHub https://github.com/GITHUB-OWNER/GITHUB-ENVIRONMENTS-REPO/actions의 `tf-apply`에서 병합 출력을 검토합니다.

1. 이제 네트워크 단계의 지침으로 이동할 수 있습니다.
[Shared VPC](https://cloud.google.com/architecture/security-foundations/networking#vpcsharedvpc-id7-1-shared-vpc-) 네트워크 모드를 사용하려면 [3-networks-svpc 단계 배포](#deploying-step-3-networks-svpc)로 이동하고,
[Hub and Spoke](https://cloud.google.com/architecture/security-foundations/networking#hub-and-spoke) 네트워크 모드를 사용하려면 [3-networks-hub-and-spoke 단계 배포](#deploying-step-3-networks-hub-and-spoke)로 이동하십시오.

1. 다음 단계로 이동하기 전에 상위 디렉토리로 돌아갑니다.

   ```bash
   cd ..
   ```

## Deploying step 3-networks-svpc

1. Clone the repository you created to host the `3-networks-svpc` terraform configuration at the same level of the `terraform-example-foundation` folder.

   ```bash
   git clone git@github.com:<GITHUB-OWNER>/<GITHUB-NETWORKS-REPO>.git gcp-networks
   ```

1. Navigate into the repo. All subsequent steps assume you are running them from the `gcp-networks` directory.
   If you run them from another directory, adjust your copy paths accordingly.

   ```bash
   cd gcp-networks
   ```

1. 아직 초기화되지 않은 경우 저장소를 시드합니다.

   ```bash
   git commit --allow-empty -m 'repository seed'
   git push --set-upstream origin main

   git checkout -b production
   git push --set-upstream origin production

   git checkout -b nonproduction
   git push --set-upstream origin nonproduction

   git checkout -b development
   git push --set-upstream origin development
   ```

1. 비프로덕션 브랜치로 변경합니다.

   ```bash
   git checkout -b plan
   ```

1. foundation의 내용을 새 저장소로 복사합니다.

   ```bash
   cp -RT ../terraform-example-foundation/3-networks-svpc/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   mkdir -p .github/workflows
   cp ../terraform-example-foundation/build/github-tf-* ./.github/workflows/
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. `common.auto.example.tfvars`를 `common.auto.tfvars`로, `production.auto.example.tfvars`를 `production.auto.tfvars`로, `access_context.auto.example.tfvars`를 `access_context.auto.tfvars`로 이름을 변경합니다.

   ```bash
   mv common.auto.example.tfvars common.auto.tfvars
   mv production.auto.example.tfvars production.auto.tfvars
   mv access_context.auto.example.tfvars access_context.auto.tfvars
   ```

1. `target_name_server_addresses`의 값으로 `production.auto.tfvars` 파일을 업데이트합니다.
1. 조직의 `access_context_manager_policy_id`로 `access_context.auto.tfvars` 파일을 업데이트합니다.

   ```bash
   export ORGANIZATION_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -json common_config | jq '.org_id' --raw-output)

   export ACCESS_CONTEXT_MANAGER_ID=$(gcloud access-context-manager policies list --organization ${ORGANIZATION_ID} --format="value(name)")

   echo "access_context_manager_policy_id = ${ACCESS_CONTEXT_MANAGER_ID}"

   sed -i'' -e "s/ACCESS_CONTEXT_MANAGER_ID/${ACCESS_CONTEXT_MANAGER_ID}/" ./access_context.auto.tfvars
   ```

1. GCP 환경의 값으로 `common.auto.tfvars` 파일을 업데이트합니다.
`common.auto.tfvars` 파일의 값에 대한 추가 정보는 envs 폴더의 [README.md](../3-networks-svpc/envs/production/README.md#inputs) 파일을 참조하십시오.
1. 프로젝트에서 생성된 리소스를 보려면 `perimeter_additional_members` 변수에 사용자 이메일을 추가해야 합니다.
1. `common.auto.tfvars` 파일에서 Bootstrap 단계의 백엔드 버킷으로 `remote_state_bucket` 변수를 업데이트합니다.

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

1. `development`, `nonproduction`, `production` 환경이 종속되어 있으므로 `shared` 환경을 수동으로 계획하고 적용해야 합니다(한 번만).
1. `terraform output`을 사용하여 gcp-bootstrap 출력에서 CI/CD 프로젝트 ID와 네트워크 단계 Terraform 서비스 계정을 가져옵니다.
1. CI/CD 프로젝트 ID는 Terraform 구성의 [검증](https://cloud.google.com/docs/terraform/policy-validation/quickstart)에 사용됩니다

   ```bash
   export CICD_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -raw cicd_project_id)
   echo ${CICD_PROJECT_ID}
   ```

1. 네트워크 단계 Terraform 서비스 계정은 다음 단계에서 [서비스 계정 가장](https://cloud.google.com/docs/authentication/use-service-account-impersonation)에 사용됩니다.
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

1. `tf-wrapper.sh` 스크립트의 `validate` 옵션을 사용하려면 terraform-tools 구성 요소를 설치하기 위해 [지침](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install)을 따르십시오.
1. `validate`를 실행하고 위반 사항을 확인합니다.

   ```bash
   ./tf-wrapper.sh validate shared $(pwd)/policy-library ${CICD_PROJECT_ID}
   ```

1. shared를 `apply`합니다.

   ```bash
   ./tf-wrapper.sh apply shared
   ```

1. `development`, `nonproduction`, `plan` 환경이 종속되어 있으므로 `production` 환경을 수동으로 계획하고 적용해야 합니다.

   ```bash
   git checkout production
   git merge plan
   ```

1. production 환경에 대해 `init`과 `plan`을 실행하고 출력을 검토합니다.

   ```bash
   ./tf-wrapper.sh init production
   ./tf-wrapper.sh plan production
   ```

1. production을 `apply`합니다.

   ```bash
   ./tf-wrapper.sh apply production
   ```

1. development와 nonproduction이 이에 종속되어 있으므로 production 브랜치를 푸시합니다.

*참고:** 다른 환경에서 사용할 DNS 허브 통신이 포함되어 있으므로 Production 환경은 처음 푸시되는 브랜치여야 합니다.

   ```bash
   git add .
   git commit -m 'Initialize networks repo - production'
   git push --set-upstream origin production
   ```

1. GitLab에서 https://gitlab.com/GITLAB-OWNER/GITLAB-NETWORKS-REPO/-/merge_requests?scope=all&state=opened에서 `production` 브랜치에서 `plan` 브랜치로 병합 요청을 열고 출력을 검토합니다.

> 참고: `3-networks-dual-svpc`의 경우 개발 및 비프로덕션 브랜치는 production 브랜치가 먼저 배포되어야 합니다.

1. plan 브랜치를 푸시합니다.

   ```bash
   git checkout plan --set-upstream origin plan
   git push
   ```

1. Open a pull request in GitHub https://github.com/GITHUB-OWNER/GITHUB-NETWORKS-REPO/pull/new/plan from the `production` branch to the `development` branch and review the output.
1. 풀 리퀘스트는 `development` 환경에서 Terraform `init`/`plan`/`validate`를 실행하는 GitHub Action을 트리거합니다.
1. Review the GitHub Action output in GitHub https://github.com/GITHUB-OWNER/GITHUB-NETWORKS-REPO/actions under `tf-pull-request`.
1. GitHub 액션이 성공하면 풀 리퀘스트를 `development` 브랜치로 병합합니다.
1. 병합은 `development` 환경에 terraform 구성을 적용하는 GitHub Action을 트리거합니다.
1. Review merge output in GitHub https://github.com/GITHUB-OWNER/GITHUB-NETWORKS-REPO/actions under `tf-apply`.
1. GitHub 액션이 성공하면 다음 환경을 적용합니다.

1. Open a pull request in GitHub https://github.com/GITHUB-OWNER/GITHUB-NETWORKS-REPO/pull/new/development from the `development` branch to the `nonproduction` branch and review the output.
1. The Pull request will trigger a GitHub Action that will run Terraform `init`/`plan`/`validate` in the `nonproduction` environment.
1. Review the GitHub Action output in GitHub https://github.com/GITHUB-OWNER/GITHUB-NETWORKS-REPO/actions under `tf-pull-request`.
1. If the GitHub action is successful, merge the pull request in to the `nonproduction` branch.
1. The merge will trigger a GitHub Action that will apply the terraform configuration for the `nonproduction` environment.
1. Review merge output in GitHub https://github.com/GITHUB-OWNER/GITHUB-NETWORKS-REPO/actions under `tf-apply`.
1. GitHub 액션이 성공하면 다음 환경을 적용합니다.

1. 다음 단계를 실행하기 전에 `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` 환경 변수를 설정 해제합니다.

   ```bash
   unset GOOGLE_IMPERSONATE_SERVICE_ACCOUNT
   ```

1. 다음 단계로 이동하기 전에 상위 디렉토리로 돌아갑니다.

   ```bash
   cd ..
   ```

1. 이제 [4-projects](#deploying-step-4-projects) 단계의 지침으로 이동할 수 있습니다.

## Deploying step 3-networks-hub-and-spoke

1. Clone the repository you created to host the `3-networks-hub-and-spoke` terraform configuration at the same level of the `terraform-example-foundation` folder.

   ```bash
   git clone git@github.com:<GITHUB-OWNER>/<GITHUB-NETWORKS-REPO>.git gcp-networks
   ```

1. Navigate into the repo. All subsequent steps assume you are running them from the `gcp-networks` directory.
   If you run them from another directory, adjust your copy paths accordingly.

   ```bash
   cd gcp-networks
   ```

1. 아직 초기화되지 않은 경우 저장소를 시드합니다.

   ```bash
   git commit --allow-empty -m 'repository seed'
   git push --set-upstream origin main

   git checkout -b production
   git push --set-upstream origin production

   git checkout -b nonproduction
   git push --set-upstream origin nonproduction

   git checkout -b development
   git push --set-upstream origin development
   ```

1. 비프로덕션 브랜치로 변경합니다.

   ```bash
   git checkout -b plan
   ```

1. foundation의 내용을 새 저장소로 복사합니다.

   ```bash
   cp -RT ../terraform-example-foundation/3-networks-hub-and-spoke/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   mkdir -p .github/workflows
   cp ../terraform-example-foundation/build/github-tf-* ./.github/workflows/
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
`common.auto.tfvars` 파일의 값에 대한 추가 정보는 envs 폴더의 [README.md](../3-networks-hub-and-spoke/envs/production/README.md#inputs) 파일을 참조하십시오.
1. 프로젝트에서 생성된 리소스를 보려면 `perimeter_additional_members` 변수에 사용자 이메일을 추가해야 합니다.
1. `common.auto.tfvars` 파일에서 Bootstrap 단계의 백엔드 버킷으로 `remote_state_bucket` 변수를 업데이트합니다.

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

1. `development`, `nonproduction`, `production` 환경이 종속되어 있으므로 `shared` 환경을 수동으로 계획하고 적용해야 합니다(한 번만).
1. `terraform output`을 사용하여 gcp-bootstrap 출력에서 CI/CD 프로젝트 ID와 네트워크 단계 Terraform 서비스 계정을 가져옵니다.
1. CI/CD 프로젝트 ID는 Terraform 구성의 [검증](https://cloud.google.com/docs/terraform/policy-validation/quickstart)에 사용됩니다

   ```bash
   export CICD_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -raw cicd_project_id)
   echo ${CICD_PROJECT_ID}
   ```

1. 네트워크 단계 Terraform 서비스 계정은 다음 단계에서 [서비스 계정 가장](https://cloud.google.com/docs/authentication/use-service-account-impersonation)에 사용됩니다.
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

1. `tf-wrapper.sh` 스크립트의 `validate` 옵션을 사용하려면 terraform-tools 구성 요소를 설치하기 위해 [지침](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install)을 따르십시오.
1. `validate`를 실행하고 위반 사항을 확인합니다.

   ```bash
   ./tf-wrapper.sh validate shared $(pwd)/policy-library ${CICD_PROJECT_ID}
   ```

1. shared를 `apply`합니다.

   ```bash
   ./tf-wrapper.sh apply shared
   ```

1. plan 브랜치를 푸시합니다.

   ```bash
   git push --set-upstream origin plan
   ```

1. Open a pull request in GitHub https://github.com/GITHUB-OWNER/GITHUB-NETWORKS-REPO/pull/new/plan from the `plan` branch to the `development` branch and review the output.
1. 풀 리퀘스트는 `development` 환경에서 Terraform `init`/`plan`/`validate`를 실행하는 GitHub Action을 트리거합니다.
1. Review the GitHub Action output in GitHub https://github.com/GITHUB-OWNER/GITHUB-NETWORKS-REPO/actions under `tf-pull-request`.
1. GitHub 액션이 성공하면 풀 리퀘스트를 `development` 브랜치로 병합합니다.
1. 병합은 `development` 환경에 terraform 구성을 적용하는 GitHub Action을 트리거합니다.
1. Review merge output in GitHub https://github.com/GITHUB-OWNER/GITHUB-NETWORKS-REPO/actions under `tf-apply`.
1. GitHub 액션이 성공하면 다음 환경을 적용합니다.

1. Open a pull request in GitHub https://github.com/GITHUB-OWNER/GITHUB-NETWORKS-REPO/pull/new/development from the `development` branch to the `nonproduction` branch and review the output.
1. The Pull request will trigger a GitHub Action that will run Terraform `init`/`plan`/`validate` in the `nonproduction` environment.
1. Review the GitHub Action output in GitHub https://github.com/GITHUB-OWNER/GITHUB-NETWORKS-REPO/actions under `tf-pull-request`.
1. If the GitHub action is successful, merge the pull request in to the `nonproduction` branch.
1. The merge will trigger a GitHub Action that will apply the terraform configuration for the `nonproduction` environment.
1. Review merge output in GitHub https://github.com/GITHUB-OWNER/GITHUB-NETWORKS-REPO/actions under `tf-apply`.
1. GitHub 액션이 성공하면 다음 환경을 적용합니다.

1. Open a pull request in GitHub https://github.com/GITHUB-OWNER/GITHUB-NETWORKS-REPO/pull/new/nonproduction from the `nonproduction` branch to the `production` branch and review the output.
1. The Pull request will trigger a GitHub Action that will run Terraform `init`/`plan`/`validate` in the `production` environment.
1. Review the GitHub Action output in GitHub https://github.com/GITHUB-OWNER/GITHUB-NETWORKS-REPO/actions under `tf-pull-request`.
1. If the GitHub action is successful, merge the pull request in to the `production` branch.
1. The merge will trigger a GitHub Action that will apply the terraform configuration for the `production` environment.
1. Review merge output in GitHub https://github.com/GITHUB-OWNER/GITHUB-NETWORKS-REPO/actions under `tf-apply`.


1. 다음 단계를 실행하기 전에 `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` 환경 변수를 설정 해제합니다.

   ```bash
   unset GOOGLE_IMPERSONATE_SERVICE_ACCOUNT
   ```

1. 다음 단계로 이동하기 전에 상위 디렉토리로 돌아갑니다.

   ```bash
   cd ..
   ```

1. 이제 [4-projects](#deploying-step-4-projects) 단계의 지침으로 이동할 수 있습니다.

## Deploying step 4-projects

1. Clone the repository you created to host the `4-projects` terraform configuration at the same level of the `terraform-example-foundation` folder.

   ```bash
   git clone git@github.com:<GITHUB-OWNER>/<GITHUB-PROJECTS-REPO>.git gcp-projects
   ```

1. Navigate into the repo. All subsequent
   steps assume you are running them from the `gcp-projects` directory.
   If you run them from another directory, adjust your copy paths accordingly.

   ```bash
   cd gcp-projects
   ```



1. 아직 초기화되지 않은 경우 저장소를 시드합니다.

   ```bash
   git commit --allow-empty -m 'repository seed'
   git push --set-upstream origin main

   git checkout -b production
   git push --set-upstream origin production

   git checkout -b nonproduction
   git push --set-upstream origin nonproduction

   git checkout -b development
   git push --set-upstream origin development
   ```

1. 비프로덕션 브랜치로 변경합니다.

   ```bash
   git checkout -b plan
   ```

1. foundation의 내용을 새 저장소로 복사합니다.

   ```bash
   cp -RT ../terraform-example-foundation/4-projects/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   mkdir -p .github/workflows
   cp ../terraform-example-foundation/build/github-tf-* ./.github/workflows/
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. `auto.example.tfvars` 파일들을 `auto.tfvars`로 이름을 변경합니다.

   ```bash
   mv common.auto.example.tfvars common.auto.tfvars
   mv shared.auto.example.tfvars shared.auto.tfvars
   mv development.auto.example.tfvars development.auto.tfvars
   mv nonproduction.auto.example.tfvars nonproduction.auto.tfvars
   mv production.auto.example.tfvars production.auto.tfvars
   ```

1. `common.auto.tfvars`, `development.auto.tfvars`, `nonproduction.auto.tfvars`, `production.auto.tfvars` 파일의 값에 대한 추가 정보는 envs 폴더의 [README.md](../4-projects/business_unit_1/production/README.md#inputs) 파일을 참조하십시오.
1. `shared.auto.tfvars` 파일의 값에 대한 추가 정보는 shared 폴더의 [README.md](../4-projects/business_unit_1/shared/README.md#inputs) 파일을 참조하십시오.

1. `terraform output`을 사용하여 bootstrap 출력에서 백엔드 버킷 값을 가져옵니다.

   ```bash
   export remote_state_bucket=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${remote_state_bucket}"

   sed -i'' -e "s/REMOTE_STATE_BUCKET/${remote_state_bucket}/" ./common.auto.tfvars
   ```

1. (선택 사항) 별도의 비즈니스 단위 또는 엔티티에 대한 추가 하위 폴더를 원하는 경우 `business_unit_1` 폴더의 추가 복사본을 만들고 `business_code`, `business_unit`, `subnet_ip_range`와 같이 비즈니스 단위마다 다른 값들을 수정하십시오.

예를 들어 business_unit_1과 유사한 새 비즈니스 단위를 생성하려면 다음을 실행하십시오:

   ```bash
   #copy the business_unit_1 folder and it's contents to a new folder business_unit_2
   cp -r  business_unit_1 business_unit_2

   # search all files under the folder `business_unit_2` and replace strings for business_unit_1 with strings for business_unit_2
   grep -rl bu1 business_unit_2/ | xargs sed -i 's/bu1/bu2/g'
   grep -rl business_unit_1 business_unit_2/ | xargs sed -i 's/business_unit_1/business_unit_2/g'
   ```


1. 변경 사항을 커밋합니다.

   ```bash
   git add .
   git commit -m 'Initialize projects repo'
   ```

1. `development`, `nonproduction`, `production`이 종속되어 있으므로 `business_unit_1/shared` 및 `business_unit_2/shared` 환경을 한 번만 수동으로 계획하고 적용해야 합니다.

1. Use `terraform output` to get the CI/CD project ID and the projects step Terraform Service Account from gcp-bootstrap output.
1. CI/CD 프로젝트 ID는 Terraform 구성의 [검증](https://cloud.google.com/docs/terraform/policy-validation/quickstart)에 사용됩니다

   ```bash
   export CICD_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared/" output -raw cicd_project_id)
   echo ${CICD_PROJECT_ID}
   ```

1. 프로젝트 단계 Terraform 서비스 계정은 다음 단계에서 [서비스 계정 가장](https://cloud.google.com/docs/authentication/use-service-account-impersonation)에 사용됩니다.
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

1. `tf-wrapper.sh` 스크립트의 `validate` 옵션을 사용하려면 terraform-tools 구성 요소를 설치하기 위해 [지침](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install)을 따르십시오.
1. `validate`를 실행하고 위반 사항을 확인합니다.

   ```bash
   ./tf-wrapper.sh validate shared $(pwd)/policy-library ${CICD_PROJECT_ID}
   ```

1. shared를 `apply`합니다.

   ```bash
   ./tf-wrapper.sh apply shared
   ```

1. plan 브랜치를 푸시합니다.

   ```bash
   git push --set-upstream origin plan
   ```

1. Open a pull request in GitHub https://github.com/GITHUB-OWNER/GITHUB-PROJECTS-REPO/pull/new/plan from the `plan` branch to the `development` branch and review the output.
1. 풀 리퀘스트는 `development` 환경에서 Terraform `init`/`plan`/`validate`를 실행하는 GitHub Action을 트리거합니다.
1. Review the GitHub Action output in GitHub https://github.com/GITHUB-OWNER/GITHUB-PROJECTS-REPO/actions under `tf-pull-request`.
1. GitHub 액션이 성공하면 풀 리퀘스트를 `development` 브랜치로 병합합니다.
1. 병합은 `development` 환경에 terraform 구성을 적용하는 GitHub Action을 트리거합니다.
1. Review merge output in GitHub https://github.com/GITHUB-OWNER/GITHUB-PROJECTS-REPO/actions under `tf-apply`.
1. GitHub 액션이 성공하면 다음 환경을 적용합니다.

1. Open a pull request in GitHub https://github.com/GITHUB-OWNER/GITHUB-PROJECTS-REPO/pull/new/development from the `development` branch to the `nonproduction` branch and review the output.
1. The Pull request will trigger a GitHub Action that will run Terraform `init`/`plan`/`validate` in the `nonproduction` environment.
1. Review the GitHub Action output in GitHub https://github.com/GITHUB-OWNER/GITHUB-PROJECTS-REPO/actions under `tf-pull-request`.
1. If the GitHub action is successful, merge the pull request in to the `nonproduction` branch.
1. The merge will trigger a GitHub Action that will apply the terraform configuration for the `nonproduction` environment.
1. Review merge output in GitHub https://github.com/GITHUB-OWNER/GITHUB-PROJECTS-REPO/actions under `tf-apply`.
1. GitHub 액션이 성공하면 다음 환경을 적용합니다.

1. Open a pull request in GitHub https://github.com/GITHUB-OWNER/GITHUB-PROJECTS-REPO/pull/new/nonproduction from the `nonproduction` branch to the `production` branch and review the output.
1. The Pull request will trigger a GitHub Action that will run Terraform `init`/`plan`/`validate` in the `production` environment.
1. Review the GitHub Action output in GitHub https://github.com/GITHUB-OWNER/GITHUB-PROJECTS-REPO/actions under `tf-pull-request`.
1. If the GitHub action is successful, merge the pull request in to the `production` branch.
1. The merge will trigger a GitHub Action that will apply the terraform configuration for the `production` environment.
1. Review merge output in GitHub https://github.com/GITHUB-OWNER/GITHUB-PROJECTS-REPO/actions under `tf-apply`.

1. `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` 환경 변수를 설정 해제합니다.

   ```bash
   unset GOOGLE_IMPERSONATE_SERVICE_ACCOUNT
   ```
