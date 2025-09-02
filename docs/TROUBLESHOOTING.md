# 문제 해결

## 용어

[GLOSSARY.md](./GLOSSARY.md)를 참조하세요.

- - -

## 문제

- [일반적인 문제](#common-issues)
- [호출자가 조직에서 권한이 없음](#caller-does-not-have-permission-in-the-organization)
- [요금 스이 할당량 초과](#billing-quota-exceeded)
- [Terraform 상태 잠금 획득 오류](#terraform-error-acquiring-the-state-lock)

- - -

## 일반적인 문제

- [프로젝트 할당량 초과](#project-quota-exceeded)
- [기본 브랜치 설정](#default-branch-setting)
- [Terraform 상태 스냅샷 잠금](#terraform-state-snapshot-lock)
- [최종 사용자 자격 증명을 사용한 애플리케이션 인증](#application-authenticated-using-end-user-credentials)
- [Cloud Shell에서 요청된 주소를 할당할 수 없음 오류](#cannot-assign-requested-address-error-in-cloud-shell)
- [오류: 지원되지 않는 속성](#error-unsupported-attribute)
- [오류: 네트워크 피어링 추가 오류](#error-error-adding-network-peering)
- [오류: GitLab 리포지토리를 찾을 수 없어 Terraform 배포 실패](#terraform-deploy-fails-due-to-gitlab-repositories-not-found)
- [오류: GitLab 파이프라인 액세스 거부](#gitlab-pipelines-access-denied)
- [오류: 4-project 단계 컨텍스트에서 알 수 없는 프로젝트 ID](#error-unknown-project-id-on-4-project-step-context)
- [오류: TagValue 커밋 목적으로 작업 가져오기 오류](#error-error-getting-operation-for-committing-purpose-for-tagvalue)
- [사용자가 프로젝트에 액세스할 권한이 없거나 프로젝트가 존재하지 않을 수 있습니다](#the-user-does-not-have-permission-to-access-project-or-it-may-not-exist)
- - -

### 프로젝트 할당량 초과

**오류 메시지:**

```text
Error code 8, message: The project cannot be created because you have exceeded your allotted project quota
```

**원인:**

이 메시지는 [프로젝트 생성 할당량](https://support.google.com/cloud/answer/6330231)에 도달했음을 의미합니다.

**해결 방법:**

이 경우 [프로젝트 할당량 증가 요청](https://support.google.com/code/contact/project_quota_increase) 양식을 사용하여 할당량 증가를 요청할 수 있습니다.

지원 양식에서 **프로젝트 생성에 사용될 이메일 주소** 필드에는 Terraform Example Foundation 0-bootstrap 단계에서 생성된 `projects_step_terraform_service_account_email`의 이메일 주소를 사용하세요.

**참고:**

- 다른 할당량 오류가 나타나면 [할당량 문서](https://cloud.google.com/docs/quota)를 참조하세요.

### 기본 브랜치 설정

**오류 메시지:**

```text
오류: 소스 참조 사양 master가 어떤 것과도 일치하지 않습니다
```

**원인:**

init.defaultBranch가 `main`이 아닌 다른 값으로 설정되어 있기 때문일 수 있습니다.

**해결 방법:**

1. 기본 브랜치를 확인합니다:

   ```bash
   git config init.defaultBranch
   ```

   main 브랜치에 있는 경우 `main`을 출력합니다.
1. 기본 브랜치가 `main`으로 설정되어 있지 않으면 설정합니다:

   ```bash
   git config --global init.defaultBranch main
   ```

### Terraform 상태 스냅샷 잠금

**오류 메시지:**

**Foundation CI/CD 파이프라인**의 3-networks 단계에서 `production` 브랜치에 대한 빌드를 실행할 때 다음과 같이 빌드가 실패합니다:

```
state snapshot was created by Terraform v1.x.x, which is newer than current v1.5.7; upgrade to Terraform v1.x.x or greater to work with this state
```

**원인:**

[3-networks](../3-networks#deploying-with-cloud-build)의 shared 환경에 대한 수동 배포 단계가 **Foundation CI/CD 파이프라인**에서 사용되는 v1.5.7보다 더 새로운 Terraform 버전으로 실행되었습니다.

**해결 방법:**

두 가지 옵션이 있습니다:

#### 로컬 Terraform 버전 다운그레이드

Terraform v1.5.7로 3-networks shared 환경의 배포를 다시 실행해야 합니다.

단계:

- `gcp-networks/envs/shared/` 폴더로 이동합니다.
- 0-bootstrap 단계에서 가져온 버킷 이름으로 `backend.tf`를 업데이트합니다.
- Terraform v1.x.x 버전을 사용하여 폴더에서 `terraform destroy`를 실행합니다.
- `gs://YOUR-TF-STATE-BUCKET/terraform/networks/envs/shared/default.tfstate`의 Terraform 상태 파일을 삭제합니다. 이 버킷은 **Seed Project**에 있습니다.
- Terraform v1.5.7을 설치합니다.
- Terraform v1.5.7을 사용하여 3-networks shared 환경의 수동 배포를 다시 실행합니다.

#### 0-bootstrap 러너 이미지 Terraform 버전 업그레이드

다음 지침에서 `1.x.x`를 로컬 Terraform 버전의 실제 버전으로 바꿉니다:

- `0-bootstrap` 폴더로 이동합니다.
- Terraform [cb.tf](../0-bootstrap/cb.tf) 파일의 로컬 `terraform_version`을 편집합니다:
  - 로컬 `terraform_version`을 `"1.5.7"`에서 `"1.x.x"`로 업그레이드
- `terraform init`을 실행합니다.
- `terraform plan`을 실행하고 출력을 검토합니다.
- `terraform apply`를 실행합니다.

### 최종 사용자 자격 증명을 사용한 애플리케이션 인증

**오류 메시지:**

Cloud Shell에서 다음과 같이 `gcloud` 명령을 실행할 때

```bash
gcloud scc notifications describe <scc_notification_name> --organization YOUR_ORGANIZATION_ID
```

또는

```bash
gcloud access-context-manager policies list --organization YOUR_ORGANIZATION_ID --format="value(name)"
```

다음 오류가 발생합니다:

```text
Error 403: Your application has authenticated using end user credentials from the Google Cloud SDK or Google Cloud Shell which are not supported by the X.googleapis.com.
We recommend configuring the billing/quota_project setting in gcloud or using a service account through the auth/impersonate_service_account setting.
서비스 계정과 애플리케이션에서 사용하는 방법에 대한 자세한 내용은 https://cloud.google.com/docs/authentication/을 참조하세요.
```

**원인:**

Cloud Shell에서 애플리케이션 기본 자격 증명을 사용할 때 `securitycenter.googleapis.com` 또는 `accesscontextmanager.googleapis.com`과 같은 API에 대한 청구 프로젝트를 사용할 수 없습니다.

**해결 방법:**

가장 또는 청구 프로젝트를 제공하여 명령을 다시 실행할 수 있습니다:

- Terraform 서비스 계정 가장

```bash
--impersonate-service-account=terraform-org-sa@<SEED_PROJECT_ID>.iam.gserviceaccount.com
```

- 청구 프로젝트 제공

```bash
--billing-project=<A-VALID-PROJECT-ID>
```

청구 프로젝트를 제공하는 경우 billing_project에 대한 `serviceusage.services.use` 권한이 있어야 합니다.

### Cloud Shell에서 요청된 주소를 할당할 수 없음 오류

**오류 메시지:**

이 리포지토리의 코드를 배포하기 위해 [Google Cloud Shell](https://cloud.google.com/shell/docs)을 사용할 때

```text
dial tcp [2607:f8b0:400c:c15::5f]:443: connect: cannot assign requested address
```

Terraform이 Google API를 호출할 때 위와 같은 오류가 발생할 수 있습니다.

**원인:**

이는 IPv6에 관련된 [알려진 terraform 문제](https://github.com/hashicorp/terraform-provider-google/issues/6782)입니다.

**해결 방법:**

현재 대안은 다음과 같습니다:

1. Cloud Shell에서 Google API 호출이 `private.googleapis.com` 범위(199.36.153.8/30)의 IP를 사용하도록 강제하는 [해결 방법](https://stackoverflow.com/a/62827358)을 사용하거나
1. IPv6을 지원하는 로컬 머신에서 foundation 코드를 배포하는 것입니다.

해결 방법을 사용하는 경우 API 목록에는 terraform-example-foundation 정책 라이브러리에서 [허용된](../policy-library/policies/constraints/serviceusage_allow_basic_apis.yaml) API가 포함되어야 합니다.

### 오류: 지원되지 않는 속성

**오류 메시지:**

```text
오류: 지원되지 않는 속성

  on main.tf line 22, in locals:
  22:   org_id               = data.terraform_remote_state.bootstrap.outputs.common_config.org_id
    ├────────────────
    │ data.terraform_remote_state.bootstrap.outputs is object with no attributes

This object does not have an attribute named "common_config".
```

**원인:**

`0-bootstrap` 이후의 단계들은 `terraform_remote_state` 데이터 소스를 사용하여 `0-bootstrap` 단계의 출력에서 조직 ID와 같은 공통 구성을 읽습니다.
이 오류는 `0-bootstrap` 단계의 Terraform 상태가 `0-bootstrap` 단계에서 생성된 Terraform 상태 버킷으로 복사되지 않았음을 의미합니다.

**해결 방법:**

`0-bootstrap` README의 [Cloud Build로 배포](../0-bootstrap/README.md#deploying-with-cloud-build) 섹션 끝에 있는 지침을 따라 Terraform 상태를 `0-bootstrap` 단계에서 생성된 Cloud Storage 버킷으로 복사하고 배포 중인 단계의 계획/적용을 다시 시도하세요.

### 오류: 네트워크 피어링 추가 오류

**오류 메시지:**

```text
오류: 네트워크 피어링 추가 오류: googleapi: 오류 403: 속도 제한 초과
Details:
[
  {
    "@type": "type.googleapis.com/google.rpc.ErrorInfo",
    "domain": "compute.googleapis.com",
    "metadatas": {
      "containerId": "76352966089",
      "containerType": "PROJECT",
      "location": "global"
    },
    "reason": "CONCURRENT_OPERATIONS_QUOTA_EXCEEDED"
  }
]
, rateLimitExceeded

```

**원인:**

[Hub and Spoke](https://cloud.google.com/architecture/security-foundations/networking#hub-and-spoke) 네트워크 모드를 사용하는 배포에서 피어링 작업이 너무 많아 Hub 네트워크와 Spoke 네트워크 간의 네트워크 피어링을 추가할 때 오류가 발생합니다.

**해결 방법:**

이는 일시적인 오류이므로 배포를 다시 시도할 수 있습니다. 최소 1분 대기하고 배포를 다시 시도하세요.


### 오류: 4-project 단계 컨텍스트에서 알 수 없는 프로젝트 ID

**오류 메시지:**

```text
Error 400: Unknown project id: 'prj-<business-unity>-<environment>-svpc-<random-suffix>', invalid
```

**원인:**

**0-bootstrap 단계에서 생성된 프로젝트 서비스 계정**에 대한 추가 프로젝트 할당량을 요청하지 않고 4-projects 단계를 실행하려고 할 때, 프로젝트 할당량 문제가 해결된 후에도 terraform 상태의 뺈일치로 인해 위와 같은 오류가 발생할 수 있습니다.

**해결 방법:**

- 다음 단계를 실행하기 전에 **프로젝트 SA 이메일**에 대한 [추가 프로젝트 할당량을 요청](#project-quota-exceeded)했는지 확인하세요.

terraform 상태의 뺈일치를 수정하기 위해 누락된 프로젝트의 재생성을 트리거하도록 일부 Terraform 리소스를 **tainted**로 표시해야 합니다.

1. 터미널에서 오류가 보고되는 경로로 이동합니다.

   예를 들어, 알 수 없는 프로젝트 ID가 `prj-bu1-p-svpc`인 경우 ./gcp-projects/business_unit_1/production으로 이동해야 합니다(`bu1`로 인한 `business_unit_1`과 `p`로 인한 `production`, 프로젝트 이름 지정 가이드라인에 대한 자세한 정보는 [명명 규칙](https://cloud.google.com/architecture/security-foundations/using-example-terraform#naming_conventions)을 참조하세요).

   ```bash
   cd ./gcp-projects/<business_unit>/<environment>
   ```

1. 원격 상태를 가져올 수 있도록 `terraform init` 명령을 실행합니다.

   ```bash
   terraform init
   ```

1. `random_project_id_suffix`로 필터링하여 `terraform state list` 명령을 실행합니다.
이 명령은 랜덤 접미사를 사용하는 이 BU 및 환경에 대해 생성되어야 하는 모든 예상 프로젝트를 제공합니다.

   ```bash
   terraform state list | grep random_project_id_suffix
   ```

1. 환경의 프로젝트 부모인 폴더를 식별합니다.
Terraform Example Foundation이 조직 하에 직접 배포된 경우 `--organization`을 사용하고, Terraform Example Foundation이 폴더 하에 배포된 경우 `--folder`를 사용합니다. "ORGANIZATION_ID"와 "PARENT_FOLDER"는 0-bootstrap 단계에 제공된 입력 값입니다.

   ```bash
   gcloud resource-manager folders list [ --organization=ORGANIZATION_ID ][ --folder=PARENT_FOLDER ]
   ```

1. `gcloud` 명령의 결과는 다음 출력과 같아 보입니다.
이 예에서 `production` 환경을 사용하면 환경의 폴더 ID는 `333333333333`이 됩니다.

   ```
   DISPLAY_NAME         PARENT_NAME                     ID
   fldr-bootstrap       folders/PARENT_FOLDER  111111111111
   fldr-common          folders/PARENT_FOLDER  222222222222
   fldr-production      folders/PARENT_FOLDER  333333333333
   fldr-nonproduction  folders/PARENT_FOLDER  444444444444
   fldr-development     folders/PARENT_FOLDER  555555555555
   ```

1. `gcloud projects list` 명령을 실행합니다.
이전 단계에서 검색된 폴더의 적절한 ID로 `id_of_the_environment_folder`를 바꿉니다.
이 명령은 실제로 생성된 모든 프로젝트를 제공합니다.

   ```bash
   gcloud projects list --filter="parent=<id_of_the_environment_folder>"
   ```

1. `gcloud projects list` 단계에서 반환되지 **않은** 프로젝트에 대한 `terraform state` 단계에 나열된 각 리소스에 대해 terraform 상태의 뺈일치를 수정하기 위해 해당 리소스를 tainted로 표시하여 재생성을 강제해야 합니다.

   ```bash
   terraform taint <resource>[index]
   ```

   예를 들어, 다음 명령에서는 env secrets 프로젝트를 tainted로 표시하고 있습니다. 누락된 프로젝트 수에 따라 `terraform taint` 명령을 여러 번 실행해야 할 수 있습니다.

   ```bash
   terraform taint module.env.module.env_secrets_project.module.project.module.project-factory.random_string.random_project_id_suffix[0]
   ```

1. 일치하지 않는 모든 항목에 대해 `terraform taint` 명령을 실행한 후 Cloud Build로 이동하여 실패한 작업에 대한 재시도 작업을 트리거합니다.
이는 성공적으로 완료되어야 하며, 다른 BU/환경에 대해 유사한 다른 오류가 발생하면 오류 로그에 보고된 BU/환경에 따라 경로를 변경하여 이 가이드를 다시 따라야 합니다.

**참고:**

   - terraform state list 단계에서 반환된 줄 끝에 [number]를 포함하는 리소스에대해서만 taint 명령을 실행해야 합니다. 그룹(끝에 []가 없는 리소스)에 대해서는 실행할 필요가 없습니다.

### 오류: TagValue 커밋 목적으로 작업 가져오기 오류

**오류 메시지:**

```text
오류: TagValue 생성 대기 오류: TagValue 생성 대기 오류: 오류 코드 13, 메시지: TagValue 커밋 목적 작업 가져오기 오류: tagValues/{tag_value_id}
```

**원인:**

[google_tags_tag_value](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/tags_tag_value)를 배포할 때 때때로 오류가 발생하여 Terraform이 실행을 완료할 수 없습니다.

**해결 방법:**

1. 이는 일시적인 오류이므로 배포를 다시 시도할 수 있습니다.
1. 통합 테스트 중에 이 오류를 방지하기 위해 재시도 정책이 추가되었습니다.
- - -

### 호출자가 조직에서 권한이 없음

**오류 메시지:**

```text
오류: 조직 읽기 또는 편집 시 오류 조직을 찾을 수 없습니다 : <organization-id>: googleapi: 오류 403: 호출자에게 권한이 없습니다, 금지됨
```

**원인:**

Terraform을 실행하는 사용자가 조직 수준에서 [조직 관리자](https://cloud.google.com/iam/docs/understanding-roles#resource-manager-roles) 미리 정의된 역할을 갖고 있지 않습니다.

**해결 방법:**

- 사용자가 **조직 관리자 역할을 갖고 있지 않다면** 다음을 시도하세요:

조직 관리 팀에 사용자에게 역할을 부여하도록 요청해야 합니다.

- 사용자가 **조직 관리자 역할을 갖고 있다면** 다음을 시도하세요:

```bash
gcloud auth application-default login
gcloud auth list # <- 올바른 계정 옆에 별표가 있는지 확인
```

후에 `terraform`을 다시 실행하세요.

### 청구 할당량 초과

**오류 메시지:**

```text
오류: 프로젝트 "projects/some-project"에 대한 청구 계정 "XXXXXX-XXXXXX-XXXXXX" 설정 오류: googleapi: 오류 400: 전제 조건 확인 실패., 전제조건실패
```

**원인:**

대부분 이는 청구 할당량 문제와 관련이 있습니다.

**해결 방법:**

다음을 시도해 보세요

```bash
gcloud alpha billing projects link projects/some-project --billing-account XXXXXX-XXXXXX-XXXXXX
```

출력에 `Cloud billing quota exceeded`가 표시되면 [청구 할당량 증가 요청](https://support.google.com/code/contact/billing_quota_increase) 양식을 사용하여 청구 할당량 증가를 요청할 수 있습니다.

### Terraform 상태 잠금 획득 오류

**오류 메시지:**

```text
오류: 상태 잠금 획득 오류
```

**원인:**

이 메시지는 [잠금 상태](https://www.terraform.io/language/state/locking)에 있는 원격 백엔드를 사용하여 Terraform 구성을 적용하려고 하고 있음을 의미합니다.

Terraform 프로세스가 빌드 시간 초과 또는 terraform 프로세스 종료와 같은 예상치 못한 이벤트로 인해 완료할 수 없는 경우 Terraform 상태가 **잠김** 상태로 유지됩니다.

**해결 방법:**

다음 명령은 Foundation Example의 한 부분인 2-environments 단계의 **development 환경** 잠금을 해제하는 방법의 예시입니다.
다른 부분에도 동일하게 적용할 수 있습니다.

1. Terraform 상태 잠금을 받은 리포지토리를 복제합니다. 다음 예시는 2-environments 단계의 **development 환경**을 가정합니다:

   ```bash
   gcloud source repos clone gcp-environments --project=YOUR_CLOUD_BUILD_PROJECT_ID
   ```

1. 리포지토리로 이동하고 development 브랜치로 변경합니다:

   ```bash
   cd gcp-environments
   git checkout development
   ```

1. 프로젝트에 원격 백엔드가 없는 경우 다음 2개 명령을 건너뛰고 `terraform init` 명령으로 바로 이동할 수 있습니다.
1. 프로젝트에 원격 백엔드가 있는 경우 원격 상태 백엔드 버킷으로 `backend.tf`를 업데이트해야 합니다.
다음 명령을 실행하여 `0-bootstrap` 단계에서 이 정보를 가져올 수 있습니다:

   ```bash
   terraform output gcs_bucket_tfstate
   ```

1. 이전에 받은 원격 상태 백엔드 버킷으로 `<YOUR-REMOTE-STATE-BACKEND-BUCKET>` 내의 `backend.tf`를 업데이트합니다:

   ```bash
   for i in `find . -name 'backend.tf'`; do sed -i'' -e 's/UPDATE_ME/<YOUR-REMOTE-STATE-BACKEND-BUCKET>/' $i; done
   ```

1. terraform 구성 파일이 있는 `envs/development`로 이동하고 terraform init을 실행합니다:

   ```bash
   cd envs/development
   terraform init
   ```

1. 이 시점에서 Terraform 상태 잠금 정보를 가져오고 상태를 잠금 해제할 수 있습니다.
1. terraform apply를 실행한 후 다음과 같은 오류 메시지가 나타납니다:

   ```text
   terraform apply
   Acquiring state lock. This may take a few moments...
   ╷
   │ 오류: 상태 잠금 획득 오류
   │
   │ Error message: writing "gs://<YOUR-REMOTE-STATE-BACKEND-BUCKET>/<PATH-TO-TERRAFORM-STATE>/<tf state file name>.tflock" failed: googleapi: Error 412: At least one
   │ of the pre-conditions you specified did not hold., conditionNotMet
   │ Lock Info:
   │   ID:        1664568683005669
   │   Path:      gs://<YOUR-REMOTE-STATE-BACKEND-BUCKET>/<PATH-TO-TERRAFORM-STATE>/<tf state file name>.tflock
   │   Operation: OperationTypeApply
   │   Who:       user@domain
   │   Version:   1.0.0
   │   Created:   2022-09-30 20:11:22.90644727 +0000 UTC
   │   Info:
   │
   │
   │ Terraform acquires a state lock to protect the state from being written
   │ 여러 사용자가 동시에 사용하고 있습니다. 위의 문제를 해결한 후 다시
   │ 시도하세요. 대부분의 명령에서는 "-lock=false"로 잠금을 비활성화할 수 있습니다
   │ 플래그로, 하지만 이는 권장되지 않습니다.
   ```

1. 잠금 `ID`를 사용하여 `terraform force-unlock` 명령을 사용하여 Terraform 상태 잠금을 제거할 수 있습니다. 실행하기 전에 [terraform force-unlock](https://www.terraform.io/language/state/locking#force-unlock) 명령에 관한 공식 문서를 검토할 것을 **강력히 권장**합니다.
1. Terraform 상태를 잠금 해제한 후 상태를 검토하기 위해 `terraform plan`을 실행할 수 있습니다. 다음 링크는 구성에 대한 Terraform 상태를 복구하고 계속 진행하는 데 도움이 될 수 있습니다:
    1. [Terraform 상태 조작](https://developer.hashicorp.com/terraform/cli/state)
    1. [리소스 이동](https://developer.hashicorp.com/terraform/cli/state/move)
    1. [인프라 가져오기](https://developer.hashicorp.com/terraform/cli/import)

**Terraform 상태 잠금 가능한 원인:**

- Terraform 상태 잠금이 빌드 시간 초과로 인한 것임을 인식한 경우 [빌드 구성](https://github.com/terraform-google-modules/terraform-example-foundation/blob/master/build/cloudbuild-tf-apply.yaml#L15)에서 빌드 시간 초과 시간을 늘리세요.

### GitLab 리포지토리를 찾을 수 없어 Terraform 배포 실패

**오류 메시지:**

```text
오류: POST https://gitlab.com/api/v4/projects/<GITLAB-ACCOUNT>/<GITLAB-REPOSITORY>/variables: 404 {메시지: 404 프로젝트를 찾을 수 없습니다}

```

**원인:**

이 메시지는 잘못된 액세스 토큰을 사용하고 있거나 GitLab 계정/그룹과 GitLab 리포지토리 모두에 액세스 토큰이 생성되었음을 의미합니다.

GitLab 계정/그룹 하의 개인 액세스 토큰만 존재해야 합니다.

**해결 방법:**

Google Secure Foundation Blueprint에서 사용하는 GitLab 리포지토리에서 모든 액세스 토큰을 제거하세요.

### GitLab 파이프라인 액세스 거부

**오류 메시지:**

파이프라인 작업 로그에서:

```text
Error response from daemon: pull access denied for registry.gitlab.com/<YOUR-GITLAB-ACCOUNT>/<YOUR-GITLAB-CICD-REPO>/terraform-gcloud, repository does not exist or may require 'docker login': denied: requested access to the resource is denied
```

**원인:**

이 메시지의 원인은 CI/CD 리포지토리의 토큰 액세스 설정에서 "이 프로젝트에 대한 액세스 제한"이 활성화되어 있기 때문입니다.

**해결 방법:**

Terraform Example Foundation에서 사용될 모든 프로젝트/리포지토리를 다음 단계의 허용 목록에 추가하세요:
`CI/CD Repo -> Settings -> CI/CD -> Token Access -> Allow CI job tokens from the following projects to access this project`.

### 사용자가 프로젝트에 액세스할 권한이 없거나 프로젝트가 존재하지 않을 수 있습니다

**오류 메시지:**

```text
Error when reading or editing GCS service account not found: googleapi: Error 400: Unknown project id: <PROJECT-ID>, invalid.
The user does not have permission to access Project <PROJECT-ID> or it may not exist.
```

**원인:**

Terraform이 주어진 프로젝트 **PROJECT-ID**와 관련된 리소스를 가져오거나 조작하려고 하지만 첫 번째 실행에서 프로젝트가 생성되지 않았습니다.

첫 번째 실행에서 생성된 것은 프로젝트를 만드는 데 사용될 프로젝트 ID입니다. 프로젝트 ID는 고정 접두사와 랜덤 접미사의 조합입니다.

첫 번째 실행에서 프로젝트 생성 실패의 가능한 원인은 다음과 같습니다:

- 사용자가 청구 계정에서 청구 계정 사용자 역할을 가지고 있지 않음
- 사용자가 Google Cloud 조직에서 프로젝트 작성자 역할을 가지고 있지 않음
- 사용자가 프로젝트 생성 할당량에 도달함
- 시간 초과 또는 중단으로 인해 Terraform apply가 중간에 실패하여 상태에 프로젝트 ID는 생성되었지만 프로젝트 자체는 생성되지 않음

**해결 방법:**

원인이 프로젝트 생성 할당량 문제인 경우 Terraform Example Foundation [문제 해결](https://github.com/terraform-google-modules/terraform-example-foundation/blob/master/docs/TROUBLESHOOTING.md#billing-quota-exceeded)의 지침을 따르세요.

이러한 수정을 수행한 후 프로젝트 ID에 사용되는 랜덤 접미사의 재생성을 강제해야 합니다.
재생성을 강제하려면 다음을 실행하세요

```bash
terraform taint <RESOURCE-ID>
```

예를 들어

```
terraform taint module.seed_bootstrap.module.seed_project.module.project-factory.random_id.random_project_id_suffix
```

그리고 배포를 다시 시도하세요.
