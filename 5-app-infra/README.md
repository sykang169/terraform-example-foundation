# 5-app-infra

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
<td>최상위 수준 공유 폴더, 네트워킹 프로젝트,
조직 수준 로깅, 조직 정책을 통한 기준선 보안 설정을 구성합니다.</td>
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
<tr>
<td><a href="../4-projects">4-projects</a></td>
<td>이전 단계에서 생성된 공유 VPC에 서비스 프로젝트로 연결되는 애플리케이션을 위한
폴더 구조, 프로젝트 및 애플리케이션 인프라 파이프라인을 설정합니다.</td>
</tr>
<tr>
<td>5-app-infra (이 파일)</td>
<td>4-projects에서 설정한 인프라 파이프라인을 사용하여 비즈니스 유닛 프로젝트 중 하나에 간단한 [Compute Engine](https://cloud.google.com/compute/) 인스턴스를 배포합니다.</td>
</tr>
</tbody>
</table>

아키텍처와 부분에 대한 개요는
[terraform-example-foundation README](https://github.com/terraform-google-modules/terraform-example-foundation)
파일을 참조하세요.

## 목적

이 단계의 목적은 4-projects에서 설정한 인프라 파이프라인을 사용하여 비즈니스 유닛 프로젝트 중 하나에 간단한 [Compute Engine](https://cloud.google.com/compute/) 인스턴스를 배포하는 것입니다.
인프라 파이프라인은 공유 환경 내의 `4-projects` 단계에서 생성되며, 프로젝트 내의 인프라를 관리하도록 구성된 [Cloud Build](https://cloud.google.com/build/docs) 파이프라인을 보유합니다.

`0-bootstrap`에서 설정한 [CI/CD 파이프라인](https://github.com/terraform-google-modules/terraform-example-foundation#0-bootstrap)과 유사한 빌드 트리거로 구성된 [Source Repository](https://cloud.google.com/source-repositories)도 있습니다.
이 Compute Engine 인스턴스는 `3-networks` 단계의 기본 네트워크를 사용하여 생성되며 비공개 서비스에 액세스하는 데 사용됩니다.

## 전제 조건

1. 0-bootstrap이 성공적으로 실행되었습니다.
1. 1-org가 성공적으로 실행되었습니다.
1. 2-environments가 성공적으로 실행되었습니다.
1. 3-networks가 성공적으로 실행되었습니다.
1. 4-projects가 성공적으로 실행되었습니다.

### 문제 해결

이 단계에서 문제가 발생하면 [문제 해결](../docs/TROUBLESHOOTING.md)을 참조하세요.

## 사용법

**참고:** MacOS를 사용하는 경우, 관련 명령에서 `cp -RT`를 `cp -R`로 바꾸세요.
`-T` 플래그는 Linux에서는 필요하지만 MacOS에서는 문제를 일으킵니다.

### Cloud Build로 배포하기

1. `0-bootstrap` 단계의 Terraform 출력을 기반으로 `gcp-policies` 리포지토리를 복제합니다.
`terraform-example-foundation` 폴더와 같은 수준에서 리포를 복제합니다. 다음 지침은 이 레이아웃을 가정합니다.
`0-bootstrap` 폴더에서 `terraform output cloudbuild_project_id`를 실행하여 Cloud Build 프로젝트 ID를 가져옵니다.

   ```bash
   export INFRA_PIPELINE_PROJECT_ID=$(terraform -chdir="gcp-projects/business_unit_1/shared/" output -raw cloudbuild_project_id)
   echo ${INFRA_PIPELINE_PROJECT_ID}

   gcloud source repos clone gcp-policies gcp-policies-app-infra --project=${INFRA_PIPELINE_PROJECT_ID}
   ```

   **참고:** `gcp-policies` 리포지토리는 `1-org` 단계에서 생성된 리포지토리와 같은 이름을 가지고 있습니다. 충돌을 방지하기 위해 이전 명령은 이 리포를 `gcp-policies-app-infra` 폴더에 복제합니다.

1. 리포지토리로 이동하고 policy-library의 콘텐츠를 새 리포지토리로 복사합니다. 이후의 모든 단계는
   gcp-policies-app-infra 디렉토리에서 실행한다고 가정합니다. 다른 디렉토리에서 실행하는 경우,
   복사 경로를 적절히 조정하세요.

   ```bash
   cd gcp-policies-app-infra
   git checkout -b main

   cp -RT ../terraform-example-foundation/policy-library/ .
   ```

1. 변경사항을 커밋하고 main 브랜치를 새 리포지토리에 푸시합니다.

   ```bash
   git add .
   git commit -m 'Initialize policy library repo'

   git push --set-upstream origin main
   ```

1. 리포지토리에서 나갑니다.

   ```bash
   cd ..
   ```

1. `bu1-example-app` 리포지토리를 복제합니다.

   ```bash
   gcloud source repos clone bu1-example-app --project=${INFRA_PIPELINE_PROJECT_ID}
   ```

1. 리포지토리로 이동하고, 메인이 아닌 브랜치로 변경한 다음, 기반 콘텐츠를 새 리포지토리로 복사합니다.
   이후의 모든 단계는 bu1-example-app 디렉토리에서 실행한다고 가정합니다.
   다른 디렉토리에서 실행하는 경우, 복사 경로를 적절히 조정하세요.

   ```bash
   cd bu1-example-app
   git checkout -b plan

   cp -RT ../terraform-example-foundation/5-app-infra/ .
   cp ../terraform-example-foundation/build/cloudbuild-tf-* .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. `common.auto.example.tfvars`를 `common.auto.tfvars`로 이름을 변경합니다.

   ```bash
   mv common.auto.example.tfvars common.auto.tfvars
   ```

1. 환경과 0-bootstrap의 값으로 파일을 업데이트합니다. `common.auto.tfvars` 파일의 값에 대한 추가 정보는 business unit 1 envs 폴더의 [README.md](./business_unit_1/production/README.md) 파일을 참조하세요.

   ```bash
   export remote_state_bucket=$(terraform -chdir="../terraform-example-foundation/0-bootstrap/" output -raw projects_gcs_bucket_tfstate)
   echo "remote_state_bucket = ${remote_state_bucket}"
   sed -i'' -e "s/REMOTE_STATE_BUCKET/${remote_state_bucket}/" ./common.auto.tfvars
   ```

1. 변경사항을 커밋합니다.

   ```bash
   git add .
   git commit -m 'Initialize bu1 example app repo'
   ```

1. plan 브랜치를 푸시하여 모든 환경에 대한 계획을 트리거합니다. _plan_ 브랜치는
   [명명된 환경 브랜치](../docs/FAQ.md#what-is-a-named-branch)가 아니므로, _plan_
   브랜치를 푸시하면 _terraform plan_이 트리거되지만 _terraform apply_는 트리거되지 않습니다. Cloud Build 프로젝트에서 계획 출력을 검토합니다 https://console.cloud.google.com/cloud-build/builds;region=DEFAULT_REGION?project=YOUR_INFRA_PIPELINE_PROJECT_ID

   ```bash
   git push --set-upstream origin plan
   ```

1. development로 변경사항을 병합합니다. 이것은 [명명된 환경 브랜치](../docs/FAQ.md#what-is-a-named-branch)이므로,
   이 브랜치에 푸시하면 _terraform plan_과 _terraform apply_가 모두 트리거됩니다. Cloud Build 프로젝트에서 apply 출력을 검토합니다 https://console.cloud.google.com/cloud-build/builds;region=DEFAULT_REGION?project=YOUR_INFRA_PIPELINE_PROJECT_ID

   ```bash
   git checkout -b development
   git push origin development
   ```

1. nonproduction으로 변경사항을 병합합니다. 이것은 [명명된 환경 브랜치](../docs/FAQ.md#what-is-a-named-branch)이므로,
   이 브랜치에 푸시하면 _terraform plan_과 _terraform apply_가 모두 트리거됩니다. Cloud Build 프로젝트에서 apply 출력을 검토합니다 https://console.cloud.google.com/cloud-build/builds;region=DEFAULT_REGION?project=YOUR_INFRA_PIPELINE_PROJECT_ID

   ```bash
   git checkout -b nonproduction
   git push origin nonproduction
   ```

1. production 브랜치로 변경사항을 병합합니다. 이것은 [명명된 환경 브랜치](../docs/FAQ.md#what-is-a-named-branch)이므로,
      이 브랜치에 푸시하면 _terraform plan_과 _terraform apply_가 모두 트리거됩니다. Cloud Build 프로젝트에서 apply 출력을 검토합니다 https://console.cloud.google.com/cloud-build/builds;region=DEFAULT_REGION?project=YOUR_INFRA_PIPELINE_PROJECT_ID

   ```bash
   git checkout -b production
   git push origin production
   ```

### 로컬에서 Terraform 실행하기

1. 다음 지침은 `terraform-example-foundation` 폴더와 같은 수준에 있다고 가정합니다. `5-app-infra` 폴더로 이동하고, Terraform 래퍼 스크립트를 복사한 다음, 실행 가능하도록 설정합니다.

   ```bash
   cd terraform-example-foundation/5-app-infra
   cp ../build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. `common.auto.example.tfvars` 파일을 `common.auto.tfvars`로 이름을 변경합니다.

   ```bash
   mv common.auto.example.tfvars common.auto.tfvars
   ```

1. `common.auto.tfvars` 파일을 환경의 값으로 업데이트합니다.
1. `terraform output`을 사용하여 0-bootstrap에서 프로젝트 백엔드 버킷 값을 가져옵니다.

   ```bash
   export remote_state_bucket=$(terraform -chdir="../0-bootstrap/" output -raw projects_gcs_bucket_tfstate)
   echo "remote_state_bucket = ${remote_state_bucket}"
   sed -i'' -e "s/REMOTE_STATE_BUCKET/${remote_state_bucket}/" ./common.auto.tfvars
   ```

1. `./tf-wrapper.sh`를 실행할 사용자에게 bu1 Terraform 서비스 계정에 대한 Service Account Token Creator 역할을 제공합니다.
1. 사용자에게 `serviceAccountTokenCreator` 권한으로 로컬에서 terraform을 실행할 수 있는 권한을 제공합니다.

   ```bash
   member="user:$(gcloud auth list --filter="status=ACTIVE" --format="value(account)")"
   echo ${member}

   project_id=$(terraform -chdir="../4-projects/business_unit_1/shared/" output -raw cloudbuild_project_id)
   echo ${project_id}

   terraform_sa=$(terraform -chdir="../4-projects/business_unit_1/shared/" output -json terraform_service_accounts | jq '."bu1-example-app"' --raw-output)
   echo ${terraform_sa}

   gcloud iam service-accounts add-iam-policy-binding ${terraform_sa} --project ${project_id} --member="${member}" --role="roles/iam.serviceAccountTokenCreator"
   ```

1. 인프라 파이프라인 출력에서 버킷 정보로 `backend.tf`를 업데이트합니다.

   ```bash
   export backend_bucket=$(terraform -chdir="../4-projects/business_unit_1/shared/" output -json state_buckets | jq '."bu1-example-app"' --raw-output)
   echo "backend_bucket = ${backend_bucket}"

   for i in `find . -name 'backend.tf'`; do sed -i'' -e "s/UPDATE_APP_INFRA_BUCKET/${backend_bucket}/" $i; done
   ```

이제 이 스크립트를 사용하여 각 환경(development/production/nonproduction)을 배포합니다.
CI/CD 도구로 Cloud Build나 Jenkins를 사용할 때, 각 환경은 `5-app-infra` 단계의 리포지토리에 있는 브랜치에 대응됩니다. 해당하는 환경만 적용됩니다.

`tf-wrapper.sh` 스크립트의 `validate` 옵션을 사용하려면, [지침](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install)을 따라 terraform-tools 구성 요소를 설치하세요.

1. `terraform output`을 사용하여 4-projects 출력에서 인프라 파이프라인 프로젝트 ID를 가져옵니다.

   ```bash
   export INFRA_PIPELINE_PROJECT_ID=$(terraform -chdir="../4-projects/business_unit_1/shared/" output -raw cloudbuild_project_id)
   echo ${INFRA_PIPELINE_PROJECT_ID}

   export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=$(terraform -chdir="../4-projects/business_unit_1/shared/" output -json terraform_service_accounts | jq '."bu1-example-app"' --raw-output)
   echo ${GOOGLE_IMPERSONATE_SERVICE_ACCOUNT}
   ```

1. production 환경에 대해 `init` 및 `plan`을 실행하고 출력을 검토합니다.

   ```bash
   ./tf-wrapper.sh init production
   ./tf-wrapper.sh plan production
   ```

1. `validate`를 실행하고 위반 사항을 확인합니다.

   ```bash
   ./tf-wrapper.sh validate production $(pwd)/../policy-library ${INFRA_PIPELINE_PROJECT_ID}
   ```

1. production에 `apply`를 실행합니다.

   ```bash
   ./tf-wrapper.sh apply production
   ```

1. nonproduction 환경에 대해 `init` 및 `plan`을 실행하고 출력을 검토합니다.

   ```bash
   ./tf-wrapper.sh init nonproduction
   ./tf-wrapper.sh plan nonproduction
   ```

1. `validate`를 실행하고 위반 사항을 확인합니다.

   ```bash
   ./tf-wrapper.sh validate nonproduction $(pwd)/../policy-library ${INFRA_PIPELINE_PROJECT_ID}
   ```

1. nonproduction에 `apply`를 실행합니다.

   ```bash
   ./tf-wrapper.sh apply nonproduction
   ```

1. development 환경에 대해 `init` 및 `plan`을 실행하고 출력을 검토합니다.

   ```bash
   ./tf-wrapper.sh init development
   ./tf-wrapper.sh plan development
   ```

1. `validate`를 실행하고 위반 사항을 확인합니다.

   ```bash
   ./tf-wrapper.sh validate development $(pwd)/../policy-library ${INFRA_PIPELINE_PROJECT_ID}
   ```

1. development에 `apply`를 실행합니다.

   ```bash
   ./tf-wrapper.sh apply development
   ```

오류가 발생하거나 Terraform 구성 또는 `common.auto.tfvars`에 변경사항을 적용한 경우, `./tf-wrapper.sh apply <env>`를 실행하기 전에 `./tf-wrapper.sh plan <env>`를 다시 실행해야 합니다.

이 단계를 실행한 후 `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` 환경 변수를 해제합니다.

```bash
unset GOOGLE_IMPERSONATE_SERVICE_ACCOUNT
```
