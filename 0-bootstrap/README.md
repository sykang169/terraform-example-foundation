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

1. 현재 부트스트랩 프로젝트에서 다음 추가 서비스를 활성화하세요:

     ```bash
     gcloud services enable cloudresourcemanager.googleapis.com
     gcloud services enable cloudbilling.googleapis.com
     gcloud services enable iam.googleapis.com
     gcloud services enable cloudkms.googleapis.com
     gcloud services enable servicenetworking.googleapis.com
     ```

### 선택사항 - Google Cloud Identity 그룹 자동 생성

기반에서 Google Cloud Identity 그룹은 [인증 및 액세스 관리](https://cloud.google.com/architecture/security-foundations/authentication-authorization)에 사용됩니다.

[그룹](https://cloud.google.com/architecture/security-foundations/authentication-authorization#groups_for_access_control)의 자동 생성을 활성화하려면 다음 작업을 완료하세요:

- Cloud Identity API 청구를 위한 기존 프로젝트가 있어야 합니다.
- 청구 프로젝트에서 Cloud Identity API (`cloudidentity.googleapis.com`)를 활성화합니다.
- 청구 프로젝트에서 Terraform을 실행하는 사용자에게 `roles/serviceusage.serviceUsageConsumer` 역할을 부여합니다.
- 필수 그룹을 생성하려면 `groups.create_required_groups` 필드를 **true**로 변경합니다.
- `groups.create_optional_groups` 필드를 **true**로 변경하고 생성할 이메일로 `groups.optional_groups`를 채웁니다.

### 선택사항 - 온프레미스에 대한 Cloud Build 액세스

온프레미스 환경에 대한 Cloud Build 액세스를 구성하는 방법은 [onprem](./onprem.md)을 참조하세요.

### 문제 해결

이 단계에서 문제가 발생하면 [문제 해결](../docs/TROUBLESHOOTING.md)을 참조하세요.

## Jenkins로 배포하기

*경고: Jenkins로 배포하는 지침은 더 이상 적극적으로 테스트하거나 유지 관리하지 않습니다. Jenkins를 선호하는 사용자를 위해 지침을 남겨두었지만, 품질에 대한 보장은 하지 않으며 문제 해결과 지침 수정은 사용자의 책임입니다.*

`jenkins_bootstrap` 하위 모듈을 사용하는 경우, 0-bootstrap 단계를 실행하는 방법에 대한 요구사항과 지침은 [README-Jenkins](./README-Jenkins.md)를 참조하세요.
Jenkins를 사용하려면 현재 Jenkins 관리자(컨트롤러) 환경과의 연결 구성을 포함한 몇 가지 수동 단계가 필요합니다.

## GitHub Actions로 배포하기

[GitHub Actions](https://docs.github.com/en/actions)를 사용하여 배포하는 경우, 0-bootstrap 단계를 실행하는 방법에 대한 요구사항과 지침은 [README-GitHub.md](./README-GitHub.md)를 참조하세요.
GitHub Actions를 사용하려면 각 단계에서 사용되는 GitHub 리포지토리를 수동으로 생성해야 합니다.

## GitLab Pipelines로 배포하기

[GitLab Pipelines](https://docs.gitlab.com/ee/ci/pipelines/)를 사용하여 배포하는 경우, 0-bootstrap 단계를 실행하는 방법에 대한 요구사항과 지침은 [README-GitLab.md](./README-GitLab.md)를 참조하세요.
GitLab Pipeline을 사용하려면 각 단계에서 사용되는 GitLab 프로젝트(리포지토리)를 수동으로 생성해야 합니다.

## Terraform Cloud로 배포하기

[Terraform Cloud](https://developer.hashicorp.com/terraform/cloud-docs)를 사용하여 배포하는 경우, 0-bootstrap 단계를 실행하는 방법에 대한 요구사항과 지침은 [README-Terraform-Cloud.md](./README-Terraform-Cloud.md)를 참조하세요.
Terraform Cloud를 사용하려면 각 단계에서 사용되는 GitHub 리포지토리 또는 GitLab 프로젝트를 수동으로 생성해야 합니다.

## Cloud Build로 배포하기

*경고: 이 방법은 [신규 고객에게 더 이상 제공되지 않는](https://cloud.google.com/source-repositories/docs) Cloud Source Repositories에 대한 종속성이 있습니다. 조직에서 이전에 CSR API를 사용한 적이 있다면 이 방법을 사용할 수 있지만, 새로 생성된 조직은 CSR을 활성화할 수 없으며 이 배포 방법을 사용할 수 없습니다. 이 경우 로컬, Github, Gitlab 또는 Terraform Cloud로 배포하는 지침을 따르는 것을 권장합니다.*

다음 단계는 Cloud Build로 배포하는 단계를 소개합니다. 또는 모든 단계를 자동화하려면 [도우미 스크립트](../helpers/foundation-deployer/README.md)를 사용하세요. 각 단계에서 많은 사용자 정의 없이 데모 또는 테스트 목적으로 전체 조직을 빠르게 생성하고 삭제하려는 경우 도우미 스크립트를 사용하세요.

1. [terraform-example-foundation](https://github.com/terraform-google-modules/terraform-example-foundation)을 로컬 환경에 복제하고 `0-bootstrap` 폴더로 이동합니다.

   ```bash
   git clone https://github.com/terraform-google-modules/terraform-example-foundation.git

   cd terraform-example-foundation/0-bootstrap
   ```

1. `terraform.example.tfvars`를 `terraform.tfvars`로 이름을 변경하고 환경의 값으로 파일을 업데이트합니다:

   ```bash
   mv terraform.example.tfvars terraform.tfvars
   ```

1. 도우미 스크립트 [validate-requirements.sh](../scripts/validate-requirements.sh)를 사용하여 환경을 검증합니다:

   ```bash
   ../scripts/validate-requirements.sh -o <ORGANIZATION_ID> -b <BILLING_ACCOUNT_ID> -u <END_USER_EMAIL>
   ```

   **참고:** 스크립트는 사용자가 필요한 역할이 있는 Cloud Identity 또는 Google Workspace 그룹에 있는지 검증할 수 없습니다.

1. `terraform init`과 `terraform plan`을 실행하고 출력을 검토합니다.

   ```bash
   terraform init
   terraform plan -input=false -out bootstrap.tfplan
   ```

1. 정책을 검증하려면 `gcloud beta terraform vet`을 실행합니다. 설치 지침은 [Google Cloud CLI 설치](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install)를 참조하세요.

1. 다음 명령을 실행하고 위반 사항이 있는지 확인합니다:

   ```bash
   export VET_PROJECT_ID=A-VALID-PROJECT-ID
   terraform show -json bootstrap.tfplan > bootstrap.json
   gcloud beta terraform vet bootstrap.json --policy-library="../policy-library" --project ${VET_PROJECT_ID}
   ```

   *`A-VALID-PROJECT-ID`*는 액세스 권한이 있는 기존 프로젝트여야 합니다. `gcloud beta terraform vet`이 리소스를 유효한 Google Cloud Platform 프로젝트에 연결해야 하기 때문에 필요합니다.

1. `terraform apply`를 실행합니다.

   ```bash
   terraform apply bootstrap.tfplan
   ```

1. `terraform output`을 실행하여 `3-networks-svpc`, `3-networks-hub-and-spoke`, `4-projects` 단계에서 `shared` 환경을 위한 수동 단계를 실행하는 데 사용될 terraform 서비스 계정의 이메일 주소와 4-projects 단계에서 사용될 상태 버킷을 확인합니다.

   ```bash
   export network_step_sa=$(terraform output -raw networks_step_terraform_service_account_email)
   export projects_step_sa=$(terraform output -raw projects_step_terraform_service_account_email)
   export projects_gcs_bucket_tfstate=$(terraform output -raw projects_gcs_bucket_tfstate)

   echo "network step service account = ${network_step_sa}"
   echo "projects step service account = ${projects_step_sa}"
   echo "projects gcs bucket tfstate = ${projects_gcs_bucket_tfstate}"
   ```

1. `terraform output`을 실행하여 Cloud Build 프로젝트의 ID를 확인합니다:

   ```bash
   export cloudbuild_project_id=$(terraform output -raw cloudbuild_project_id)
   echo "cloud build project ID = ${cloudbuild_project_id}"
   ```

1. 백엔드를 복사하고 Terraform 상태를 위한 Google Cloud Storage 버킷 이름으로 `backend.tf`를 업데이트합니다. 모든 단계의 `backend.tf`도 업데이트합니다.

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

1. `terraform init`을 다시 실행합니다. 메시지가 표시되면 Terraform 상태를 Cloud Storage로 복사하는 데 동의합니다.

   ```bash
   terraform init
   ```

1. (선택사항) `terraform plan`을 실행하여 상태가 올바르게 구성되었는지 확인합니다. 이전 상태에서 변경 사항이 없어야 합니다.
1. 정책 리포지토리를 복제하고 policy-library의 내용을 새 리포지토리에 복사합니다. `terraform-example-foundation` 폴더와 같은 수준에서 리포지토리를 복제합니다.

   ```bash
   cd ../..

   gcloud source repos clone gcp-policies --project=${cloudbuild_project_id}

   cd gcp-policies
   git checkout -b main
   cp -RT ../terraform-example-foundation/policy-library/ .
   ```

1. 변경 사항을 커밋하고 main 브랜치를 정책 리포지토리에 푸시합니다.

   ```bash
   git add .
   git commit -m 'Initialize policy library repo'
   git push --set-upstream origin main
   ```

1. 리포지토리에서 나갑니다.

   ```bash
   cd ..
   ```

1. `0-bootstrap` Terraform 구성을 `gcp-bootstrap` 소스 리포지토리에 저장합니다:

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

1. [1-org](../1-org/README.md) 단계의 지침으로 계속합니다.

**참고 1:** `0-bootstrap` 이후의 단계들은 `terraform_remote_state` 데이터 소스를 사용하여 `0-bootstrap` 단계의 출력에서 조직 ID와 같은 공통 구성을 읽습니다. 상태가 Cloud Storage 버킷에 복사되지 않으면 [실패](../docs/TROUBLESHOOTING.md#error-unsupported-attribute)합니다.

**참고 2:** 배포 후 [문제 해결 가이드](../docs/TROUBLESHOOTING.md#project-quota-exceeded)에 설명된 프로젝트 할당량 오류를 받지 않았더라도, 이 단계에서 생성된 **projects step service account**에 대해 50개의 추가 프로젝트를 요청하는 것을 권장합니다.

## 로컬에서 Terraform 실행하기

다음 단계는 Cloud Build를 사용하지 않고 배포하는 방법을 안내합니다.

1. [terraform-example-foundation](https://github.com/terraform-google-modules/terraform-example-foundation)을 로컬 환경에 복제하고 같은 수준에 `gcp-bootstrap` 폴더를 생성합니다. `0-bootstrap` 내용과 `.gitignore`를 `gcp-bootstrap`에 복사합니다.

   ```bash
   git clone https://github.com/terraform-google-modules/terraform-example-foundation.git

   mkdir gcp-bootstrap

   cp -R terraform-example-foundation/0-bootstrap/* gcp-bootstrap/

   cp terraform-example-foundation/.gitignore gcp-bootstrap
   ```

1. `gcp-bootstrap`로 이동하고 로컬 Git 리포지토리를 초기화하여 로컬에서 버전을 관리합니다. 그런 다음 환경 브랜치를 생성합니다.

   ```bash
   cd gcp-bootstrap

   git init
   git commit -m "initialize empty directory" --allow-empty
   git checkout -b plan

   git checkout -b shared
   ```

1. `terraform.example.tfvars`를 `terraform.tfvars`로 이름을 변경하고 환경의 값으로 파일을 업데이트합니다:

   ```bash
   mv terraform.example.tfvars terraform.tfvars
   ```

1. `cb.tf`를 `cb.tf.example`로 이름을 변경합니다:

   ```bash
   mv cb.tf cb.tf.example
   ```

1. `outputs.tf`에서 Cloud Build 관련 출력을 주석 처리합니다.

1. `sa.tf` 파일에서 Cloud Build와 관련된 줄을 주석 처리합니다. 특히 `cicd_project_iam_member`를 검색하고 해당 모듈과 주석 처리된 모듈에 의존하는 모든 모듈의 "depends_on" 메타 인수를 주석 처리합니다.

1. `sa.tf` 파일에서 `local.cicd_project_id`를 검색하고 해당 코드를 주석 처리합니다.

1. 도우미 스크립트 [validate-requirements.sh](../scripts/validate-requirements.sh)를 사용하여 환경을 검증합니다:

   ```bash
   ../terraform-example-foundation/scripts/validate-requirements.sh -o <ORGANIZATION_ID> -b <BILLING_ACCOUNT_ID> -u <END_USER_EMAIL>
   ```

   **참고:** 스크립트는 사용자가 필요한 역할이 있는 Cloud Identity 또는 Google Workspace 그룹에 있는지 검증할 수 없습니다.

1. `terraform init`과 `terraform plan`을 실행하고 출력을 검토합니다.

   ```bash
   git checkout plan
   terraform init
   terraform plan -input=false -out bootstrap.tfplan
   ```

1. `terraform-example-foundation` 폴더와 같은 디렉토리 수준에 gcp-policies라는 새 폴더를 만듭니다. Git 리포지토리를 초기화하고, `main`이라는 브랜치를 생성한 다음, `terraform-example-foundation` 폴더의 `policy-library` 디렉토리 내용을 gcp-policies 폴더에 복사합니다.

   ```bash
   cd ../

   mkdir gcp-policies

   cd gcp-policies
   git init
   git checkout -b main
   cp -RT ../terraform-example-foundation/policy-library/ .
   ```

1. 정책 리포지토리의 main 브랜치에 변경 사항을 커밋합니다. 이렇게 하면 로컬에서 버전을 관리할 수 있습니다.

   ```bash
   git add .
   git commit -m 'Initialize policy library repo'
   ```

1. `gcp-bootstrap` 리포지토리로 돌아갑니다.

   ```bash
   cd ../gcp-bootstrap
   ```

1. 정책을 검증하려면 `gcloud beta terraform vet`을 실행합니다. 설치 지침은 [Google Cloud CLI 설치](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install)를 참조하세요.

1. 다음 명령을 실행하고 위반 사항이 있는지 확인합니다:

   ```bash
   export VET_PROJECT_ID=A-VALID-PROJECT-ID
   terraform show -json bootstrap.tfplan > bootstrap.json
   gcloud beta terraform vet bootstrap.json --policy-library="$(pwd)/../gcp-policies" --project ${VET_PROJECT_ID}
   ```

   *`A-VALID-PROJECT-ID`*는 액세스 권한이 있는 기존 프로젝트여야 합니다. `gcloud beta terraform vet`이 리소스를 유효한 Google Cloud Platform 프로젝트에 연결해야 하기 때문에 필요합니다.

1. 검증된 코드를 plan 브랜치에 커밋합니다.

   ```bash
   git add .
   git commit -m "Initial version os gcp-bootstrap."
   ```

1. `shared` 브랜치로 체크아웃하고 `plan` 브랜치를 병합합니다. 그런 다음 `terraform apply`를 실행합니다.

   ```bash
   git checkout shared
   git merge plan

   terraform apply bootstrap.tfplan
   ```

1. `terraform output`을 실행하여 수동으로 단계를 실행하는 데 사용될 terraform 서비스 계정의 이메일 주소와 `4-projects` 단계에서 사용될 상태 버킷을 확인합니다.

   ```bash
   export network_step_sa=$(terraform output -raw networks_step_terraform_service_account_email)
   export projects_step_sa=$(terraform output -raw projects_step_terraform_service_account_email)
   export projects_gcs_bucket_tfstate=$(terraform output -raw projects_gcs_bucket_tfstate)

   echo "network step service account = ${network_step_sa}"
   echo "projects step service account = ${projects_step_sa}"
   echo "projects gcs bucket tfstate = ${projects_gcs_bucket_tfstate}"
   ```

1. 백엔드를 복사하고 Terraform 상태를 위한 Google Cloud Storage 버킷 이름으로 `backend.tf`를 업데이트합니다. 모든 단계의 `backend.tf`도 업데이트합니다.

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

1. `terraform init`을 다시 실행합니다. 메시지가 표시되면 Terraform 상태를 Cloud Storage로 복사하는 데 동의합니다.

   ```bash
   terraform init
   ```

1. 새 코드 버전을 커밋하여 로컬에서 버전을 관리할 수 있게 합니다.

   ```sh
   git add backend.tf
   git commit -m "Init gcs backend."
   cd ../
   ```

<!-- BEGINNING OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
## 입력

| 이름 | 설명 | 타입 | 기본값 | 필수 |
|------|-------------|------|---------|:--------:|
| attribute\_condition | Workload Identity Pool Provider attribute condition expression. [More info](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/iam_workload_identity_pool_provider#attribute_condition) | `string` | `null` | no |
| billing\_account | The ID of the billing account to associate projects with. | `string` | n/a | yes |
| bucket\_force\_destroy | 버킷을 삭제할 때 이 부울 옵션은 포함된 모든 객체를 삭제합니다. false인 경우 Terraform은 객체가 포함된 버킷 삭제에 실패합니다. | `bool` | `false` | no |
| bucket\_prefix | 상태 버킷 생성에 사용할 이름 접두사입니다. | `string` | `"bkt"` | no |
| bucket\_tfstate\_kms\_force\_destroy | 버킷을 삭제할 때 이 부울 옵션은 Terraform 상태 버킷에 사용된 KMS 키를 삭제합니다. | `bool` | `false` | no |
| default\_region | 해당하는 경우 리소스를 생성할 기본 지역입니다. | `string` | `"us-central1"` | no |
| default\_region\_2 | 해당하는 경우 리소스를 생성할 보조 기본 지역입니다. | `string` | `"us-west1"` | no |
| default\_region\_gcs | 해당하는 경우 GCS 리소스를 생성할 대소문자 구분 기본 지역입니다. | `string` | `"US"` | no |
| default\_region\_kms | 해당하는 경우 KMS 리소스를 생성할 보조 기본 지역입니다. | `string` | `"us"` | no |
| folder\_deletion\_protection | Terraform이 폴더를 삭제하거나 재생성하는 것을 방지합니다. | `string` | `true` | no |
| folder\_prefix | 폴더 생성에 사용할 이름 접두사입니다. 모든 단계에서 동일해야 합니다. | `string` | `"fldr"` | no |
| groups | 생성할 그룹의 세부 정보를 포함합니다. | <pre>object({<br>    create_required_groups = optional(bool, false)<br>    create_optional_groups = optional(bool, false)<br>    billing_project        = optional(string, null)<br>    required_groups = object({<br>      group_org_admins     = string<br>      group_billing_admins = string<br>      billing_data_users   = string<br>      audit_data_users     = string<br>    })<br>    optional_groups = optional(object({<br>      gcp_security_reviewer    = optional(string, "")<br>      gcp_network_viewer       = optional(string, "")<br>      gcp_scc_admin            = optional(string, "")<br>      gcp_global_secrets_admin = optional(string, "")<br>      gcp_kms_admin            = optional(string, "")<br>    }), {})<br>  })</pre> | n/a | yes |
| initial\_group\_config | 그룹이 초기화될 때의 그룹 구성을 정의합니다. 유효한 값: WITH\_INITIAL\_OWNER, EMPTY, INITIAL\_GROUP\_CONFIG\_UNSPECIFIED. | `string` | `"WITH_INITIAL_OWNER"` | no |
| org\_id | GCP Organization ID | `string` | n/a | yes |
| org\_policy\_admin\_role | 관리자 그룹을 위한 추가 조직 정책 관리자 역할입니다. 테스트 목적으로 사용할 수 있습니다. | `bool` | `false` | no |
| parent\_folder | 선택사항 - 기존 프로젝트가 있는 조직 또는 개발/검증용입니다. 루트 조직 대신 제공된 폴더 아래에 모든 예제 기반 리소스를 배치합니다. 값은 숫자 폴더 ID입니다. 폴더는 이미 존재해야 합니다. | `string` | `""` | no |
| project\_deletion\_policy | 생성된 프로젝트의 삭제 정책입니다. | `string` | `"PREVENT"` | no |
| project\_prefix | 프로젝트 생성에 사용할 이름 접두사입니다. 모든 단계에서 동일해야 합니다. 최대 크기는 3자입니다. | `string` | `"prj"` | no |
| workflow\_deletion\_protection | Terraform이 워크플로우 삭제를 방지할지 여부입니다. 이 필드가 true로 설정되거나 Terraform 상태에서 설정되지 않은 경우, 워크플로우를 삭제하는 `terraform apply` 또는 `terraform destroy`가 실패합니다. 이 필드가 false로 설정되면 워크플로우 삭제가 허용됩니다. | `bool` | `true` | no |

## 출력

| 이름 | 설명 |
|------|-------------|
| bootstrap\_step\_terraform\_service\_account\_email | 부트스트랩 단계 Terraform 계정 |
| cloud\_build\_peered\_network\_id | Cloud Build 피어링 네트워크의 ID입니다. |
| cloud\_build\_private\_worker\_pool\_id | Cloud Build 비공개 워커 풀 ID입니다. |
| cloud\_build\_worker\_peered\_ip\_range | 피어링 서비스 네트워크의 IP 범위입니다. |
| cloud\_build\_worker\_range\_id | Cloud Build 비공개 워커 IP 범위 ID입니다. |
| cloud\_builder\_artifact\_repo | TF Cloud Builder 이미지를 저장하기 위해 생성된 Artifact Registry (AR) 리포지토리입니다. |
| cloudbuild\_project\_id | Cloud Build 구성과 terraform 컴테이너 이미지가 위치할 프로젝트입니다. |
| common\_config | 다른 단계에서 사용할 공통 구성 데이터입니다. |
| csr\_repos | 모듈에서 생성한 Cloud Source Repos 목록으로, Cloud Build 트리거와 연결되어 있습니다. |
| environment\_step\_terraform\_service\_account\_email | 환경 단계 Terraform 계정 |
| gcs\_bucket\_cloudbuild\_artifacts | CICD 프로젝트에서 Cloud Build 아티팩트를 저장하는 데 사용되는 버킷입니다. |
| gcs\_bucket\_cloudbuild\_logs | CICD 프로젝트에서 Cloud Build 로그를 저장하는 데 사용되는 버킷입니다. |
| gcs\_bucket\_tfstate | 시드 프로젝트의 기반 파이프라인에 대한 terraform 상태를 저장하는 데 사용되는 버킷입니다. |
| networks\_step\_terraform\_service\_account\_email | 네트워크 단계 Terraform 계정 |
| optional\_groups | 예제 기반 단계에 선택사항인 Google 그룹 목록입니다. |
| organization\_step\_terraform\_service\_account\_email | 조직 단계 Terraform 계정 |
| projects\_gcs\_bucket\_tfstate | 시드 프로젝트의 4단계 프로젝트 기반 파이프라인에 대한 terraform 상태를 저장하는 데 사용되는 버킷입니다. |
| projects\_step\_terraform\_service\_account\_email | 프로젝트 단계 Terraform 계정 |
| required\_groups | 예제 기반 단계에서 필수인 Google 그룹 목록입니다. |
| seed\_project\_id | 서비스 계정과 핵심 API가 활성화될 프로젝트입니다. |

<!-- END OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
