# 2-environments

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
<td><span style="white-space: nowrap;">2-environments</span> (이 파일)</td>
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
여기서는 아키텍처가 허브 앤 스포크 네트워크 모델을 기반으로 합니다. 또한 글로벌 DNS 허브를 설정합니다.</td>
</tr>
</tr>
<tr>
<td><a href="../4-projects">4-projects</a></td>
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

이 단계의 목적은 생성한 Google Cloud 조직 내에서 개발, 비운영, 운영 환경을 설정하는 것입니다.

## 전제 조건

1. 0-bootstrap이 성공적으로 실행되었습니다.
1. 1-org가 성공적으로 실행되었습니다.

### 문제 해결

이 단계에서 문제가 발생하면 [문제 해결](../docs/TROUBLESHOOTING.md)을 참조하세요.

## Assured Workloads

프로덕션 폴더에서 [Assured Workloads](https://cloud.google.com/assured-workloads)를 활성화하려면 [main.tf](./envs/production/main.tf#L26) 파일을 편집하고 `assured_workload_configuration.enable`을 `true`로 업데이트하세요.

Workload에 대해 구성할 수 있는 값에 대한 추가 정보는 `env_baseline` 모듈 [README.md](./modules/env_baseline/README.md) 파일을 참조하세요.

**Assured Workload는 유료 서비스입니다.**
FedRAMP Moderate 워크로드는 Google Cloud 제품 및 서비스 사용에 대한 추가 비용 없이 배포할 수 있습니다.
다른 준수 제도에 대해서는 [Assured Workloads 요금](https://cloud.google.com/assured-workloads/pricing)을 참조하세요.

Assured Workloads를 활성화한 경우, Assured 워크로드를 삭제하려면 그 하위의 리소스를 수동으로 삭제해야 합니다.
삭제할 리소스를 식별하려면 [GCP 콘솔](https://console.cloud.google.com/compliance/assuredworkloads)을 사용하세요.

## 사용법

**참고:** MacOS를 사용하는 경우, 관련 명령에서 `cp -RT`를 `cp -R`로 바꾸세요.
`-T` 플래그는 Linux에서는 필요하지만 MacOS에서는 문제를 일으킵니다.

### Cloud Build로 배포하기

1. `0-bootstrap` 단계의 Terraform 출력을 기반으로 `gcp-environments` 리포지토리를 복제합니다.
`terraform-example-foundation` 폴더와 같은 수준에서 리포를 복제합니다. 다음 지침은 이 레이아웃을 가정합니다.
`0-bootstrap` 폴더에서 `terraform output cloudbuild_project_id`를 실행하여 Cloud Build 프로젝트 ID를 가져옵니다.

   ```bash
   export CLOUD_BUILD_PROJECT_ID=$(terraform -chdir="terraform-example-foundation/0-bootstrap/" output -raw cloudbuild_project_id)
   echo ${CLOUD_BUILD_PROJECT_ID}

   gcloud source repos clone gcp-environments --project=${CLOUD_BUILD_PROJECT_ID}
   ```

1. 리포지토리로 이동하고, 메인이 아닌 브랜치로 변경한 다음, 기반 콘텐츠를 새 리포지토리로 복사합니다.
   이후의 모든 단계는 gcp-environments 디렉토리에서 실행한다고 가정합니다.
   다른 디렉토리에서 실행하는 경우, 복사 경로를 적절히 조정하세요.

   ```bash
   cd gcp-environments
   git checkout -b plan

   cp -RT ../terraform-example-foundation/2-environments/ .
   cp ../terraform-example-foundation/build/cloudbuild-tf-* .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. `terraform.example.tfvars`를 `terraform.tfvars`로 이름을 바꿉니다.

   ```bash
   mv terraform.example.tfvars terraform.tfvars
   ```

1. 환경과 부트스트랩의 값으로 파일을 업데이트합니다(이러한 값을 찾기 위해 0-bootstrap 디렉토리에서 `terraform output`을 다시 실행할 수 있습니다). `terraform.tfvars` 파일의 값에 대한 추가 정보는 envs 폴더 [README.md](./envs/production/README.md#inputs) 파일을 참조하세요.

   ```bash
   export backend_bucket=$(terraform -chdir="../terraform-example-foundation/0-bootstrap/" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${backend_bucket}"

   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" terraform.tfvars
   ```

1. 변경사항을 커밋합니다.

   ```bash
   git add .
   git commit -m 'Initialize environments repo'
   ```

1. plan 브랜치를 푸시하여 모든 환경에 대한 계획을 트리거합니다. _plan_ 브랜치는 [명명된 환경 브랜치](../docs/FAQ.md#what-is-a-named-branch)가 아니므로, _plan_ 브랜치를 푸시하면 _terraform plan_이 트리거되지만 _terraform apply_는 트리거되지 않습니다.

   ```bash
   git push --set-upstream origin plan
   ```

1. Cloud Build 프로젝트에서 계획 출력을 검토합니다 https://console.cloud.google.com/cloud-build/builds;region=DEFAULT_REGION?project=YOUR_CLOUD_BUILD_PROJECT_ID
1. development 브랜치로 변경사항을 병합합니다. 이것은 [명명된 환경 브랜치](../docs/FAQ.md#what-is-a-named-branch)이므로, 이 브랜치에 푸시하면 _terraform plan_과 _terraform apply_가 모두 트리거됩니다.

   ```bash
   git checkout -b development
   git push origin development
   ```

1. Cloud Build 프로젝트에서 apply 출력을 검토합니다 https://console.cloud.google.com/cloud-build/builds;region=DEFAULT_REGION?project=YOUR_CLOUD_BUILD_PROJECT_ID
1. nonproduction으로 변경사항을 병합합니다. 이것은 [명명된 환경 브랜치](../docs/FAQ.md#what-is-a-named-branch)이므로, 이 브랜치에 푸시하면 _terraform plan_과 _terraform apply_가 모두 트리거됩니다. Cloud Build 프로젝트에서 apply 출력을 검토합니다 https://console.cloud.google.com/cloud-build/builds;region=DEFAULT_REGION?project=YOUR_CLOUD_BUILD_PROJECT_ID

   ```bash
   git checkout -b nonproduction
   git push origin nonproduction
   ```

1. production 브랜치로 변경사항을 병합합니다. 이것은 [명명된 환경 브랜치](../docs/FAQ.md#what-is-a-named-branch)이므로, 이 브랜치에 푸시하면 _terraform plan_과 _terraform apply_가 모두 트리거됩니다. Cloud Build 프로젝트에서 apply 출력을 검토합니다 https://console.cloud.google.com/cloud-build/builds;region=DEFAULT_REGION?project=YOUR_CLOUD_BUILD_PROJECT_ID

   ```bash
   git checkout -b production
   git push origin production
   ```

1. 이제 네트워크 단계의 지침으로 이동할 수 있습니다. [듀얼 공유 VPC](https://cloud.google.com/architecture/security-foundations/networking#vpcsharedvpc-id7-1-shared-vpc-) 네트워크 모드를 사용하려면 [3-networks-svpc](../3-networks-svpc/README.md)로 이동하고, [허브 앤 스포크](https://cloud.google.com/architecture/security-foundations/networking#hub-and-spoke) 네트워크 모드를 사용하려면 [3-networks-hub-and-spoke](../3-networks-hub-and-spoke/README.md)로 이동하세요.

### Jenkins로 배포하기

`0-bootstrap` [README-Jenkins.md](../0-bootstrap/README-Jenkins.md#deploying-step-2-environments)를 참조하세요.

### GitHub Actions로 배포하기

`0-bootstrap` [README-GitHub.md](../0-bootstrap/README-GitHub.md#deploying-step-2-environments)를 참조하세요.

### 로컬에서 Terraform 실행하기

1. 다음 지침은 `terraform-example-foundation` 폴더와 같은 수준에 있다고 가정합니다. `gcp-environments` 폴더를 생성하고, Terraform 래퍼 스크립트를 복사한 다음 실행할 수 있도록 설정합니다.

   ```bash
   mkdir gcp-environments
   cp -R terraform-example-foundation/2-environments/* gcp-environments/
   cp terraform-example-foundation/build/tf-wrapper.sh gcp-environments/
   cp terraform-example-foundation/.gitignore gcp-environments/
   chmod 755 ./gcp-environments/tf-wrapper.sh
   ```

1. `gcp-environments`로 이동하여 로컬에서 버전을 관리하기 위해 로컬 Git 리포지토리를 초기화합니다. 그런 다음 환경 브랜치를 생성합니다.

   ```bash
   cd gcp-environments
   git init
   git commit -m "initialize empty directory" --allow-empty
   git checkout -b production
   git checkout -b nonproduction
   git checkout -b development
   ```

1. `terraform.example.tfvars`를 `terraform.tfvars`로 이름을 바꿉니다.

   ```bash
   mv terraform.example.tfvars terraform.tfvars
   ```

1. 환경과 0-bootstrap 출력의 값으로 파일을 업데이트합니다. `terraform.tfvars` 파일의 값에 대한 추가 정보는 envs 폴더 [README.md](./envs/production/README.md#inputs) 파일을 참조하세요.
1. `terraform output`을 사용하여 0-bootstrap 출력에서 백엔드 버킷 값을 가져옵니다.

   ```bash
   export backend_bucket=$(terraform -chdir="../gcp-bootstrap/" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${backend_bucket}"

   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" ./terraform.tfvars
   ```

이제 이 스크립트를 사용하여 각 환경(development/production/nonproduction)을 배포합니다.

`tf-wrapper.sh` 스크립트의 `validate` 옵션을 사용하려면, [지침](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install)을 따라 terraform-tools 구성 요소를 설치하세요.

1. `terraform output`을 사용하여 gcp-bootstrap 출력에서 Seed 프로젝트 ID와 조직 단계 Terraform 서비스 계정을 가져옵니다. 가장을 활성화하기 위해 Terraform 서비스 계정을 사용하여 환경 변수 `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT`가 설정됩니다.

   ```bash
   export SEED_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/" output -raw seed_project_id)
   echo ${SEED_PROJECT_ID}

   export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=$(terraform -chdir="../gcp-bootstrap/" output -raw environment_step_terraform_service_account_email)
   echo ${GOOGLE_IMPERSONATE_SERVICE_ACCOUNT}
   ```

1. `development` 브랜치를 체크아웃합니다. `init`과 `plan`을 실행하고 development 환경에 대한 출력을 검토합니다.

   ```bash
   git checkout development
   ./tf-wrapper.sh init development
   ./tf-wrapper.sh plan development
   ```

1. `validate`를 실행하고 위반 사항을 확인합니다.

   ```bash
   ./tf-wrapper.sh validate development $(pwd)/../gcp-policies ${SEED_PROJECT_ID}
   ```

1. `apply` development를 실행하고 `development` 브랜치의 초기 버전을 커밋합니다.

   ```bash
   ./tf-wrapper.sh apply development
   git add .
   git commit -m "Development initial commit."
   ```

1. `nonproduction` 브랜치를 체크아웃하고 `development` 브랜치를 병합합니다. `init`과 `plan`을 실행하고 nonproduction 환경에 대한 출력을 검토합니다.

   ```bash
   git checkout nonproduction
   git merge development
   ./tf-wrapper.sh init nonproduction
   ./tf-wrapper.sh plan nonproduction
   ```

1. `validate`를 실행하고 위반 사항을 확인합니다.

   ```bash
   ./tf-wrapper.sh validate nonproduction $(pwd)/../gcp-policies ${SEED_PROJECT_ID}
   ```

1. `apply` nonproduction을 실행하고 nonproduction의 초기 버전을 커밋합니다.

   ```bash
   ./tf-wrapper.sh apply nonproduction
   git add .
   git commit -m "Nonproduction initial commit."
   ```

1. `production` 브랜치를 체크아웃하고 `nonproduction` 브랜치를 병합합니다. `init`과 `plan`을 실행하고 production 환경에 대한 출력을 검토합니다.

   ```bash
   git checkout production
   git merge nonproduction
   ./tf-wrapper.sh init production
   ./tf-wrapper.sh plan production
   ```

1. `validate`를 실행하고 위반 사항을 확인합니다.

   ```bash
   ./tf-wrapper.sh validate production $(pwd)/../gcp-policies ${SEED_PROJECT_ID}
   ```

1. `apply` production을 실행하고 production의 초기 버전을 커밋합니다.

   ```bash
   ./tf-wrapper.sh apply production
   git add .
   git commit -m "Production initial commit."
   ```

오류가 발생하거나 Terraform 구성 또는 `terraform.tfvars`에 변경사항을 적용한 경우, `./tf-wrapper.sh apply <env>`를 실행하기 전에 `./tf-wrapper.sh plan <env>`를 다시 실행해야 합니다.

다음 단계를 실행하기 전에 `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` 환경 변수를 해제합니다.

```bash
unset GOOGLE_IMPERSONATE_SERVICE_ACCOUNT
```
