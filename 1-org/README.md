# 1-org

이 리포지토리는 [Google Cloud 보안 기반 가이드](https://cloud.google.com/architecture/security-foundations)에서 설명된
example.com 참조 아키텍처를 구성하고 배포하는 방법을 보여주는 다중 파트 가이드의 일부입니다. 다음 표는 가이드의 부분들을 나열합니다.

<table>
<tbody>
<tr>
<td><a href="../0-bootstrap">0-bootstrap</a></td>
<td>Google Cloud 조직을 부트스트래핑하여 Cloud Foundation Toolkit(CFT) 사용을 시작하는 데 필요한 모든 리소스와
권한을 생성합니다. 이 단계는 또한 후속 단계에서 기반 코드를 위한 <a href="../docs/GLOSSARY.md#foundation-cicd-pipeline">CI/CD 파이프라인</a>을 구성합니다.</td>
</tr>
<tr>
<td>1-org (이 파일)</td>
<td>최상위 수준 공유 폴더, 네트워킹 프로젝트 및 조직 수준 로깅을 설정하고,
조직 정책을 통해 기준선 보안 설정을 구성합니다.</td>
</tr>
<tr>
<td><a href="../2-environments"><span style="white-space: nowrap;">2-environments</span></a></td>
<td>생성한 Google Cloud 조직 내에서 개발, 비프로덕션, 프로덕션 환경을 설정합니다.</td>
</tr>
<tr>
<td><a href="../3-networks-svpc">3-networks-svpc</a></td>
<td>기본 DNS, NAT(선택사항), Private Service 네트워킹, VPC 서비스 제어, 온프레미스 Dedicated Interconnect, 
각 환경에 대한 기준 방화벽 규칙이 있는 공유 VPC를 설정합니다. 또한 글로벌 DNS 허브를 설정합니다.</td>
</tr>
<tr>
<td><a href="../3-networks-hub-and-spoke">3-networks-hub-and-spoke</a></td>
<td>3-networks-svpc 단계에서 찾을 수 있는 모든 기본 구성으로 공유 VPC를 설정하지만, 
여기서는 아키텍처가 허브-앤-스포크 네트워크 모델을 기반으로 합니다. 또한 글로벌 DNS 허브를 설정합니다.</td>
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
<a href="https://cloud.google.com/compute/">Compute Engine</a> 인스턴스를 배포합니다.</td>
</tr>
</tbody>
</table>

아키텍처와 부분에 대한 개요는 
[terraform-example-foundation README](https://github.com/terraform-google-modules/terraform-example-foundation)를 참조하세요.

## 목적

이 단계의 목적은 최상위 수준 공유 폴더, 네트워킹 프로젝트, 조직 수준 로깅, 그리고 조직 정책을 통한 기준선 보안 설정을 구성하는 것입니다.

## 사전 준비사항

1. 0-bootstrap을 실행합니다.
1. Security Command Center 알림을 활성화하려면, Security Command Center 계층을 선택하고 [Security Command Center 설정](https://cloud.google.com/security-command-center/docs/quickstart-security-command-center)에 설명된 대로 Security Command Center 서비스 계정에 대한 권한을 생성하고 부여합니다.

### 문제 해결

이 단계에서 문제가 발생하면 [문제 해결](../docs/TROUBLESHOOTING.md)을 참조하세요.

## 사용법

다음을 고려하세요:

- 이 모듈은 모든 로그를 Cloud Logging 버킷으로 내보내는 싱크를 생성합니다. 또한 보안 관련 로그의 하위 집합을
Bigquery와 Pub/Sub로 내보내는 싱크도 생성합니다. 이로 인해 해당 로그 사본에 대해 추가 요금이 발생합니다. 로그 버킷 대상의 경우, 기본 보존 기간(30일) 동안 보관된 로그는 [스토리지 비용이 발생하지 않습니다](https://cloud.google.com/stackdriver/pricing#:~:text=Logs%20retained%20for%20the%20default%20retention%20period%20don%27t%20incur%20a%20storage%20cost.).
  `envs/shared/log_sinks.tf`의 구성을 수정하여 필터와 싱크를 변경할 수 있습니다.

- 이 모듈은 조직 로그에 대한 [버킷 정책 보존](https://cloud.google.com/storage/docs/bucket-lock)을 구현하지만 활성화하지는 않습니다. 필요한 경우 `log_export_storage_retention_policy` 변수를 구성하여 보존 정책을 활성화하세요.

- 이 모듈은 조직 로그에 대한 [객체 버전 관리](https://cloud.google.com/storage/docs/object-versioning)를 구현하지만 활성화하지는 않습니다. 필요한 경우 `log_export_storage_versioning` 변수를 true로 설정하여 객체 버전 관리를 활성화하세요.

- 버킷 정책 보존과 객체 버전 관리는 **상호 배타적**입니다.

- [Google Cloud 보안 기반 가이드](https://cloud.google.com/architecture/security-foundations/networking#hub-and-spoke)의 **네트워킹** 섹션에서 설명된 **허브-앤-스포크** 아키텍처를 사용하려면 `enable_hub_and_spoke` 변수를 `true`로 설정하세요.

- MacOS를 사용하는 경우, 관련 명령에서 `cp -RT`를 `cp -R`로 바꾸세요. 
`-T` 플래그는 Linux에서는 필요하지만 MacOS에서는 문제를 일으킵니다.

- 이 모듈은 [Essential Contacts](https://cloud.google.com/resource-manager/docs/managing-notification-contacts)를 사용하여 알림용 연락처를 관리합니다. Essential Contacts는 모든 하위 리소스에 상속되도록 구성한 상위(조직 또는 폴더)에 할당됩니다. project-factory [essential_contacts 하위모듈](https://registry.terraform.io/modules/terraform-google-modules/project-factory/google/13.1.0/submodules/essential_contacts#example-usage)을 사용하여 프로젝트에 Essential Contacts를 직접 할당할 수도 있습니다. 청구 알림은 `group_billing_admins` 필수 그룹으로 전송됩니다. 법적 및 정지 알림은 `group_org_admins` 필수 그룹으로 전송됩니다. 다른 모든 그룹을 제공하는 경우, 다음 표에 설명된 대로 알림이 구성됩니다.

| 그룹 | 알림 카테고리 | 대체 그룹 |
|-------|-----------------------|----------------|
| gcp_network_viewer | 기술적 | 조직 관리자 |
| gcp_platform_viewer | 제품 업데이트 및 기술적 | 조직 관리자 |
| gcp_scc_admin | 제품 업데이트 및 보안 | 조직 관리자 |
| gcp_security_reviewer | 보안 및 기술적 | 조직 관리자 |

이 모듈은 공통, 네트워크 및 부트스트랩 폴더에 [태그](https://cloud.google.com/resource-manager/docs/tags/tags-overview)를 생성하고 적용합니다. 이러한 태그는 [2-environments](../2-environments/README.md) 단계의 환경 폴더에도 적용됩니다. `tags.tf`에서 `local.tags` 맵을 편집하고 주석 처리된 템플릿을 따라 고유한 태그를 생성할 수 있습니다. 다음 표는 리소스에 적용되는 태그에 대한 세부 정보를 설명합니다:

| 리소스 | 유형 | 단계 | 태그 키 | 태그 값 |
|----------|------|------|---------|-----------|
| bootstrap | folder | 1-org | environment | bootstrap |
| common | folder | 1-org | environment | production |
| network | folder | 1-org | environment | production |
| enviroment development | folder | [2-environments](../2-environments/README.md) | environment | development |
| enviroment nonproduction | folder | [2-environments](../2-environments/README.md) | environment | nonproduction |
| enviroment production | folder | [2-environments](../2-environments/README.md) | environment | production |

### Cloud Build로 배포하기

1. `0-bootstrap` 단계의 Terraform 출력을 기반으로 `gcp-org` 리포지토리를 복제합니다.
`terraform-example-foundation` 폴더와 같은 수준에서 리포를 복제합니다.
필요한 경우 `0-bootstrap` 폴더에서 `terraform output cloudbuild_project_id`를 실행하여 Cloud Build 프로젝트 ID를 가져옵니다.

   ```bash
   export CLOUD_BUILD_PROJECT_ID=$(terraform -chdir="terraform-example-foundation/0-bootstrap/" output -raw cloudbuild_project_id)
   echo ${CLOUD_BUILD_PROJECT_ID}

   gcloud source repos clone gcp-org --project=${CLOUD_BUILD_PROJECT_ID}
   ```

   **참고:** `warning: You appear to have cloned an empty repository.` 메시지는
   정상이며 무시할 수 있습니다.

1. 리포지토리로 이동하고, 비프로덕션 브랜치로 변경한 다음, 기반 콘텐츠를 새 리포지토리로 복사합니다.
   이후의 모든 단계는 `gcp-org` 디렉토리에서 실행한다고 가정합니다.
   다른 디렉토리에서 실행하는 경우, 복사 경로를 적절히 조정하세요.

   ```bash
   cd gcp-org
   git checkout -b plan

   cp -RT ../terraform-example-foundation/1-org/ .
   cp ../terraform-example-foundation/build/cloudbuild-tf-* .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. `./envs/shared/terraform.example.tfvars`를 `./envs/shared/terraform.tfvars`로 이름을 바꿉니다.

   ```bash
   mv ./envs/shared/terraform.example.tfvars ./envs/shared/terraform.tfvars
   ```

1. Check if a Security Command Center notification with the default name, **scc-notify**, already exists. If it exists, choose a different value for the `scc_notification_name` variable in the `./envs/shared/terraform.tfvars` file.

   ```bash
   export ORGANIZATION_ID=$(terraform -chdir="../terraform-example-foundation/0-bootstrap/" output -json common_config | jq '.org_id' --raw-output)
   gcloud scc notifications describe "scc-notify" --organization=${ORGANIZATION_ID}
   ```

1. Check if your organization already has an Access Context Manager policy.

   ```bash
   export ACCESS_CONTEXT_MANAGER_ID=$(gcloud access-context-manager policies list --organization ${ORGANIZATION_ID} --format="value(name)")
   echo "access_context_manager_policy_id = ${ACCESS_CONTEXT_MANAGER_ID}"
   ```

1. Update the `envs/shared/terraform.tfvars` file with values from your environment and 0-bootstrap step. If the previous step showed a numeric value, un-comment the variable `create_access_context_manager_access_policy = false`. See the shared folder [README.md](./envs/shared/README.md) for additional information on the values in the `terraform.tfvars` file.

   ```bash
   export backend_bucket=$(terraform -chdir="../terraform-example-foundation/0-bootstrap/" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${backend_bucket}"

   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" ./envs/shared/terraform.tfvars

   if [ ! -z "${ACCESS_CONTEXT_MANAGER_ID}" ]; then sed -i'' -e "s=//create_access_context_manager_access_policy=create_access_context_manager_access_policy=" ./envs/shared/terraform.tfvars; fi
   ```

1. Commit changes.

   ```bash
   git add .
   git commit -m 'Initialize org repo'
   ```

1. Push your plan branch to trigger a plan for all environments. Because the
   _plan_ branch is not a [named environment branch](../docs/FAQ.md#what-is-a-named-branch), pushing your _plan_
   branch triggers _terraform plan_ but not _terraform apply_. Review the plan output in your Cloud Build project.

   ```bash
   git push --set-upstream origin plan
   ```

1. Merge changes to the production branch. Because the _production_ branch is a [named environment branch](../docs/FAQ.md#what-is-a-named-branch),
   pushing to this branch triggers both _terraform plan_ and _terraform apply_. Review the apply output in your Cloud Build project.

   ```bash
   git checkout -b production
   git push origin production
   ```

1. Proceed to the [2-environments](../2-environments/README.md) step.

**Troubleshooting:**
If you received a `PERMISSION_DENIED` error while running the `gcloud access-context-manager` or the `gcloud scc notifications` commands, you can append the following to run the command as the Terraform service account:

```bash
--impersonate-service-account=$(terraform -chdir="../terraform-example-foundation/0-bootstrap/" output -raw organization_step_terraform_service_account_email)
```

### Deploying with Jenkins

See `0-bootstrap` [README-Jenkins.md](../0-bootstrap/README-Jenkins.md#deploying-step-1-org).

### Deploying with GitHub Actions

See `0-bootstrap` [README-GitHub.md](../0-bootstrap/README-GitHub.md#deploying-step-1-org).

### Running Terraform locally

1. The next instructions assume that you are at the same level of the `terraform-example-foundation` folder.
Create `gcp-org` folder, copy `1-org` content and Terraform wrapper script; ensure it can be executed.

   ```bash
   mkdir gcp-org
   cp -R terraform-example-foundation/1-org/* gcp-org/
   cp terraform-example-foundation/build/tf-wrapper.sh gcp-org/
   cp terraform-example-foundation/.gitignore gcp-org/
   cd gcp-org
   chmod 755 ./tf-wrapper.sh
   ```

1. Initialize a local Git repository to manage versions locally. Then, create the environment branches.

   ```bash
      git init
      git commit -m "initialize empty directory" --allow-empty
      git checkout -b plan
      git checkout -b production
   ```

1. Rename `./envs/shared/terraform.example.tfvars` to `./envs/shared/terraform.tfvars`.

   ```bash
   mv ./envs/shared/terraform.example.tfvars ./envs/shared/terraform.tfvars
   ```

1. Check if a Security Command Center notification with the default name, **scc-notify**, already exists. If it exists, choose a different value for the `scc_notification_name` variable in the `./envs/shared/terraform.tfvars` file.

   ```bash
   export ORGANIZATION_ID=$(terraform -chdir="../gcp-bootstrap/" output -json common_config | jq '.org_id' --raw-output)
   gcloud scc notifications describe "scc-notify" --organization=${ORGANIZATION_ID}
   ```

1. Check if your organization already has an Access Context Manager policy.

   ```bash
   export ACCESS_CONTEXT_MANAGER_ID=$(gcloud access-context-manager policies list --organization ${ORGANIZATION_ID} --format="value(name)")
   echo "access_context_manager_policy_id = ${ACCESS_CONTEXT_MANAGER_ID}"
   ```

1. Update the `envs/shared/terraform.tfvars` file with values from your environment and `gcp-bootstrap` step. If the previous step showed a numeric value, un-comment the variable `create_access_context_manager_access_policy = false`. See the shared folder [README.md](./envs/shared/README.md) for additional information on the values in the `terraform.tfvars` file.

   ```bash
   export backend_bucket=$(terraform -chdir="../gcp-bootstrap/" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${backend_bucket}"

   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" ./envs/shared/terraform.tfvars

   if [ ! -z "${ACCESS_CONTEXT_MANAGER_ID}" ]; then sed -i'' -e "s=//create_access_context_manager_access_policy=create_access_context_manager_access_policy=" ./envs/shared/terraform.tfvars; fi
   ```

You can now deploy your environment (production) using this script.

To use the `validate` option of the `tf-wrapper.sh` script, follow the [instructions](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install) to install the terraform-tools component.

1. Use `terraform output` to get the Seed project ID and the organization step Terraform service account from gcp-bootstrap output. An environment variable `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` will be set using the Terraform Service Account to enable impersonation.

   ```bash
   export SEED_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/" output -raw seed_project_id)
   echo ${SEED_PROJECT_ID}

   export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=$(terraform -chdir="../gcp-bootstrap/" output -raw organization_step_terraform_service_account_email)
   echo ${GOOGLE_IMPERSONATE_SERVICE_ACCOUNT}
   ```

1. Run `init` and `plan` and review the output.

   ```bash
   git checkout plan
   ./tf-wrapper.sh init production
   ./tf-wrapper.sh plan production
   ```

1. Run `validate` and resolve any violations.

   ```bash
   ./tf-wrapper.sh validate production $(pwd)/../gcp-policies ${SEED_PROJECT_ID}
   ```

1. Commit validated code in plan branch.

   ```bash
   git add .
   git commit -m "Initial version of gcp-org."
   ```

1. Checkout `production` branch and merge plan into it. Run `apply production`.

   ```bash
   git checkout production
   git merge plan
   ./tf-wrapper.sh apply production
   ```

If you receive any errors or made any changes to the Terraform config or `terraform.tfvars`, re-run `./tf-wrapper.sh plan production` before you run `./tf-wrapper.sh apply production`.

Before executing the next stages, unset the `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` environment variable.

```bash
unset GOOGLE_IMPERSONATE_SERVICE_ACCOUNT
```
