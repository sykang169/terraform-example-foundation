# 3-networks-svpc

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
<td><a>3-networks-svpc (이 파일)</a></td>
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

이 단계의 목적은 다음과 같습니다:

- 글로벌 [DNS Hub](https://cloud.google.com/blog/products/networking/cloud-forwarding-peering-and-zones)를 설정합니다.
- 각 환경에 대해 기본 DNS, NAT(선택사항), Private Service 네트워킹, VPC Service Controls(선택사항), 온프레미스 Dedicated 또는 Partner Interconnect, 기준선 방화벽 규칙이 포함된 공유 VPC를 설정합니다.

## 전제 조건

1. 0-bootstrap이 성공적으로 실행되었습니다.
1. 1-org가 성공적으로 실행되었습니다.
1. 2-environments가 성공적으로 실행되었습니다.
1. access_context_manager_policy_id 변수의 값을 가져옵니다. 다음 명령을 실행하여 가져올 수 있습니다. `terraform-example-foundation` 디렉토리와 같은 수준에 있다고 가정합니다. 다른 디렉토리에서 실행하는 경우, 경로를 적절히 조정하세요.

   ```bash
   export ORGANIZATION_ID=$(terraform -chdir="terraform-example-foundation/0-bootstrap/" output -json common_config | jq '.org_id' --raw-output)
   export ACCESS_CONTEXT_MANAGER_ID=$(gcloud access-context-manager policies list --organization ${ORGANIZATION_ID} --format="value(name)")
   echo "access_context_manager_policy_id = ${ACCESS_CONTEXT_MANAGER_ID}"
   ```

1. 이 문서에서 설명하는 수동 단계의 경우, 빌드 파이프라인에서 사용하는 같은 [Terraform](https://www.terraform.io/downloads.html) 버전을 사용해야 합니다.
그렇지 않으면 Terraform 상태 스냅샷 잠금 오류가 발생할 수 있습니다.

### 문제 해결

이 단계에서 문제가 발생하면 [문제 해결](../docs/TROUBLESHOOTING.md)을 참조하세요.

## 사용법

**참고:** MacOS를 사용하는 경우, 관련 명령에서 `cp -RT`를 `cp -R`로 바꾸세요.
`-T` 플래그는 Linux에서는 필요하지만 MacOS에서는 문제를 일으킵니다.

### 네트워킹 아키텍처

이 단계는 **이중 공유 VPC** 아키텍처를 사용하며, 더 자세한 내용은 [Google 클라우드 보안 기반 가이드](https://cloud.google.com/architecture/security-foundations/networking)의 **네트워킹** 섹션에서 찾을 수 있습니다. 허브 앤 스포크 모드를 사용하는 버전을 보려면 [3-networks-hub-and-spoke](../3-networks-hub-and-spoke) 단계를 확인하세요.

### Dedicated Interconnect 사용하기

[Dedicated Interconnect README](./modules/dedicated_interconnect/README.md)에 나열된 전제 조건을 프로비저닝한 경우, 다음 단계를 따라 Dedicated Interconnect를 활성화하여 온프레미스 리소스에 액세스할 수 있습니다.

1. `3-networks-svpc/envs/shared`의 공유 envs 폴더에서 `interconnect.tf.example`을 `interconnect.tf`로 이름을 바꿉니다
1. interconnects, locations, candidate subnetworks, vlan_tag8021q 및 peer 정보에 대해 사용자 환경에 유효한 값으로 `interconnect.tf` 파일을 업데이트합니다.
1. `3-networks-svpc/modules/base_env`의 base_env 폴더에서 `interconnect.tf.example`을 `interconnect.tf`로 이름을 바꿉니다.
1. interconnects, locations, candidate subnetworks, vlan_tag8021q 및 peer 정보에 대해 사용자 환경에 유효한 값으로 `interconnect.tf` 파일을 업데이트합니다.
1. `enable_dedicated_interconnect` 변수를 `true`로 설정합니다
1. candidate subnetworks와 vlan_tag8021q 변수는 interconnect 모듈이 이러한 값을 자동으로 생성하도록 `null`로 설정할 수 있습니다.

### Partner Interconnect 사용하기

[Partner Interconnect README](./modules/partner_interconnect/README.md)에 나열된 전제 조건을 프로비저닝한 경우, 다음 단계를 따라 Partner Interconnect를 활성화하여 온프레미스 리소스에 액세스할 수 있습니다.

1. `3-networks-svpc/envs/shared`의 공유 envs 폴더에서 `partner_interconnect.tf.example`을 `partner_interconnect.tf`로 이름을 바꿉니다
1. `3-networks-svpc/envs/shared`의 공유 envs 폴더에서 `partner_interconnect.auto.tfvars.example`을 `partner_interconnect.auto.tfvars`로 이름을 바꿉니다
1. interconnects, locations, candidate subnetworks, vlan_tag8021q 및 peer 정보에 대해 사용자 환경에 유효한 값으로 `interconnect.tf` 파일을 업데이트합니다.
1. `3-networks-svpc/modules/base_env`의 base-env 폴더에서 `partner_interconnect.tf.example`을 `partner_interconnect.tf`로 이름을 바꿉니다.
1. `3-networks-svpc/envs/<environment>`의 환경 폴더에 있는 각 `main.tf` 파일에서 `enable_partner_interconnect`를 `true`로 업데이트합니다.
1. VLAN 연결, 위치 및 후보 서브네트워크에 대해 사용자 환경에 유효한 값으로 `partner_interconnect.tf` 파일을 업데이트합니다.
1. candidate subnetworks 변수는 interconnect 모듈이 이 값을 자동으로 생성하도록 `null`로 설정할 수 있습니다.

### 선택사항 - 고가용성 VPN 사용하기

Dedicated 또는 Partner Interconnect를 사용할 수 없는 경우, HA Cloud VPN을 사용하여 온프레미스 리소스에 액세스할 수도 있습니다.

1. `3-networks-svpc/modules/base_env`의 base-env 폴더에서 `vpn.tf.example`을 `vpn.tf`로 이름을 바꿉니다.
1. VPN private pre-shared key에 대한 시크릿을 생성하고 Networks terraform 서비스 계정에 필요한 역할을 부여합니다.

   ```bash
   echo '<YOUR-PRESHARED-KEY-SECRET>' | gcloud secrets create <VPN_PRIVATE_PSK_SECRET_NAME> --project <ENV_SECRETS_PROJECT> --replication-policy=automatic --data-file=-

   gcloud secrets add-iam-policy-binding <VPN_PRIVATE_PSK_SECRET_NAME> --member='serviceAccount:<NETWORKS_TERRAFORM_SERVICE_ACCOUNT>' --role='roles/secretmanager.viewer' --project <ENV_SECRETS_PROJECT>
   gcloud secrets add-iam-policy-binding <VPN_PRIVATE_PSK_SECRET_NAME> --member='serviceAccount:<NETWORKS_TERRAFORM_SERVICE_ACCOUNT>' --role='roles/secretmanager.secretAccessor' --project <ENV_SECRETS_PROJECT>
   ```

1. VPN restricted pre-shared key에 대한 시크릿을 생성하고 Networks terraform 서비스 계정에 필요한 역할을 부여합니다.

   ```bash
   echo '<YOUR-PRESHARED-KEY-SECRET>' | gcloud secrets create <VPN_RESTRICTED_PSK_SECRET_NAME> --project <ENV_SECRETS_PROJECT> --replication-policy=automatic --data-file=-

   gcloud secrets add-iam-policy-binding <VPN_RESTRICTED_PSK_SECRET_NAME> --member='serviceAccount:<NETWORKS_TERRAFORM_SERVICE_ACCOUNT>' --role='roles/secretmanager.viewer' --project <ENV_SECRETS_PROJECT>
   gcloud secrets add-iam-policy-binding <VPN_RESTRICTED_PSK_SECRET_NAME> --member='serviceAccount:<NETWORKS_TERRAFORM_SERVICE_ACCOUNT>' --role='roles/secretmanager.secretAccessor' --project <ENV_SECRETS_PROJECT>
   ```

1. `vpn.tf` 파일에서 `environment`, `vpn_psk_secret_name`, `on_prem_router_ip_address1`, `on_prem_router_ip_address2` 및 `bgp_peer_asn`의 값을 업데이트합니다.
1. 다른 기본값이 사용자 환경에 유효한지 확인합니다.

### Cloud Build로 배포하기

1. `0-bootstrap` 단계의 Terraform 출력을 기반으로 `gcp-networks` 리포지토리를 복제합니다.
`terraform-example-foundation` 폴더와 같은 수준에서 리포를 복제합니다. 다음 지침은 이 레이아웃을 가정합니다.
`0-bootstrap` 폴더에서 `terraform output cloudbuild_project_id`를 실행하여 Cloud Build 프로젝트 ID를 가져옵니다.

   ```bash
   export CLOUD_BUILD_PROJECT_ID=$(terraform -chdir="terraform-example-foundation/0-bootstrap/" output -raw cloudbuild_project_id)
   echo ${CLOUD_BUILD_PROJECT_ID}

   gcloud source repos clone gcp-networks --project=${CLOUD_BUILD_PROJECT_ID}
   ```

1. 새로 복제된 리포지토리로 이동하고, 메인이 아닌 브랜치로 변경한 다음, 기반 콘텐츠를 새 리포지토리로 복사합니다.

   ```bash
   cd gcp-networks/
   git checkout -b plan

   cp -RT ../terraform-example-foundation/3-networks-svpc/ .
   cp ../terraform-example-foundation/build/cloudbuild-tf-* .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. `common.auto.example.tfvars`를 `common.auto.tfvars`로, `production.auto.example.tfvars`를 `production.auto.tfvars`로, `access_context.auto.example.tfvars`를 `access_context.auto.tfvars`로 이름을 바꿉니다.

   ```bash
   mv common.auto.example.tfvars common.auto.tfvars
   mv production.auto.example.tfvars production.auto.tfvars
   mv access_context.auto.example.tfvars access_context.auto.tfvars
   ```

1. 사용자 환경과 bootstrap의 값으로 `common.auto.tfvars` 파일을 업데이트합니다. `common.auto.tfvars` 파일의 값에 대한 추가 정보는 어떤 envs 폴더 [README.md](./envs/production/README.md) 파일에서 확인할 수 있습니다.
   `target_name_server_addresses`로 `production.auto.tfvars` 파일을 업데이트합니다.
   `access_context_manager_policy_id`로 `access_context.auto.tfvars` 파일을 업데이트합니다.
   0-bootstrap 출력에서 backend bucket 값을 가져오려면 `terraform output`을 사용합니다.

   ```bash
   export ORGANIZATION_ID=$(terraform -chdir="../terraform-example-foundation/0-bootstrap/" output -json common_config | jq '.org_id' --raw-output)
   export ACCESS_CONTEXT_MANAGER_ID=$(gcloud access-context-manager policies list --organization ${ORGANIZATION_ID} --format="value(name)")
   echo "access_context_manager_policy_id = ${ACCESS_CONTEXT_MANAGER_ID}"

   sed -i'' -e "s/ACCESS_CONTEXT_MANAGER_ID/${ACCESS_CONTEXT_MANAGER_ID}/" ./access_context.auto.tfvars

   export backend_bucket=$(terraform -chdir="../terraform-example-foundation/0-bootstrap/" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${backend_bucket}"

   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" ./common.auto.tfvars
   ```

   **참고:** VPC Service Controls로 보호되는 프로젝트의 리소스를 보거나 액세스하려면 사용자 ID로 `perimeter_additional_members` 변수를 업데이트해야 합니다.

1. 변경 사항 커밋

   ```bash
   git add .
   git commit -m 'Initialize networks repo'
   ```

1. `development`, `nonproduction` 및 `production` 환경이 `shared` 환경에 의존하므로, `shared` 환경을 수동으로 plan과 apply(단 한 번만)해야 합니다.
1. `tf-wrapper.sh` 스크립트의 `validate` 옵션을 사용하려면, [설치 지침](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install)을 따라 terraform-tools 구성 요소를 설치하시기 바랍니다.
1. 0-bootstrap 출력에서 Cloud Build 프로젝트 ID와 networks step Terraform Service Account를 가져오려면 `terraform output`을 사용합니다. 가장 기능을 활성화하기 위해 Terraform Service Account를 사용하여 환경 변수 `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT`가 설정됩니다.

   ```bash
   export CLOUD_BUILD_PROJECT_ID=$(terraform -chdir="../terraform-example-foundation/0-bootstrap/" output -raw cloudbuild_project_id)
   echo ${CLOUD_BUILD_PROJECT_ID}

   export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=$(terraform -chdir="../terraform-example-foundation/0-bootstrap/" output -raw networks_step_terraform_service_account_email)
   echo ${GOOGLE_IMPERSONATE_SERVICE_ACCOUNT}
   ```

1. `init`과 `plan`을 실행하고 shared 환경에 대한 출력을 검토합니다.

   ```bash
   ./tf-wrapper.sh init shared
   ./tf-wrapper.sh plan shared
   ```

1. `validate`를 실행하고 위반 사항을 확인합니다.

   ```bash
   ./tf-wrapper.sh validate shared $(pwd)/../gcp-policies ${CLOUD_BUILD_PROJECT_ID}
   ```

1. `apply` shared를 실행합니다.

   ```bash
   ./tf-wrapper.sh apply shared
   ```

1. `development`, `nonproduction` 및 `plan` 환경이 `production` 환경에 의존하므로, `production` 환경을 수동으로 plan과 apply해야 합니다.

   ```bash
   git checkout -b production
   ```

1. `init`과 `plan`을 실행하고 production 환경에 대한 출력을 검토합니다.

   ```bash
   ./tf-wrapper.sh init production
   ./tf-wrapper.sh plan production
   ```

1. production을 `apply`로 실행합니다.

   ```bash
   ./tf-wrapper.sh apply production
   ```

   1. development와 nonproduction이 production에 의존하므로 production 브랜치를 푸시합니다. 이것은 [named environment branch](../docs/FAQ.md#what-is-a-named-branch)이므로, 이 브랜치에 푸시하면 _terraform plan_과 _terraform apply_ 모두가 실행됩니다. Cloud Build 프로젝트에서 apply 출력을 검토하세요 https://console.cloud.google.com/cloud-build/builds;region=DEFAULT_REGION?project=YOUR_CLOUD_BUILD_PROJECT_ID

**참고:** Production 환경에는 다른 환경에서 사용될 DNS Hub 통신이 포함되어 있으므로, 가장 먼저 푸시되어야 하는 브랜치입니다.

   ```bash
   git push --set-upstream origin production
   ```

1. 모든 환경에 대한 plan을 실행하려면 plan 브랜치를 푸시합니다. _plan_ 브랜치는 [named environment branch](../docs/FAQ.md#what-is-a-named-branch)가 아니므로, _plan_ 브랜치를 푸시하면 _terraform plan_만 실행되고 _terraform apply_는 실행되지 않습니다. Cloud Build 프로젝트에서 plan 출력을 검토하세요 https://console.cloud.google.com/cloud-build/builds;region=DEFAULT_REGION?project=YOUR_CLOUD_BUILD_PROJECT_ID

   ```bash
   git checkout plan
   git push --set-upstream origin plan
   ```

1. plan이 적용된 후 development를 apply합니다.
1. development에 변경 사항을 병합합니다. 이것은 [named environment branch](../docs/FAQ.md#what-is-a-named-branch)이므로, 이 브랜치에 푸시하면 _terraform plan_과 _terraform apply_ 모두가 실행됩니다. Cloud Build 프로젝트에서 apply 출력을 검토하세요 https://console.cloud.google.com/cloud-build/builds;region=DEFAULT_REGION?project=YOUR_CLOUD_BUILD_PROJECT_ID

   ```bash
   git checkout -b development
   git push origin development
   ```

1. development가 적용된 후 nonproduction을 apply합니다.
1. nonproduction에 변경 사항을 병합합니다. 이것은 [named environment branch](../docs/FAQ.md#what-is-a-named-branch)이므로, 이 브랜치에 푸시하면 _terraform plan_과 _terraform apply_ 모두가 실행됩니다. Cloud Build 프로젝트에서 apply 출력을 검토하세요 https://console.cloud.google.com/cloud-build/builds;region=DEFAULT_REGION?project=YOUR_CLOUD_BUILD_PROJECT_ID

   ```bash
   git checkout -b nonproduction
   git push origin nonproduction
   ```

1. 다음 단계를 실행하기 전에 `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` 환경 변수를 해제합니다.

   ```bash
   unset GOOGLE_IMPERSONATE_SERVICE_ACCOUNT
   ```

1. 이제 [4-projects](../4-projects/README.md) 단계의 지침으로 이동할 수 있습니다.

### Jenkins로 배포하기

`0-bootstrap` [README-Jenkins.md](../0-bootstrap/README-Jenkins.md#deploying-step-3-networks-svpc)를 참조하세요.

### GitHub Actions로 배포하기

`0-bootstrap` [README-GitHub.md](../0-bootstrap/README-GitHub.md#deploying-step-3-networks-svpc)를 참조하세요.

### 로컬에서 Terraform 실행하기

1. 다음 지침은 `terraform-example-foundation` 폴더와 같은 수준에 있다고 가정합니다. `gcp-network` 폴더를 생성하고 이동한 다음, `3-networks-svpc` 콘텐츠, Terraform 래퍼 스크립트를 복사하고 실행할 수 있도록 설정합니다. 또한 로컬에서 버전을 관리할 수 있도록 git을 초기화합니다.

   ```bash
   mkdir gcp-network
   cp -R terraform-example-foundation/3-networks-svpc/* gcp-network
   cp terraform-example-foundation/build/tf-wrapper.sh gcp-network/
   cp terraform-example-foundation/.gitignore gcp-network/
   chmod 755 ./gcp-network/tf-wrapper.sh
   ```

1. `gcp-network`으로 이동하여 로컬에서 버전을 관리하기 위해 로컬 Git 리포지토리를 초기화합니다. 그런 다음 환경 브랜치를 생성합니다.

   ```bash
   cd gcp-network
   git init
   git commit -m "initialize empty directory" --allow-empty
   git checkout -b shared
   git checkout -b production
   git checkout -b development
   git checkout -b nonproduction
   ```

1. 다음 지침은 `terraform-example-foundation` 폴더와 같은 수준에 있다고 가정합니다. `3-networks-svpc` 폴더로 이동하고, Terraform 래퍼 스크립트를 복사하여 실행할 수 있도록 설정합니다.

   ```bash
   cd terraform-example-foundation/3-networks-svpc
   cp ../build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. `common.auto.example.tfvars`를 `common.auto.tfvars`로, `production.auto.example.tfvars`를 `production.auto.tfvars`로, `access_context.auto.example.tfvars`를 `access_context.auto.tfvars`로 이름을 바꿉니다.

   ```bash
   mv common.auto.example.tfvars common.auto.tfvars
   mv production.auto.example.tfvars production.auto.tfvars
   mv access_context.auto.example.tfvars access_context.auto.tfvars
   ```

1. 사용자 환경과 bootstrap의 값으로 `common.auto.tfvars` 파일을 업데이트합니다. `common.auto.tfvars` 파일의 값에 대한 추가 정보는 어떤 envs 폴더 [README.md](./envs/production/README.md) 파일에서 확인할 수 있습니다.
1. `target_name_server_addresses`로 `production.auto.tfvars` 파일을 업데이트합니다.
1. `access_context_manager_policy_id`로 `access_context.auto.tfvars` 파일을 업데이트합니다.
1. gcp-bootstrap 출력에서 backend bucket 값을 가져오려면 `terraform output`을 사용합니다.

   ```bash
   export ORGANIZATION_ID=$(terraform -chdir="../gcp-bootstrap/" output -json common_config | jq '.org_id' --raw-output)
   export ACCESS_CONTEXT_MANAGER_ID=$(gcloud access-context-manager policies list --organization ${ORGANIZATION_ID} --format="value(name)")
   echo "access_context_manager_policy_id = ${ACCESS_CONTEXT_MANAGER_ID}"

   sed -i'' -e "s/ACCESS_CONTEXT_MANAGER_ID/${ACCESS_CONTEXT_MANAGER_ID}/" ./access_context.auto.tfvars

   export backend_bucket=$(terraform -chdir="../gcp-bootstrap/" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${backend_bucket}"

   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" ./common.auto.tfvars
   ````

이제 이 스크립트를 사용하여 각 환경(development/production/nonproduction)을 배포합니다.
Cloud Build 또는 Jenkins를 CI/CD 도구로 사용할 때, 각 환경은 3-networks-svpc 단계의 리포지토리에서 브랜치에 해당하며
해당 환경만 적용됩니다.

`tf-wrapper.sh` 스크립트의 `validate` 옵션을 사용하려면, [지침](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install)을 따라 terraform-tools 구성 요소를 설치하세요.

1. 0-bootstrap 출력에서 Seed 프로젝트 ID와 organization step Terraform 서비스 계정을 가져오려면 `terraform output`을 사용합니다. 가장 기능을 활성화하기 위해 Terraform Service Account를 사용하여 환경 변수 `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT`가 설정됩니다.

   ```bash
   export SEED_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/" output -raw seed_project_id)
   echo ${SEED_PROJECT_ID}

   export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=$(terraform -chdir="../gcp-bootstrap/" output -raw networks_step_terraform_service_account_email)
   echo ${GOOGLE_IMPERSONATE_SERVICE_ACCOUNT}
   ```

1. `shared` 브랜치로 체크아웃합니다. shared 환경에 대해 `init`과 `plan`을 실행하고 출력을 검토합니다.

   ```bash
   git checkout shared
   ./tf-wrapper.sh init shared
   ./tf-wrapper.sh plan shared
   ```

1. `validate`를 실행하고 위반 사항을 확인합니다.

   ```bash
   ./tf-wrapper.sh validate shared $(pwd)/../gcp-policies ${SEED_PROJECT_ID}
   ```

1. `apply` shared를 실행합니다.

   ```bash
   ./tf-wrapper.sh apply shared
   ```

1. `production` 브랜치로 체크아웃합니다. production 환경에 대해 `init`과 `plan`을 실행하고 출력을 검토합니다.

   ```bash
   git checkout production
   git merge shared
   ./tf-wrapper.sh init production
   ./tf-wrapper.sh plan production
   ```

1. `validate`를 실행하고 위반 사항을 확인합니다.

   ```bash
   ./tf-wrapper.sh validate production $(pwd)/../gcp-policies ${SEED_PROJECT_ID}
   ```

1. production을 `apply`로 실행합니다.

   ```bash
   ./tf-wrapper.sh apply production
   git add .
   git commit -m "Initial production commit."
   cd ../
   ```

1. shared를 `git commit`으로 실행합니다.

   ```bash
   git checkout shared
   git add .
   git commit -m "Initial shared commit."
   ```

1. `development` 브랜치로 체크아웃하고 `shared`를 병합합니다. development 환경에 대해 `init`과 `plan`을 실행하고 출력을 검토합니다.

   ```bash
   git checkout development
   git merge shared
   ./tf-wrapper.sh init development
   ./tf-wrapper.sh plan development
   ```

1. `validate`를 실행하고 위반 사항을 확인합니다.

   ```bash
   ./tf-wrapper.sh validate development $(pwd)/../gcp-policies ${SEED_PROJECT_ID}
   ```

1. development를 `apply`로 실행합니다.

   ```bash
   ./tf-wrapper.sh apply development
   git add .
   git commit -m "Initial development commit."
   ```

1. `nonproduction`으로 체크아웃하고 `development`를 병합합니다. nonproduction 환경에 대해 `init`과 `plan`을 실행하고 출력을 검토합니다.

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

1. nonproduction을 `apply`로 실행합니다.

   ```bash
   ./tf-wrapper.sh apply nonproduction
   git add .
   git commit -m "Initial nonproduction commit."
   ```

오류가 발생하거나 Terraform 구성 또는 `.tfvars`에 변경사항을 적용한 경우, `./tf-wrapper.sh apply <env>`를 실행하기 전에 `./tf-wrapper.sh plan <env>`를 다시 실행해야 합니다.

다음 단계를 실행하기 전에 `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` 환경 변수를 해제합니다.

```bash
unset GOOGLE_IMPERSONATE_SERVICE_ACCOUNT
```

### (선택사항) VPC 서비스 제어 적용하기

VPC 서비스 제어를 활성화하는 것은 중단을 일으킬 수 있는 프로세스이므로, 이 리포지토리는 기본적으로 VPC 서비스 제어 경계를 드라이 런 모드로 구성합니다. 이 구성은 보안 경계를 가로지르는 서비스 트래픽 (경계 내부에서 외부 리소스와 통신하는 API 요청 또는 외부 리소스에서 경계 내부 리소스와 통신하는 API 요청)을 모니터링하지만 여전히 서비스 트래픽을 정상적으로 허용합니다.

VPC 서비스 제어를 적용할 준비가 되면, [VPC 서비스 제어 활성화를 위한 모범 사례](https://cloud.google.com/vpc-service-controls/docs/enable)의 지침을 검토하는 것이 좋습니다. 필요한 예외를 추가하고 VPC 서비스 제어가 의도한 작업을 방해하지 않을 것이라고 확신한 후, `shared_vpc` 모듈 하의 `enforce_vpcsc` 변수를 `true`로 설정하고 이 단계를 다시 적용합니다. 그런 다음 새 설정을 상속받고 해당 프로젝트를 적용된 경계 내에 포함시킬 4-projects 단계를 다시 적용합니다.

기존의 적용된 경계를 변경해야 할 때, [드라이 런 경계](https://cloud.google.com/vpc-service-controls/docs/dry-run-mode)의 구성을 수정하여 안전하게 테스트할 수 있습니다. 이렇게 하면 적용된 경계가 트래픽을 허용하거나 거부하는지에 영향을 주지 않으면서 드라이 런 경계에서 거부된 트래픽을 기록합니다.
