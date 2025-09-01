# Terraform Example Foundation 배포 헬퍼

Cloud Build 및 Cloud Source 저장소를 사용하여 Terraform example foundation을 배포하는 헬퍼 도구입니다.

## 사용법

## 요구 사항

- [Go](https://go.dev/doc/install) 1.22 이상
- [Google Cloud SDK](https://cloud.google.com/sdk/install) 버전 393.0.0 이상
- [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) 버전 2.28.0 이상
- [Terraform](https://www.terraform.io/downloads.html) 버전 1.5.7 이상
- Foundation을 배포하는 사용자에 대한 추가 IAM [요구 사항](../../0-bootstrap/README.md#prerequisites)은 `0-bootstrap` README를 참조하십시오.
- Security Command Center를 활성화하려면 Security Command Center 계층을 선택하고 [Security Command Center 설정](https://cloud.google.com/security-command-center/docs/quickstart-security-command-center)에 설명된 대로 Security Command Center 서비스 계정에 대한 권한을 생성하고 부여하십시오.

환경에서는 빌드 파이프라인에서 사용하는 것과 동일한 [Terraform](https://www.terraform.io/downloads.html) 버전을 사용해야 합니다.
그렇지 않으면 Terraform 상태 스냅샷 잠금 오류가 발생할 수 있습니다.

버전 1.5.7은 라이선스 모델 변경 전 마지막 버전입니다. 최신 버전의 Terraform을 사용하려면 `3-networks` 및 `4-projects`의 단계 일부를 수동으로 실행하기 위해 운영 시스템에서 사용하는 Terraform 버전이 다음 코드에 구성된 버전과 동일한지 확인하십시오

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

### 필요한 도구 검증

- 필요한 도구인 Go 1.22.0+, Terraform 1.5.7+, gcloud 393.0.0+, Git 2.28.0+가 설치되어 있는지 확인하십시오:

    ```bash
    go version

    terraform -version

    gcloud --version

    git --version
    ```

- `gcloud`의 필요한 구성 요소가 설치되어 있는지 확인하십시오:

    ```bash
    gcloud components list --filter="id=beta OR id=terraform-tools"
    ```

- `beta` 및 `terraform-tools` 구성 요소가 설치되어 있지 않으면 명령 출력의 지침에 따라 설치하십시오.

### 배포 환경 준비

- 생성될 Cloud Source 저장소와 terraform example foundation의 복사본을 호스팅할 파일 시스템에 디렉토리를 생성하십시오.
- 이 디렉토리에 `terraform-example-foundation` 저장소를 복제하십시오.

    ```text
    deploy-directory/
    └── terraform-example-foundation
    ```

- [global.tfvars.example](./global.tfvars.example) 파일을 같은 디렉토리에 `global.tfvars`로 복사하십시오.

    ```text
    deploy-directory/
    └── global.tfvars
    └── terraform-example-foundation
    ```

- `global.tfvars`를 환경의 값으로 업데이트하십시오.
- `0-bootstrap` README [전제 조건](https://github.com/terraform-google-modules/terraform-example-foundation/blob/master/0-bootstrap/README.md#prerequisites) 섹션에는 이 헬퍼를 실행하는 데 필요한 추가 전제 조건이 있습니다.
- 변수 `code_checkout_path`는 `deploy-directory` 디렉토리의 전체 경로입니다.
- 변수 `foundation_code_path`는 `terraform-example-foundation` 디렉토리의 전체 경로입니다.
- 추가 정보는 단계별 README를 참조하십시오:
  - [0-bootstrap](https://github.com/terraform-google-modules/terraform-example-foundation/blob/master/0-bootstrap/README.md)
  - [1-org](https://github.com/terraform-google-modules/terraform-example-foundation/blob/master/1-org/README.md)
  - [2-environments](https://github.com/terraform-google-modules/terraform-example-foundation/blob/master/2-environments/README.md)
  - [3-networks-svpc](https://github.com/terraform-google-modules/terraform-example-foundation/blob/master/3-networks-svpc)
  - [3-networks-hub-and-spoke](https://github.com/terraform-google-modules/terraform-example-foundation/blob/master/3-networks-hub-and-spoke)
  - [4-projects](https://github.com/terraform-google-modules/terraform-example-foundation/blob/master/4-projects)
  - [5-app-infra](https://github.com/terraform-google-modules/terraform-example-foundation/blob/master/5-app-infra)

### 위치

기본적으로 foundation 지역 리소스는 `us-west1` 및 `us-central1` 지역에 배포되고 다중 지역 리소스는 `US` 다중 지역에 배포됩니다.

위치 구성을 위해 `global.tfvars` 파일에 선언된 변수 외에도 네트워크 단계(`3-networks-svpc` 및 `3-networks-hub-and-spoke`)의 각 환경(`production`, `nonproduction`, `development`)에 두 개의 로컬 변수인 `default_region1` 및 `default_region2`가 있습니다.
이들은 각 환경의 [main.tf](../../3-networks-svpc/envs/production/main.tf#L20-L21) 파일에 위치합니다.
다른 지역에 배포하려면 배포를 시작하기 **전에** 두 로컬 변수를 변경하십시오.

**참고:** `global.tfvars` 파일의 `default_region` 변수에 사용된 지역은 `default_region1` 및 `default_region2` 로컬 변수에 사용된 지역 중 하나여야 **합니다**.

### 애플리케이션 기본 자격 증명

- `gcloud` 구성에서 청구 할당량 프로젝트를 설정하십시오

    ```
    gcloud config set billing/quota_project <QUOTA-PROJECT>

    gcloud services enable \
    "cloudresourcemanager.googleapis.com" \
    "iamcredentials.googleapis.com" \
    "cloudbuild.googleapis.com" \
    "securitycenter.googleapis.com" \
    "accesscontextmanager.googleapis.com" \
    --project <QUOTA-PROJECT>
    ```

- [애플리케이션 기본 자격 증명](https://cloud.google.com/sdk/gcloud/reference/auth/application-default/login)을 구성하십시오

    ```bash
    gcloud auth application-default login
    ```

### 헬퍼 실행

- 헬퍼를 설치하십시오:

    ```bash
    go install
    ```

- tfvars 파일을 검증하십시오. `global.tfvars` 파일에 `validator_project_id`를 구성한 경우 `validate` 플래그는 Secure Command Center 알림 이름과 Tag Key 이름에 대한 추가 검사를 수행합니다.
이러한 추가 검사를 위해서는 최소한 *Security Center Notification Configurations Viewer* (`roles/securitycenter.notificationConfigViewer`) 및 *Tag Viewer* (`roles/resourcemanager.tagViewer`) 역할이 필요합니다:

    ```bash
    $HOME/go/bin/foundation-deployer -tfvars_file <'global.tfvars' 파일 경로> -validate
    ```

- 헬퍼를 실행하십시오:

    ```bash
    $HOME/go/bin/foundation-deployer -tfvars_file <'global.tfvars' 파일 경로>
    ```

- 추가 출력을 억제하려면 다음을 사용하십시오:

    ```bash
    $HOME/go/bin/foundation-deployer -tfvars_file <'global.tfvars' 파일 경로> -quiet
    ```

- 배포를 삭제하려면 다음을 실행하십시오:

    ```bash
    $HOME/go/bin/foundation-deployer -tfvars_file <'global.tfvars' 파일 경로> -destroy
    ```

- After deployment:

    ```text
    deploy-directory/
    └── bu1-example-app
    └── gcp-bootstrap
    └── gcp-environments
    └── gcp-networks
    └── gcp-org
    └── gcp-policies
    └── gcp-policies-app-infra
    └── gcp-projects
    └── global.tfvars
    └── terraform-example-foundation
    ```

### 지원되는 플래그

```bash
  -tfvars_file file
        사용할 구성이 있는 Terraform .tfvars 파일의 전체 경로.
  -steps_file file
        진행 상황을 저장하는 데 사용할 단계 파일의 경로. (기본값 ".steps.json")
  -list_steps
        기존 단계를 나열합니다.
  -reset_step step
        재설정할 단계의 이름. 단계가 대기 상태로 표시됩니다.
  -validate
        tfvars 파일 입력을 검증합니다
  -quiet
        true이면 추가 출력이 억제됩니다.
  -disable_prompt
        대화형 프롬프트를 비활성화합니다.
  -destroy
        배포를 삭제합니다.
  -help
        이 도움말 텍스트를 인쇄하고 종료합니다.
```

## 문제 해결

이 배포 중에 문제가 발생하면 [문제 해결](../../docs/TROUBLESHOOTING.md)을 참조하십시오.
