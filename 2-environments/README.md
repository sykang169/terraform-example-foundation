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

## Usage

**Note:** If you are using MacOS, replace `cp -RT` with `cp -R` in the relevant
commands. The `-T` flag is needed for Linux, but causes problems for MacOS.

### Deploying with Cloud Build

1. Clone the `gcp-environments` repo based on the Terraform output from the `0-bootstrap` step.
Clone the repo at the same level of the `terraform-example-foundation` folder, the following instructions assume this layout.
Run `terraform output cloudbuild_project_id` in the `0-bootstrap` folder to get the Cloud Build Project ID.

   ```bash
   export CLOUD_BUILD_PROJECT_ID=$(terraform -chdir="terraform-example-foundation/0-bootstrap/" output -raw cloudbuild_project_id)
   echo ${CLOUD_BUILD_PROJECT_ID}

   gcloud source repos clone gcp-environments --project=${CLOUD_BUILD_PROJECT_ID}
   ```

1. Navigate into the repo, change to the non-main branch and copy contents of foundation to new repo.
   All subsequent steps assume you are running them from the gcp-environments directory.
   If you run them from another directory, adjust your copy paths accordingly.

   ```bash
   cd gcp-environments
   git checkout -b plan

   cp -RT ../terraform-example-foundation/2-environments/ .
   cp ../terraform-example-foundation/build/cloudbuild-tf-* .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. Rename `terraform.example.tfvars` to `terraform.tfvars`.

   ```bash
   mv terraform.example.tfvars terraform.tfvars
   ```

1. Update the file with values from your environment and bootstrap (you can re-run `terraform output` in the 0-bootstrap directory to find these values). See any of the envs folder [README.md](./envs/production/README.md#inputs) files for additional information on the values in the `terraform.tfvars` file.

   ```bash
   export backend_bucket=$(terraform -chdir="../terraform-example-foundation/0-bootstrap/" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${backend_bucket}"

   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" terraform.tfvars
   ```

1. Commit changes.

   ```bash
   git add .
   git commit -m 'Initialize environments repo'
   ```

1. Push your plan branch to trigger a plan for all environments. Because the
   _plan_ branch is not a [named environment branch](../docs/FAQ.md#what-is-a-named-branch), pushing your _plan_
   branch triggers _terraform plan_ but not _terraform apply_.

   ```bash
   git push --set-upstream origin plan
   ```

1. Review the plan output in your cloud build project https://console.cloud.google.com/cloud-build/builds;region=DEFAULT_REGION?project=YOUR_CLOUD_BUILD_PROJECT_ID
1. Merge changes to development branch. Because this is a [named environment branch](../docs/FAQ.md#what-is-a-named-branch),
   pushing to this branch triggers both _terraform plan_ and _terraform apply_.

   ```bash
   git checkout -b development
   git push origin development
   ```

1. Review the apply output in your cloud build project https://console.cloud.google.com/cloud-build/builds;region=DEFAULT_REGION?project=YOUR_CLOUD_BUILD_PROJECT_ID
1. Merge changes to nonproduction. Because this is a [named environment branch](../docs/FAQ.md#what-is-a-named-branch),
   pushing to this branch triggers both _terraform plan_ and _terraform apply_. Review the apply output in your cloud build project https://console.cloud.google.com/cloud-build/builds;region=DEFAULT_REGION?project=YOUR_CLOUD_BUILD_PROJECT_ID

   ```bash
   git checkout -b nonproduction
   git push origin nonproduction
   ```

1. Merge changes to production branch. Because this is a [named environment branch](../docs/FAQ.md#what-is-a-named-branch),
   pushing to this branch triggers both _terraform plan_ and _terraform apply_. Review the apply output in your cloud build project https://console.cloud.google.com/cloud-build/builds;region=DEFAULT_REGION?project=YOUR_CLOUD_BUILD_PROJECT_ID

   ```bash
   git checkout -b production
   git push origin production
   ```

1. You can now move to the instructions in the network step. To use the [Dual Shared VPC](https://cloud.google.com/architecture/security-foundations/networking#vpcsharedvpc-id7-1-shared-vpc-) network mode go to [3-networks-svpc](../3-networks-svpc/README.md), or go to [3-networks-hub-and-spoke](../3-networks-hub-and-spoke/README.md) to use the [Hub and Spoke](https://cloud.google.com/architecture/security-foundations/networking#hub-and-spoke) network mode.

### Deploying with Jenkins

See `0-bootstrap` [README-Jenkins.md](../0-bootstrap/README-Jenkins.md#deploying-step-2-environments).

### Deploying with GitHub Actions

See `0-bootstrap` [README-GitHub.md](../0-bootstrap/README-GitHub.md#deploying-step-2-environments).

### Run Terraform locally

1. The next instructions assume that you are at the same level of the `terraform-example-foundation` folder. Create the `gcp-environments` folder, copy the Terraform wrapper script and ensure it can be executed.

   ```bash
   mkdir gcp-environments
   cp -R terraform-example-foundation/2-environments/* gcp-environments/
   cp terraform-example-foundation/build/tf-wrapper.sh gcp-environments/
   cp terraform-example-foundation/.gitignore gcp-environments/
   chmod 755 ./gcp-environments/tf-wrapper.sh
   ```

1. Navigate to `gcp-environments` and initialize a local Git repository to manage versions locally. Then, create the environment branches.

   ```bash
   cd gcp-environments
   git init
   git commit -m "initialize empty directory" --allow-empty
   git checkout -b production
   git checkout -b nonproduction
   git checkout -b development
   ```

1. Rename `terraform.example.tfvars` to `terraform.tfvars`.

   ```bash
   mv terraform.example.tfvars terraform.tfvars
   ```

1. Update the file with values from your environment and 0-bootstrap output.See any of the envs folder [README.md](./envs/production/README.md#inputs) files for additional information on the values in the `terraform.tfvars` file.
1. Use `terraform output` to get the backend bucket value from 0-bootstrap output.

   ```bash
   export backend_bucket=$(terraform -chdir="../gcp-bootstrap/" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${backend_bucket}"

   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" ./terraform.tfvars
   ```

We will now deploy each of our environments(development/production/nonproduction) using this script.

To use the `validate` option of the `tf-wrapper.sh` script, please follow the [instructions](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install) to install the terraform-tools component.

1. Use `terraform output` to get the Seed project ID and the organization step Terraform service account from gcp-bootstrap output. An environment variable `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` will be set using the Terraform Service Account to enable impersonation.

   ```bash
   export SEED_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/" output -raw seed_project_id)
   echo ${SEED_PROJECT_ID}

   export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=$(terraform -chdir="../gcp-bootstrap/" output -raw environment_step_terraform_service_account_email)
   echo ${GOOGLE_IMPERSONATE_SERVICE_ACCOUNT}
   ```

1. Checkout `development` branch. Run `init` and `plan` and review output for environment development.

   ```bash
   git checkout development
   ./tf-wrapper.sh init development
   ./tf-wrapper.sh plan development
   ```

1. Run `validate` and check for violations.

   ```bash
   ./tf-wrapper.sh validate development $(pwd)/../gcp-policies ${SEED_PROJECT_ID}
   ```

1. Run `apply` development and commit the initial version of `development` branch.

   ```bash
   ./tf-wrapper.sh apply development
   git add .
   git commit -m "Development initial commit."
   ```

1. Checkout `nonproduction` branch and merge `development` branch into it. Run `init` and `plan` and review output for environment nonproduction.

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

1. Run `apply` production and commit initial version of nonproduction.

   ```bash
   ./tf-wrapper.sh apply nonproduction
   git add .
   git commit -m "Nonproduction initial commit."
   ```

1. Checkout `production` branch and merge `nonproduction` branch into it. Run `init` and `plan` and review output for environment production.

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

1. Run `apply` production and commit initial version of production.

   ```bash
   ./tf-wrapper.sh apply production
   git add .
   git commit -m "Production initial commit."
   ```

If you received any errors or made any changes to the Terraform config or `terraform.tfvars` you must re-run `./tf-wrapper.sh plan <env>` before running `./tf-wrapper.sh apply <env>`.

Before executing the next stages, unset the `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` environment variable.

```bash
unset GOOGLE_IMPERSONATE_SERVICE_ACCOUNT
```
