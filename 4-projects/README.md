# 4-projects

이 리포지토리는 [Google Cloud 보안 기반 가이드](https://cloud.google.com/architecture/security-foundations)에서 설명된
example.com 참조 아키텍처를 구성하고 배포하는 방법을 보여주는 다중 파트 가이드의 일부입니다. 다음 표는 가이드의 부분들을 나열합니다.

<table>
<tbody>
<tr>
<td><a href="../0-bootstrap">0-bootstrap</a></td>
<td>Google Cloud 조직을 부트스트랩하여 Cloud Foundation Toolkit(CFT) 사용을 시작하는 데 필요한 모든 리소스와
권한을 생성합니다. 이 단계는 또한 후속 단계에서 기반 코드를 위한 <a href="../docs/GLOSSARY.md#foundation-cicd-pipeline">CI/CD 파이프라인</a>을 구성합니다.</td>
</tr>
<tr>
<td><a href="../1-org">1-org</a></td>
<td>최상위 수준 공유 폴더, 네트워킹 프로젝트 및 조직 수준 로깅을 설정하고,
조직 정책을 통해 기준선 보안 설정을 구성합니다.</td>
</tr>
<tr>
<td><a href="../2-environments"><span style="white-space: nowrap;">2-environments</span></a></td>
<td>생성한 Google Cloud 조직 내에서 개발, 비프로덕션, 프로덕션 환경을 설정합니다.</td>
</tr>
<tr>
<td><a href="../3-networks-svpc">3-networks-svpc</a></td>
<td>기본 DNS, NAT(선택사항), Private Service 네트워킹, VPC 서비스 제어, 온프레미스 Dedicated
Interconnect, 각 환경에 대한 기준 방화벽 규칙이 있는 공유 VPC를 설정합니다. 또한
글로벌 DNS 허브를 설정합니다.</td>
</tr>
<tr>
<td><a href="../3-networks-hub-and-spoke">3-networks-hub-and-spoke</a></td>
<td>3-networks-svpc 단계에서 찾을 수 있는 모든 기본 구성으로 공유 VPC를 설정하지만,
여기서는 아키텍처가 허브 앤 스포크 네트워크 모델을 기반으로 합니다. 또한 글로벌 DNS 허브를 설정합니다</td>
</tr>
</tr>
<tr>
<td>4-projects (이 파일)</td>
<td>이전 단계에서 생성된 공유 VPC에 서비스 프로젝트로 연결되는 애플리케이션을 위한
폴더 구조, 프로젝트 및 애플리케이션 인프라 파이프라인을 설정합니다.</td>
</tr>
<tr>
<td><a href="../5-app-infra">5-app-infra</a></td>
<td>4-projects에서 설정한 인프라 파이프라인을 사용하여 비즈니스 유닛 프로젝트 중 하나에
간단한 <a href="https://cloud.google.com/compute/">Compute Engine</a> 인스턴스를 배포합니다.</td>
</tr>
</tbody>
</table>

아키텍처와 부분에 대한 개요는
[terraform-example-foundation README](https://github.com/terraform-google-modules/terraform-example-foundation)를 참조하세요.

## 목적

이 단계의 목적은 이전 단계에서 생성된 공유 VPC에 서비스 프로젝트로 연결되는 애플리케이션을 위한 폴더 구조, 프로젝트 및 인프라 파이프라인을 설정하는 것입니다.

각 비즈니스 유닛에 대해 Cloud Build 트리거, 애플리케이션 인프라 코드용 CSR, 상태 저장을 위한 Google Cloud Storage 버킷과 함께 공유 `infra-pipeline` 프로젝트가 생성됩니다.

이 단계는 [0-bootstrap](https://github.com/terraform-google-modules/terraform-example-foundation/blob/master/0-bootstrap/README.md)에서 배포된 Foundation 파이프라인과 동일한 [규칙](https://github.com/terraform-google-modules/terraform-example-foundation#branching-strategy)을 따릅니다.
이 파이프라인에 의해 사용자 정의 [워크스페이스](https://github.com/terraform-google-modules/terraform-google-bootstrap/blob/master/modules/tf_cloudbuild_workspace/README.md) (`bu1-example-app`)가 생성되고, 이 [예시](https://github.com/terraform-google-modules/terraform-example-foundation/blob/master/4-projects/modules/base_env/example_shared_vpc_project.tf)에 표시된 대로 `sa_roles` 변수를 활성화하여 이 워크스페이스의 Terraform 서비스 계정에 필요한 역할이 부여됩니다.

이 파이프라인은 [5-app-infra](../5-app-infra/README.md) 단계에서 development/nonproduction/production 전반의 프로젝트에 리소스를 배포하는 데 사용됩니다.
필요한 경우 배포를 격리하기 위해 다른 워크스페이스도 생성할 수 있습니다.

## 전제 조건

1. 0-bootstrap이 성공적으로 실행되었습니다.
1. 1-org가 성공적으로 실행되었습니다.
1. 2-environments가 성공적으로 실행되었습니다.
1. 3-networks가 성공적으로 실행되었습니다.

1. 이 문서에서 설명하는 수동 단계에서는 빌드 파이프라인에서 사용되는 동일한 [Terraform](https://www.terraform.io/downloads.html) 버전을 사용해야 합니다.
   그렇지 않으면 Terraform 상태 스냅샷 잠금 오류가 발생할 수 있습니다.

   **참고:** Cloud Build 배포 섹션 끝의 0-bootstrap [README 참고 2](../0-bootstrap/README.md#deploying-with-cloud-build)에서 언급된 바와 같이, **projects step service account**에 대해 최소 50개의 추가 프로젝트를 요청했는지 확인하세요. 그렇지 않으면 다음 단계에서 프로젝트 할당량 초과 오류 메시지가 발생할 수 있으며, 계속하려면 문제 해결 가이드의 [이 항목](../docs/TROUBLESHOOTING.md#attempt-to-run-4-projects-step-without-enough-project-quota)에서 수정 사항을 적용해야 합니다.

### 문제 해결

이 단계에서 문제가 발생하면 [문제 해결](../docs/TROUBLESHOOTING.md)을 참조하세요.

## 사용법

**참고:** MacOS를 사용하는 경우, 관련 명령에서 `cp -RT`를 `cp -R`로 바꾸세요.
`-T` 플래그는 Linux에서는 필요하지만 MacOS에서는 문제를 일으킵니다.

### Cloud Build로 배포하기

1. `0-bootstrap` 단계의 Terraform 출력을 기반으로 `gcp-projects` 리포지토리를 복제합니다.
   `terraform-example-foundation` 폴더와 같은 수준에서 리포지토리를 복제하고, 다음 지침은 이 레이아웃을 가정합니다.
   Cloud Build 프로젝트 ID를 가져오기 위해 `0-bootstrap` 폴더에서 `terraform output cloudbuild_project_id`를 실행합니다.

   ```bash
   export CLOUD_BUILD_PROJECT_ID=$(terraform -chdir="terraform-example-foundation/0-bootstrap/" output -raw cloudbuild_project_id)
   echo ${CLOUD_BUILD_PROJECT_ID}

   gcloud source repos clone gcp-projects --project=${CLOUD_BUILD_PROJECT_ID}
   ```

1. 새로 복제된 리포지토리로 변경하고, main이 아닌 브랜치로 변경한 다음, 기반의 콘텐츠를 새 리포지토리로 복사합니다.

   ```bash
   cd gcp-projects
   git checkout -b plan

   cp -RT ../terraform-example-foundation/4-projects/ .
   cp ../terraform-example-foundation/build/cloudbuild-tf-* .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. `auto.example.tfvars` 파일들을 `auto.tfvars`로 이름을 바꿉니다.

   ```bash
   mv common.auto.example.tfvars common.auto.tfvars
   mv shared.auto.example.tfvars shared.auto.tfvars
   mv development.auto.example.tfvars development.auto.tfvars
   mv nonproduction.auto.example.tfvars nonproduction.auto.tfvars
   mv production.auto.example.tfvars production.auto.tfvars
   ```

1. `common.auto.tfvars`, `development.auto.tfvars`, `nonproduction.auto.tfvars`, `production.auto.tfvars` 파일의 값에 대한 추가 정보는 envs 폴더의 [README.md](./business_unit_1/production/README.md) 파일들을 참조하세요.
1. `shared.auto.tfvars` 파일의 값에 대한 추가 정보는 shared 폴더의 [README.md](./business_unit_1/shared/README.md) 파일들을 참조하세요.

1. `terraform output`을 사용하여 0-bootstrap 출력에서 백엔드 버킷 값을 가져옵니다.

   ```bash
   export remote_state_bucket=$(terraform -chdir="../terraform-example-foundation/0-bootstrap/" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${remote_state_bucket}"

   sed -i'' -e "s/REMOTE_STATE_BUCKET/${remote_state_bucket}/" ./common.auto.tfvars
   ```

1. (Optional) If you want additional subfolders for separate business units or entities, make additional copies of the folder `business_unit_1` and modify any values that vary across business unit like `business_code`, `business_unit`, or `subnet_ip_range`.

예를 들어, business_unit_1과 비슷한 새로운 비즈니스 유닛을 생성하려면 다음을 실행하세요:

```bash
#copy the business_unit_1 folder and it's contents to a new folder business_unit_2
cp -r  business_unit_1 business_unit_2

# search all files under the folder `business_unit_2` and replace strings for business_unit_1 with strings for business_unit_2
grep -rl bu1 business_unit_2/ | xargs sed -i 's/bu1/bu2/g'
grep -rl business_unit_1 business_unit_2/ | xargs sed -i 's/business_unit_1/business_unit_2/g'
# search subnet_ip_range 10.3.64.0 and replace for the new range 10.4.64.0
grep -rl 10.3.64.0 business_unit_2/ | xargs sed -i 's/10.3.64.0/10.4.64.0/g'
```

1. Commit changes.

   ```bash
   git add .
   git commit -m 'Initialize projects repo'
   ```

1. You need to manually plan and apply only once the `business_unit_1/shared` and `business_unit_2/shared` environments since `development`, `nonproduction`, and `production` depend on them.
1. `tf-wrapper.sh` 스크립트의 `validate` 옵션을 사용하려면, [지침](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install)을 따라 terraform-tools 구성 요소를 설치하세요.
1. `terraform output`을 사용하여 0-bootstrap 출력에서 Cloud Build 프로젝트 ID와 프로젝트 단계 Terraform 서비스 계정을 가져옵니다. 가장을 활성화하기 위해 Terraform 서비스 계정을 사용하여 환경 변수 `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT`가 설정됩니다.

   ```bash
   export CLOUD_BUILD_PROJECT_ID=$(terraform -chdir="../terraform-example-foundation/0-bootstrap/" output -raw cloudbuild_project_id)
   echo ${CLOUD_BUILD_PROJECT_ID}

   export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=$(terraform -chdir="../terraform-example-foundation/0-bootstrap/" output -raw projects_step_terraform_service_account_email)
   echo ${GOOGLE_IMPERSONATE_SERVICE_ACCOUNT}
   ```

1. Run `init` and `plan` and review output for environment shared.

   ```bash
   ./tf-wrapper.sh init shared
   ./tf-wrapper.sh plan shared
   ```

1. Run `validate` and check for violations.

   ```bash
   ./tf-wrapper.sh validate shared $(pwd)/../gcp-policies ${CLOUD_BUILD_PROJECT_ID}
   ```

1. Run `apply` shared.

   ```bash
   ./tf-wrapper.sh apply shared
   ```

1. plan 브랜치를 푸시하여 모든 환경에 대한 계획을 트리거합니다. _plan_ 브랜치는
   [명명된 환경 브랜치](../docs/FAQ.md#what-is-a-named-branch)가 아니므로, _plan_
   브랜치를 푸시하면 _terraform plan_이 트리거되지만 _terraform apply_는 트리거되지 않습니다. Cloud Build 프로젝트에서 계획 출력을 검토합니다 https://console.cloud.google.com/cloud-build/builds;region=DEFAULT_REGION?project=YOUR_CLOUD_BUILD_PROJECT_ID

   ```bash
   git push --set-upstream origin plan
   ```

1. production으로 변경사항을 병합합니다. 이것은 [명명된 환경 브랜치](../docs/FAQ.md#what-is-a-named-branch)이므로,
   이 브랜치에 푸시하면 _terraform plan_과 _terraform apply_가 모두 트리거됩니다. Cloud Build 프로젝트에서 apply 출력을 검토합니다. https://console.cloud.google.com/cloud-build/builds;region=DEFAULT_REGION?project=YOUR_CLOUD_BUILD_PROJECT_ID

   ```bash
   git checkout -b production
   git push origin production
   ```

1. After production has been applied, apply development.
1. development로 변경사항을 병합합니다. 이것은 [명명된 환경 브랜치](../docs/FAQ.md#what-is-a-named-branch)이므로,
   이 브랜치에 푸시하면 _terraform plan_과 _terraform apply_가 모두 트리거됩니다. Cloud Build 프로젝트에서 apply 출력을 검토합니다 https://console.cloud.google.com/cloud-build/builds;region=DEFAULT_REGION?project=YOUR_CLOUD_BUILD_PROJECT_ID

   ```bash
   git checkout -b development
   git push origin development
   ```

1. After development has been applied, apply nonproduction.
1. nonproduction으로 변경사항을 병합합니다. 이것은 [명명된 환경 브랜치](../docs/FAQ.md#what-is-a-named-branch)이므로,
   이 브랜치에 푸시하면 _terraform plan_과 _terraform apply_가 모두 트리거됩니다. Cloud Build 프로젝트에서 apply 출력을 검토합니다. https://console.cloud.google.com/cloud-build/builds;region=DEFAULT_REGION?project=YOUR_CLOUD_BUILD_PROJECT_ID

   ```bash
   git checkout -b nonproduction
   git push origin nonproduction
   ```

1. 다음 단계를 실행하기 전에 `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` 환경 변수를 해제합니다.

   ```bash
   unset GOOGLE_IMPERSONATE_SERVICE_ACCOUNT
   ```

1. 이제 [5-app-infra](../5-app-infra/README.md) 단계의 지침으로 이동할 수 있습니다.

### Jenkins로 배포하기

`0-bootstrap` [README-Jenkins.md](../0-bootstrap/README-Jenkins.md#deploying-step-4-projects)를 참조하세요.

### GitHub Actions로 배포하기

`0-bootstrap` [README-GitHub.md](../0-bootstrap/README-GitHub.md#deploying-step-4-projects)를 참조하세요.

### 로컬에서 Terraform 실행하기

1. 다음 지침은 `terraform-example-foundation` 폴더와 같은 수준에 있다고 가정합니다. `gcp-projects` 폴더를 생성하고 이동한 다음, 코드, Terraform 래퍼 스크립트를 복사하고 실행할 수 있도록 설정합니다.

   ```bash
   mkdir gcp-projects
   cp -R  terraform-example-foundation/4-projects/* gcp-projects/
   cp terraform-example-foundation/build/tf-wrapper.sh gcp-projects/
   cp terraform-example-foundation/.gitignore gcp-projects/
   chmod 755 ./gcp-projects/tf-wrapper.sh
   ```

1. `gcp-projects`로 이동하여 로컬에서 버전을 관리하기 위해 로컬 Git 리포지토리를 초기화합니다. 그런 다음 환경 브랜치를 생성합니다.

   ```bash
   cd gcp-projects
   git init
   git commit -m "initialize empty directory" --allow-empty
   git checkout -b shared
   git checkout -b development
   git checkout -b nonproduction
   git checkout -b production
   ```

1. `auto.example.tfvars` 파일들을 `auto.tfvars`로 이름을 바꿉니다.

   ```bash
   mv common.auto.example.tfvars common.auto.tfvars
   mv shared.auto.example.tfvars shared.auto.tfvars
   mv development.auto.example.tfvars development.auto.tfvars
   mv nonproduction.auto.example.tfvars nonproduction.auto.tfvars
   mv production.auto.example.tfvars production.auto.tfvars
   ```

1. `common.auto.tfvars`, `development.auto.tfvars`, `nonproduction.auto.tfvars`, `production.auto.tfvars` 파일의 값에 대한 추가 정보는 envs 폴더의 [README.md](./business_unit_1/production/README.md) 파일들을 참조하세요.
   `shared.auto.tfvars` 파일의 값에 대한 추가 정보는 shared 폴더의 [README.md](./business_unit_1/shared/README.md) 파일들을 참조하세요.
   `terraform output`을 사용하여 `gcp-bootstrap` 출력에서 원격 상태 버킷(이전 단계에서 사용한 백엔드 버킷) 값을 가져옵니다.

   ```bash
   export remote_state_bucket=$(terraform -chdir="../gcp-bootstrap/" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${remote_state_bucket}"

   sed -i'' -e "s/REMOTE_STATE_BUCKET/${remote_state_bucket}/" ./common.auto.tfvars
   ```

이제 `tf-wrapper.sh` 스크립트를 사용하여 각 환경(development/production/nonproduction)을 배포합니다.

`tf-wrapper.sh` 스크립트의 `validate` 옵션을 사용하려면, [지침](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install)을 따라 terraform-tools 구성 요소를 설치하세요.

1. `terraform output`을 사용하여 gcp-bootstrap 출력에서 시드 프로젝트 ID와 조직 단계 Terraform 서비스 계정을 가져옵니다. 가장을 활성화하기 위해 Terraform 서비스 계정을 사용하여 환경 변수 `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT`가 설정됩니다.

   ```bash
   export SEED_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/" output -raw seed_project_id)
   echo ${SEED_PROJECT_ID}

   export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=$(terraform -chdir="../gcp-bootstrap/" output -raw projects_step_terraform_service_account_email)
   echo ${GOOGLE_IMPERSONATE_SERVICE_ACCOUNT}
   ```

1. (Optional) If you want additional subfolders for separate business units or entities, make additional copies of the folder `business_unit_1` and modify any values that vary across business unit like `business_code`, `business_unit`, or `subnet_ip_range`.

예를 들어, business_unit_1과 비슷한 새로운 비즈니스 유닛을 생성하려면 다음을 실행하세요:

```bash
#copy the business_unit_1 folder and it's contents to a new folder business_unit_2
cp -r  business_unit_1 business_unit_2

# search all files under the folder `business_unit_2` and replace strings for business_unit_1 with strings for business_unit_2
grep -rl bu1 business_unit_2/ | xargs sed -i 's/bu1/bu2/g'
grep -rl business_unit_1 business_unit_2/ | xargs sed -i 's/business_unit_1/business_unit_2/g'
# search subnet_ip_range 10.3.64.0 and replace for the new range 10.4.64.0
grep -rl 10.3.64.0 business_unit_2/ | xargs sed -i 's/10.3.64.0/10.4.64.0/g'
```

1. Checkout `shared` branch. Run `init` and `plan` and review output for environment shared.

   ```bash
   git checkout shared
   ./tf-wrapper.sh init shared
   ./tf-wrapper.sh plan shared
   ```

1. Run `validate` and check for violations.

   ```bash
   ./tf-wrapper.sh validate shared $(pwd)/../gcp-policies ${SEED_PROJECT_ID}
   ```

1. Run `apply` shared.

   ```bash
   ./tf-wrapper.sh apply shared
   git add .
   git commit -m "Initial shared commit."
   ```

1. Checkout `development` branch and merge `shared` into it. Run `init` and `plan` and review output for environment production.

   ```bash
   git checkout development
   git merge shared
   ./tf-wrapper.sh init development
   ./tf-wrapper.sh plan development
   ```

1. Run `validate` and check for violations.

   ```bash
   ./tf-wrapper.sh validate development $(pwd)/../gcp-policies ${SEED_PROJECT_ID}
   ```

1. Run `apply` development.

   ```bash
   ./tf-wrapper.sh apply development
   git add .
   git commit -m "Initial development commit."
   ```

1. Checkout `nonproduction` and merge `development` into it. Run `init` and `plan` and review output for environment nonproduction.

   ```bash
   git checkout nonproduction
   git merge development
   ./tf-wrapper.sh init nonproduction
   ./tf-wrapper.sh plan nonproduction
   ```

1. Run `validate` and check for violations.

   ```bash
   ./tf-wrapper.sh validate nonproduction $(pwd)/../gcp-policies ${SEED_PROJECT_ID}
   ```

1. Run `apply` nonproduction.

   ```bash
   ./tf-wrapper.sh apply nonproduction
   git add .
   git commit -m "Initial nonproduction commit."
   ```

1. Checkout shared `production`. Run `init` and `plan` and review output for environment development.

   ```bash
   git checkout production
   git merge nonproduction
   ./tf-wrapper.sh init production
   ./tf-wrapper.sh plan production
   ```

1. Run `validate` and check for violations.

   ```bash
   ./tf-wrapper.sh validate production $(pwd)/../gcp-policies ${SEED_PROJECT_ID}
   ```

1. Run `apply` production.

   ```bash
   ./tf-wrapper.sh apply production
   git add .
   git commit -m "Initial production commit."
   cd ../
   ```

오류가 발생하거나 Terraform 구성 또는 `.tfvars`에 변경사항을 적용한 경우, `./tf-wrapper.sh apply <env>`를 실행하기 전에 `./tf-wrapper.sh plan <env>`를 다시 실행해야 합니다.

다음 단계를 실행하기 전에 `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` 환경 변수를 해제합니다.

```bash
unset GOOGLE_IMPERSONATE_SERVICE_ACCOUNT
```
