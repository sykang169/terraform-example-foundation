*경고: Jenkins를 사용한 배포 가이드는 더 이상 적극적으로 테스트되거나 유지보수되지 않습니다. Jenkins를 선호하는 사용자를 위해 가이드를 제공하고 있지만, 품질에 대한 보장은 없으며, 문제 해결 및 지침 수정에 대한 책임은 사용자에게 있을 수 있습니다.*

# 0-bootstrap - Jenkins 호환 환경 배포

이 단계의 목적은 GCP 조직을 부트스트랩하여 CFT(Cloud Foundation Toolkit) 사용을 시작하는 데 필요한 모든 리소스와 권한을 생성하는 것입니다. 이 단계는 또한 기존 Jenkins Controller 인프라와 자체 Git 저장소(온프레미스에 있을 수 있음)에 연결되는 Jenkins Agent를 호스팅하기 위한 CI/CD 프로젝트를 구성하는 방법에 대해 안내합니다. Jenkins Agent는 후속 단계에서 foundation 코드에 대한 [CI/CD 파이프라인](../docs/GLOSSARY.md#foundation-cicd-pipeline)을 실행합니다.

다른 CI/CD 옵션은 Cloud Build와 Cloud Source Repos를 사용하는 것입니다. Jenkins 구현이 없고 원하지 않는 경우, 대신 [Cloud Build 모듈 사용](./README.md#deploying-with-cloud-build)을 권장합니다.

**면책조항:** Jenkins 지원은 향후 릴리스에서 더 이상 사용되지 않을 예정입니다. 대신 [Cloud Build 모듈 사용](./README.md#deploying-with-cloud-build)을 고려하십시오.

## 개요

아래 지침의 목표는 Jenkins를 사용하여 다음 단계(`1-org, 2-environments, 3-networks, 4-projects`)에 대한 CI/CD 배포를 실행할 수 있는 인프라를 구성하는 것입니다. 인프라는 두 개의 Google Cloud Platform 프로젝트(`prj-b-seed` 및 `prj-b-cicd`)와 온프레미스 환경에 연결하는 VPN 구성으로 구성됩니다.

관심사 분리를 위해 여기서 두 개의 별도 프로젝트(`prj-b-seed` 및 `prj-b-cicd`)를 갖는 것이 모범 사례입니다. 한편으로, `prj-b-seed`는 terraform 상태를 저장하고 인프라를 생성/수정할 수 있는 서비스 계정을 보유합니다. 반면에, 해당 인프라의 배포는 `prj-b-cicd`에서 구현되고 온프레미스의 Controller에 연결된 Jenkins에 의해 조정됩니다.

**아래 지침을 따른 후 다음을 보유하게 됩니다:**

- `prj-b-seed` 프로젝트, 여기에는 다음이 포함됩니다:
  - Terraform 상태 버킷
  - GCP에서 새 리소스를 생성하기 위해 Terraform에서 사용하는 사용자 지정 서비스 계정
- `prj-b-cicd` 프로젝트, 여기에는 다음이 포함됩니다:
  - SSH를 사용하여 현재 Jenkins Controller에 연결된 Jenkins Agent용 GCE 인스턴스
  - Jenkins GCE 인스턴스를 연결할 VPC
  - 포트 22를 통한 통신을 허용하는 방화벽 규칙
  - 온프레미스(또는 Jenkins Controller가 있는 어느 곳이든)와의 VPN 연결
  - GCE 인스턴스용 사용자 지정 서비스 계정 `sa-jenkins-agent-gce@prj-b-cicd-xxxx.iam.gserviceaccount.com`
    - 이 서비스 계정은 `prj-b-seed` 프로젝트의 Terraform 사용자 지정 서비스 계정에서 토큰을 생성할 수 있는 액세스 권한이 부여됩니다.

- **참고: 이 지침은 Jenkins Controller를 생성하는 방법을 설명하지 않습니다.** Jenkins Controller를 배포하려면 [Jenkins 아키텍처](https://www.jenkins.io/doc/book/scaling/architecting-for-scale/) 권장사항을 따르십시오.

**Jenkins 구현이 없고 원하지 않는 경우**, 대신 [Cloud Build 모듈 사용](./README.md#deploying-with-cloud-build)을 권장합니다.

## 요구사항

아래 지침을 따르기 전에 소프트웨어, 인프라 및 권한에 대한 **[요구사항](./modules/jenkins-agent/README.md#Requirements)**을 참조하십시오.

## 사용법

**참고:** MacOS를 사용하는 경우, 관련 명령에서 `cp -RT`를 `cp -R`로 바꿉니다. `-T` 플래그는 Linux에서 필요하지만 MacOS에서는 문제를 일으킵니다.

## 지침

`cloudbuild_bootstrap` 대신 `jenkins_bootstrap`을 사용하여 0-bootstrap 단계를 실행하기 때문에 이 지침에 도달했습니다. 아래 지시사항을 따르십시오:

- 아래 지침을 따르기 전에 소프트웨어, 인프라 및 권한의 모든 [요구사항](./modules/jenkins-agent/README.md#Requirements)을 충족하는지 확인하십시오.

### I. 환경 설정

- 필수 정보:
  - `ssh-keygen` 명령을 실행할 Jenkins Controller 호스트에 대한 액세스
  - Jenkins Controller Web UI에 대한 액세스
  - Jenkins Controller에 설치된 [SSH Agent Jenkins plugin](https://plugins.jenkins.io/ssh-agent)
  - Jenkins Agent의 사설 IP 주소: 일반적으로 네트워크 관리자가 할당합니다. 이 IP는 [II. Terraform을 사용하여 SEED 및 CI/CD 프로젝트 생성](#ii-create-the-seed-and-cicd-projects-using-terraform) 단계에서 `prj-b-cicd` GCP 프로젝트에 생성될 GCE 인스턴스에 사용됩니다.
  - 이 [monorepo](https://github.com/terraform-google-modules/terraform-example-foundation)의 각 디렉토리에 대해 하나씩 다섯 개의 Git 저장소를 만들 수 있는 액세스(`gcp-bootstrap, gcp-org, gcp-environments, gcp-networks, gcp-projects`). 이들은 보통 온프레미스에 있을 수 있는 비공개 저장소입니다.

1. SSH 키 쌍을 생성합니다. Jenkins Controller 호스트에서 `ssh-keygen` 명령을 사용하여 SSH 키 쌍을 생성합니다.
   - Controller와 Agent 간의 인증을 활성화하려면 이 키 쌍이 필요합니다. 키 쌍은 어떤 linux 머신에서도 생성할 수 있지만, 비밀 사설 키를 한 호스트에서 다른 호스트로 복사하지 않는 것이 좋으므로, Jenkins Controller 호스트 명령줄에서 이 작업을 수행하는 것이 좋습니다.
   - `ssh-keygen` 명령은 `-N` 옵션을 사용하여 사설 키를 암호로 보호합니다. 이 예시에서는 `-N "my-password"`를 사용합니다. Jenkins Controller Web UI에서 SSH Agent를 구성할 때 사설 키와 암호가 모두 필요하므로 이는 중요합니다.

   ```bash
   SSH_LOCAL_CONFIG_DIR="$HOME/.ssh"
   JENKINS_USER="jenkins"
   JENKINS_AGENT_NAME="AgentGCE1"
   SSH_KEY_FILE_PATH="$SSH_LOCAL_CONFIG_DIR/$JENKINS_USER-${JENKINS_AGENT_NAME}_rsa"
   mkdir -p "$SSH_LOCAL_CONFIG_DIR"
   ssh-keygen -t rsa -m PEM -N "my-password" -C $JENKINS_USER -f $SSH_KEY_FILE_PATH
   cat $SSH_KEY_FILE_PATH
   ```

   - 다음과 비슷한 출력을 볼 수 있습니다:

      ![RSA private key example](./files/private_key_example.png)

1. Jenkins Controller의 Web UI에서 새 SSH Jenkins Agent를 구성합니다. 다음 정보가 필요합니다:
   - Controller에 설치된 [SSH Agent Jenkins plugin](https://plugins.jenkins.io/ssh-agent/)
   - 이전 단계에서 방금 생성한 SSH 사설 키
   - 사설 키를 보호하는 암호구(`-N` 옵션에서 사용한 것)
   - Jenkins Agent의 사설 IP 주소(보통 네트워크 관리자가 할당. 제공된 예시에서 이 IP는 "172.16.1.6"입니다). 이 사설 IP는 나중에 생성할 VPN 연결을 통해 접근 가능합니다.

1. Git 서버에 다섯 개의 개별 Git 저장소를 만듭니다(이는 인프라 팀에 위임될 수 있는 작업입니다)
   - 이 인프라 코드는 [monorepo](https://github.com/terraform-google-modules/terraform-example-foundation)로 배포되지만, 각 디렉토리에 대해 하나씩 다섯 개의 서로 다른 저장소에 코드를 저장합니다:

   ```text
   ./gcp-bootstrap
   ./gcp-org
   ./gcp-environments
   ./gcp-networks
   ./gcp-projects
   ```

   - 간단히 하기 위해 다섯 개의 저장소 이름을 다음과 같이 지정합니다:

   ```text
   YOUR_NEW_REPO-gcp-bootstrap
   YOUR_NEW_REPO-gcp-org
   YOUR_NEW_REPO-gcp-environments
   YOUR_NEW_REPO-gcp-networks
   YOUR_NEW_REPO-gcp-projects
   ```

   - **참고:** 이 지침의 끝에서 **다음 저장소에만** 새 자동 파이프라인으로 Jenkins Controller를 구성합니다:

   ```text
   YOUR_NEW_REPO-gcp-org
   YOUR_NEW_REPO-gcp-environments
   YOUR_NEW_REPO-gcp-networks
   YOUR_NEW_REPO-gcp-projects
   ```

   - **참고: `YOUR_NEW_REPO-gcp-bootstrap`에는 자동 파이프라인이 필요하지 않습니다**
   - 이 0-bootstrap 섹션에서는 `./0-bootstrap` 디렉토리의 복사본인 새 저장소(`YOUR_NEW_REPO-gcp-bootstrap`)만 작업합니다

1. 다음으로 이 모노 저장소를 복제합니다:

   ```bash
   git clone https://github.com/terraform-google-modules/terraform-example-foundation
   ```

1. `0-bootstrap` 디렉토리를 호스팅하기 위해 생성한 저장소를 복제합니다:

   ```bash
   git clone <YOUR_NEW_REPO-gcp-bootstrap> gcp-bootstrap
   ```

1. 방금 복제한 저장소로 이동하고 새 브랜치로 변경합니다

   ```bash
   cd gcp-bootstrap
   git checkout -b plan
   ```

1. foundation의 내용을 새 저장소로 복사합니다(현재 디렉토리에 따라 적절히 수정).

   ```bash
   mkdir -p envs/shared
   cp -RT ../terraform-example-foundation/0-bootstrap/ ./envs/shared
   cd ./envs/shared
   ```

1. Jenkins 모듈을 활성화하고 Cloud Build 모듈을 비활성화합니다. 이는 다음 파일을 수동으로 편집하는 것을 의미합니다:
   1. 파일 `./cb.tf`를 `./cb.tf.example`로 이름 변경

   ```bash
   mv ./cb.tf ./cb.tf.example
   ```

   1. 파일 `./jenkins.tf.example`을 `./jenkins.tf`로 이름 변경

   ```bash
   mv ./jenkins.tf.example ./jenkins.tf
   ```

   1. `./variables.tf`에서 `jenkins_bootstrap` 변수들의 주석 해제
   1. `./outputs.tf`에서 `jenkins_bootstrap` 출력들의 주석 해제
   1. `./outputs.tf`에서 `cloudbuild_bootstrap` 출력들을 주석 처리
1. `terraform.example.tfvars`를 `terraform.tfvars`로 이름을 변경하고 환경의 값으로 파일을 업데이트합니다.

   ```bash
   mv ./terraform.example.tfvars ./terraform.tfvars
   ```

1. 제공해야 하는 값 중 하나(변수 `jenkins_agent_gce_ssh_pub_key`)는 첫 번째 단계에서 생성한 **공개 SSH 키**입니다.
   - **참고: 이는 비밀 사설 키가 아닙니다**. 공개 SSH 키는 저장소 코드에 있을 수 있습니다.
1. 다음을 사용하여 공개 키를 표시합니다

   ```bash
   cat "${SSH_KEY_FILE_PATH}.pub"
   ```

1. 이를 `terraform.tfvars` 파일(변수 `jenkins_agent_gce_ssh_pub_key`)에 복사/붙여넣기 합니다.
1. `terraform.tfvars`에 필요한 나머지 값들을 제공합니다

1. 환경을 검증하려면 헬퍼 스크립트 [validate-requirements.sh](../scripts/validate-requirements.sh)를 사용합니다:

   ```bash
   ../../../terraform-example-foundation/scripts/validate-requirements.sh  -o <ORGANIZATION_ID> -b <BILLING_ACCOUNT_ID> -u <END_USER_EMAIL> -e
   ```

   **참고:** 스크립트는 사용자가 필요한 역할을 가진 Cloud Identity 또는 Google Workspace 그룹에 있는지 검증할 수 없습니다.

1. 변경 사항을 커밋합니다:

   ```bash
   cd ../..
   git add .
   git commit -m 'Bootstrap configuration using jenkins_module'
   ```

1. 다음으로 `plan` 브랜치를 YOUR_NEW_REPO-gcp-bootstrap 저장소에 푸시합니다

   ```bash
   git push --set-upstream origin plan
   cd ./envs/shared
   ```

### II. Terraform을 사용하여 SEED 및 CI/CD 프로젝트 생성

- 필수 정보:
  - Terraform 버전 1.5.7 - 자세한 내용은 [요구사항](#requirements) 섹션을 참조하십시오.
  - 모든 필수 값을 갖는 `terraform.tfvars` 파일.

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


1. 적절한 자격 증명을 가져옵니다: [필요한 권한](./modules/jenkins-agent/README.md#permissions)을 가진 계정으로 다음 명령을 실행합니다.

   ```bash
   gcloud auth application-default login
   ```

1. 브라우저에서 링크를 열고 수락합니다.

1. Run terraform commands.
   - After the credentials are configured, we will create the `prj-b-seed` project (which contains the GCS state bucket and Terraform custom service account) and the `prj-b-cicd` project (which contains the Jenkins Agent, its custom service account and where we will add VPN configuration)
   - **Use Terraform 1.5.7** to run the terraform script with the commands below

   ```bash
   terraform init
   terraform plan
   terraform apply
   ```

   - The Terraform script will take about 10 to 15 minutes. Once it finishes, note that communication between on-prem and the `prj-b-cicd` project won’t happen yet - you will configure the VPN network connectivity in step [III. Create VPN connection](#iii-configure-vpn-connection).

1. Move Terraform State to the GCS bucket created in the Seed Project
   1. Get tfstate bucket name

   ```bash
   export backend_bucket=$(terraform output -raw gcs_bucket_tfstate)
   echo "backend_bucket = ${backend_bucket}"
   ```

   1. Rename `backend.tf.example` to `backend.tf` and update with your tfstate bucket name

   ```bash
   mv backend.tf.example backend.tf
   for i in `find . -name 'backend.tf'`; do sed -i'' -e "s/UPDATE_ME/${backend_bucket}/" $i; done
   ```

1. Re-run `terraform init` and agree to copy state to gcs when prompted

   ```bash
   terraform init
   ```

   - (Optional) Run `terraform apply` to verify state is configured correctly. You can confirm the terraform state is now in that bucket by visiting the bucket url in your Seed Project.

1. Commit changes

   ```bash
   git add backend.tf
   git commit -m 'Terraform Backend configuration using GCS'
   ```

1. 다음으로 `plan` 브랜치를 YOUR_NEW_REPO-gcp-bootstrap 저장소에 푸시합니다

   ```bash
   git push
   ```

### III. Configure VPN connection

Here you will configure a VPN Network tunnel to enable connectivity between the `prj-b-cicd` project and your on-prem environment. Learn more about [a VPN tunnel in GCP](https://cloud.google.com/network-connectivity/docs/vpn/how-to).

- Required information:
  - On-prem VPN public IP Address
  - Jenkins Controller’s network CIDR (the example code uses "10.1.0.0/24")
  - Jenkins Agent network CIDR (the example code uses "172.16.1.0/24")
  - VPN PSK (pre-shared secret key)

1. Check in the `prj-b-cicd` project for the VPN gateway static IP addresses which have been reserved. These addresses are required by the Network Administrator for the configuration of the on-prem side of the VPN tunnels to GCP.
   1. Assuming your network administrator already configured the on-prem end of the VPN, the CI/CD end of the VPN might show the message `First Handshake` for around 5 minutes.
   1. When the VPN is ready, the status will show `Tunnel is up and running`. At this point, your Jenkins Controller (on-prem) and Jenkins Agent (in `prj-b-cicd` project) must have network connectivity through the VPN.

1. Test a pipeline using the Jenkins Controller Web UI:
   1. Make sure your [SSH Agent](https://plugins.jenkins.io/ssh-agent) is online and troubleshoot network connectivity if needed.
   1. Test that your Jenkins Controller can deploy a [pipeline](https://www.jenkins.io/doc/book/pipeline/getting-started/) to the Jenkins Agent located in the `prj-b-cicd` project (you can test this by running with a simple `echo "Hello World"` pipeline build).

### IV. Configure the Git repositories and Multibranch Pipelines in your Jenkins Controller

- **Note:** this section is considered out of the scope of this document. Since there are multiple options on how to configure the Git repositories and **Multibranch Pipeline** in your Jenkins Controller, here we can only provide some guidance that you should keep in mind while completing this step. Visit the [Jenkins website](https://jenkins.io) for more information, there are plenty of Jenkins Plugins that could help with the task.
  - You need to configure a **"Multibranch Pipeline"**. Note that the `Jenkinsfile` and `tf-wrapper.sh` files use the `$BRANCH_NAME` environment variable. **the `$BRANCH_NAME` variable is only available in Jenkins' Multibranch Pipelines**.
- **Jenkinsfile:** A [Jenkinsfile](../build/Jenkinsfile) has been included which closely aligns with the Cloud Build pipeline. Additionally, the stage `TF wait for approval` which lets you confirm via Jenkins UI before proceeding with `terraform apply` has been disabled by default. It can be enabled by un-commenting that stage in the file.

1. Create Multibranch pipelines for your new repos (`YOUR_NEW_REPO-gcp-org, YOUR_NEW_REPO-gcp-environments, YOUR_NEW_REPO-gcp-networks, YOUR_NEW_REPO-gcp-projects`).
   - **DO NOT configure an automatic pipeline for your `YOUR_NEW_REPO-gcp-bootstrap` repository**

1. In your Jenkins Controller Web UI, **create Multibranch Pipelines only for the following repositories:**

   ```text
   YOUR_NEW_REPO-gcp-org
   YOUR_NEW_REPO-gcp-environments
   YOUR_NEW_REPO-gcp-networks
   YOUR_NEW_REPO-gcp-projects
   ```

1. Assuming your new Git repositories are private, you may need to configure new credentials In your Jenkins Controller web UI, so it can connect to the repositories.

1. You will also want to configure automatic triggers in each one of the Jenkins Multibranch Pipelines, unless you want to run the pipelines manually from the Jenkins Web UI after each commit to your repositories.

1. You can now move to the instructions for step 1-org.

## Deploying step 1-org

1. Clone the repo you created manually in 0-bootstrap instructions.

   ```bash
   git clone <YOUR_NEW_REPO-gcp-org> gcp-org
   ```

1. Navigate into the repo and change to a nonproduction branch. All subsequent
   steps assume you are running them from the `gcp-org` directory. If
   you run them from another directory, adjust your copy paths accordingly.

   ```bash
   cd gcp-org
   git checkout -b plan
   ```

1. Copy contents of foundation to new repo.

   ```bash
   cp -RT ../terraform-example-foundation/1-org/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   cp ../terraform-example-foundation/build/Jenkinsfile .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. Update the variables located in the `environment {}` section of the `Jenkinsfile` with values from gcp-bootstrap:

   ```text
   _TF_SA_EMAIL
   _STATE_BUCKET_NAME
   _PROJECT_ID (the CI/CD project ID)
   ```

1. You can re-run `terraform output` in the gcp-bootstrap directory to find these values.

   ```bash
   BACKEND_STATE_BUCKET_NAME=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw gcs_bucket_tfstate)
   echo "_STATE_BUCKET_NAME = ${BACKEND_STATE_BUCKET_NAME}"
   sed -i'' -e "s/BACKEND_STATE_BUCKET_NAME/${BACKEND_STATE_BUCKET_NAME}/" ./Jenkinsfile

   TERRAFORM_SA_EMAIL=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw organization_step_terraform_service_account_email)
   echo "_TF_SA_EMAIL = ${TERRAFORM_SA_EMAIL}"
   sed -i'' -e "s/TERRAFORM_SA_EMAIL/${TERRAFORM_SA_EMAIL}/" ./Jenkinsfile

   CICD_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw cicd_project_id)
   echo "_PROJECT_ID = ${CICD_PROJECT_ID}"
   sed -i'' -e "s/CICD_PROJECT_ID/${CICD_PROJECT_ID}/" ./Jenkinsfile
   ```

1. Rename `./envs/shared/terraform.example.tfvars` to `./envs/shared/terraform.tfvars`

   ```bash
   mv ./envs/shared/terraform.example.tfvars ./envs/shared/terraform.tfvars
   ```

1. Check if a Security Command Center Notification with the default name, **scc-notify**, already exists. If it exists, choose a different value for the `scc_notification_name` variable in the `./envs/shared/terraform.tfvars` file.

   ```bash
   export ORGANIZATION_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -json common_config | jq '.org_id' --raw-output)
   gcloud scc notifications describe "scc-notify" --organization=${ORGANIZATION_ID}
   ```

1. Check if your organization already has an Access Context Manager Policy.

   ```bash
   export ACCESS_CONTEXT_MANAGER_ID=$(gcloud access-context-manager policies list --organization ${ORGANIZATION_ID} --format="value(name)")
   echo "access_context_manager_policy_id = ${ACCESS_CONTEXT_MANAGER_ID}"
   ```

1. Update the `envs/shared/terraform.tfvars` file with values from your environment and 0-bootstrap step. If the previous step showed a numeric value, make sure to un-comment the variable `create_access_context_manager_access_policy = false`. See the shared folder [README.md](../1-org/envs/shared/README.md) for additional information on the values in the `terraform.tfvars` file.

   ```bash
   export backend_bucket=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${backend_bucket}"

   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" ./envs/shared/terraform.tfvars

   if [ ! -z "${ACCESS_CONTEXT_MANAGER_ID}" ]; then sed -i'' -e "s=//create_access_context_manager_access_policy=create_access_context_manager_access_policy=" ./envs/shared/terraform.tfvars; fi
   ```

1. Commit changes.

   ```bash
   git add .
   git commit -m 'Initialize org repo'
   ```

1. Push your plan branch.
   - Assuming you configured an automatic trigger in your Jenkins Controller (see [Jenkins sub-module README](./modules/jenkins-agent/README.md)), this will trigger a plan. You can also trigger a Jenkins job manually. Given the many options to do this in Jenkins, it is out of the scope of this document see [Jenkins website](https://www.jenkins.io) for more details.

   ```bash
   git push --set-upstream origin plan
   ```

1. Review the plan output in your Controller's web UI.
1. Merge changes to production branch.

   ```bash
   git checkout -b production
   git push --set-upstream origin production
   ```

1. Review the apply output in your Controller's web UI. (you might want to use the option to "Scan Multibranch Pipeline Now" in your Jenkins Controller UI).

## Deploying step 2-environments

1. Clone the repo you created manually in 0-bootstrap.

   ```bash
   git clone <YOUR_NEW_REPO-gcp-environments> gcp-environments
   ```

1. Navigate into the repo and change to a nonproduction branch. All subsequent
   steps assume you are running them from the `gcp-environments` directory. If
   you run them from another directory, adjust your copy paths accordingly.

   ```bash
   cd gcp-environments
   git checkout -b plan
   ```

1. Copy contents of foundation to new repo.

   ```bash
   cp -RT ../terraform-example-foundation/2-environments/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   cp ../terraform-example-foundation/build/Jenkinsfile .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. Update the variables located in the `environment {}` section of the `Jenkinsfile` with values from your environment:

   ```text
   _TF_SA_EMAIL
   _STATE_BUCKET_NAME
   _PROJECT_ID (the CI/CD project ID)
   ```

1. You can re-run `terraform output` in the gcp-bootstrap directory to find these values.

   ```bash
   BACKEND_STATE_BUCKET_NAME=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw gcs_bucket_tfstate)
   echo "_STATE_BUCKET_NAME = ${BACKEND_STATE_BUCKET_NAME}"
   sed -i'' -e "s/BACKEND_STATE_BUCKET_NAME/${BACKEND_STATE_BUCKET_NAME}/" ./Jenkinsfile

   TERRAFORM_SA_EMAIL=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw environment_step_terraform_service_account_email)
   echo "_TF_SA_EMAIL = ${TERRAFORM_SA_EMAIL}"
   sed -i'' -e "s/TERRAFORM_SA_EMAIL/${TERRAFORM_SA_EMAIL}/" ./Jenkinsfile

   CICD_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw cicd_project_id)
   echo "_PROJECT_ID = ${CICD_PROJECT_ID}"
   sed -i'' -e "s/CICD_PROJECT_ID/${CICD_PROJECT_ID}/" ./Jenkinsfile
   ```

1. Rename `terraform.example.tfvars` to `terraform.tfvars` and update the file with values from your environment and 0-bootstrap.

   ```bash
   mv terraform.example.tfvars terraform.tfvars
   ```

1. You can re-run `terraform output` in the gcp-bootstrap directory to find these values. See any of the envs folder [README.md](../2-environments/envs/production/README.md) files for additional information on the values in the `terraform.tfvars` file.

   ```bash
   export backend_bucket=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${backend_bucket}"
   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" ./terraform.tfvars
   ```

1. Commit changes.

   ```bash
   git add .
   git commit -m 'Initialize environments repo'
   ```

1. Push your plan branch.

   ```bash
   git push --set-upstream origin plan
   ```

   - Assuming you configured an automatic trigger in your Jenkins Controller (see [Jenkins sub-module README](./modules/jenkins-agent/README.md)), this will trigger a plan. You can also trigger a Jenkins job manually. Given the many options to do this in Jenkins, it is out of the scope of this document see [Jenkins website](https://www.jenkins.io) for more details.
1. Review the plan output in your Controller's web UI.
1. Merge changes to development.

   ```bash
   git checkout -b development
   git push --set-upstream origin development
   ```

1. Review the apply output in your Controller's web UI (you might want to use the option to "Scan Multibranch Pipeline Now" in your Jenkins Controller UI).
1. Merge changes to nonproduction with.

   ```bash
   git checkout -b nonproduction
   git push --set-upstream origin nonproduction
   ```

1. Review the apply output in your Controller's web UI (you might want to use the option to "Scan Multibranch Pipeline Now" in your Jenkins Controller UI).
1. Merge changes to production branch.

   ```bash
   git checkout -b production
   git push --set-upstream origin production
   ```

1. Review the apply output in your Controller's web UI (you might want to use the option to "Scan Multibranch Pipeline Now" in your Jenkins Controller UI).
1. You can now move to the instructions in the next step, go to [Deploying step 3-networks-svpc](#deploying-step-3-networks-svpc) to use the Dual Shared VPC mode, or go to [Deploying step  3-networks-hub-and-spoke](#deploying-step-3-networks-hub-and-spoke) to use the Hub and Spoke network mode.

## Deploying step 3-networks-svpc

1. Clone the repo you created manually in 0-bootstrap.

   ```bash
   git clone <YOUR_NEW_REPO-gcp-networks> gcp-networks
   ```

1. Navigate into the repo and change to a nonproduction branch. All subsequent
   steps assume you are running them from the `gcp-networks` directory. If
   you run them from another directory, adjust your copy paths accordingly.

   ```bash
   cd gcp-networks
   git checkout -b plan
   ```

1. Copy contents of foundation to new repo.

   ```bash
   cp -RT ../terraform-example-foundation/3-networks-svpc/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   cp ../terraform-example-foundation/build/Jenkinsfile .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. Update the variables located in the `environment {}` section of the `Jenkinsfile` with values from your environment:

   ```text
   _TF_SA_EMAIL
   _STATE_BUCKET_NAME
   _PROJECT_ID (the CI/CD project ID)
   ```

1. You can re-run `terraform output` in the gcp-bootstrap directory to find these values.

   ```bash
   BACKEND_STATE_BUCKET_NAME=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw gcs_bucket_tfstate)
   echo "_STATE_BUCKET_NAME = ${BACKEND_STATE_BUCKET_NAME}"
   sed -i'' -e "s/BACKEND_STATE_BUCKET_NAME/${BACKEND_STATE_BUCKET_NAME}/" ./Jenkinsfile

   TERRAFORM_SA_EMAIL=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw networks_step_terraform_service_account_email)
   echo "_TF_SA_EMAIL = ${TERRAFORM_SA_EMAIL}"
   sed -i'' -e "s/TERRAFORM_SA_EMAIL/${TERRAFORM_SA_EMAIL}/" ./Jenkinsfile

   CICD_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw cicd_project_id)
   echo "_PROJECT_ID = ${CICD_PROJECT_ID}"
   sed -i'' -e "s/CICD_PROJECT_ID/${CICD_PROJECT_ID}/" ./Jenkinsfile
   ```

1. Rename `common.auto.example.tfvars` to `common.auto.tfvars`, rename `production.auto.example.tfvars` to `production.auto.tfvars` and rename `access_context.auto.example.tfvars` to `access_context.auto.tfvars`.

   ```bash
   mv common.auto.example.tfvars common.auto.tfvars
   mv production.auto.example.tfvars production.auto.tfvars
   mv access_context.auto.example.tfvars access_context.auto.tfvars
   ```

1. Update `common.auto.tfvars` file with values from your environment and bootstrap. See any of the envs folder [README.md](../3-networks-svpc/envs/production/README.md) files for additional information on the values in the `common.auto.tfvars` file.
1. Update `production.auto.tfvars` file with the `target_name_server_addresses`.
1. Update `access_context.auto.tfvars` file with the `access_context_manager_policy_id`.
1. Use `terraform output` to get the backend bucket and networks step Terraform Service Account values from gcp-bootstrap output.

   ```bash
   export ORGANIZATION_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -json common_config | jq '.org_id' --raw-output)
   export ACCESS_CONTEXT_MANAGER_ID=$(gcloud access-context-manager policies list --organization ${ORGANIZATION_ID} --format="value(name)")
   echo "access_context_manager_policy_id = ${ACCESS_CONTEXT_MANAGER_ID}"
   sed -i'' -e "s/ACCESS_CONTEXT_MANAGER_ID/${ACCESS_CONTEXT_MANAGER_ID}/" ./access_context.auto.tfvars

   export backend_bucket=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${backend_bucket}"
   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" ./common.auto.tfvars
   ```

1. Commit changes.

   ```bash
   git add .
   git commit -m 'Initialize networks repo'
   ```

1. You must manually plan and apply the `shared` environment (only once) since the `development`, `nonproduction` and `production` environments depend on it.
1. To use the `validate` option of the `tf-wrapper.sh` script, please follow the [instructions](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install) to install the terraform-tools component.
1. Also update `backend.tf` with your backend bucket from gcp-bootstrap output.

   ```bash
   for i in `find . -name 'backend.tf'`; do sed -i'' -e "s/UPDATE_ME/${backend_bucket}/" $i; done
   ```

1. Use `terraform output` to get the Cloud Build project ID and the networks step Terraform Service Account from gcp-bootstrap output. An environment variable `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` will be set using the Terraform Service Account to enable impersonation.

   ```bash
   export CICD_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw cicd_project_id)
   echo ${CICD_PROJECT_ID}

   export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw networks_step_terraform_service_account_email)
   echo ${GOOGLE_IMPERSONATE_SERVICE_ACCOUNT}
   ```

1. Run `init` and `plan` and review output for environment shared.

   ```bash
   ./tf-wrapper.sh init shared
   ./tf-wrapper.sh plan shared
   ```

1. Run `validate` and check for violations.

   ```bash
   ./tf-wrapper.sh validate shared $(pwd)/policy-library ${CICD_PROJECT_ID}
   ```

1. Run `apply` shared.

   ```bash
   ./tf-wrapper.sh apply shared
   ```

1. Push your plan branch.

   ```bash
   git push --set-upstream origin plan
   ```

   - Assuming you configured an automatic trigger in your Jenkins Controller (see [Jenkins sub-module README](./modules/jenkins-agent/README.md)), this will trigger a plan. You can also trigger a Jenkins job manually. Given the many options to do this in Jenkins, it is out of the scope of this document see [Jenkins website](https://www.jenkins.io) for more details.
1. Review the plan output in your Controller's web UI.
1. Merge changes to production branch.

   ```bash
   git checkout -b production
   git push --set-upstream origin production
   ```

1. Review the apply output in your Controller's web UI (you might want to use the option to "Scan Multibranch Pipeline Now" in your Jenkins Controller UI).
1. After production has been applied, apply development and nonproduction.
1. Merge changes to development

   ```bash
   git checkout -b development
   git push --set-upstream origin development
   ```

1. Review the apply output in your Controller's web UI (you might want to use the option to "Scan Multibranch Pipeline Now" in your Jenkins Controller UI).
1. Merge changes to nonproduction.

   ```bash
   git checkout -b nonproduction
   git push --set-upstream origin nonproduction
   ```

1. Review the apply output in your Controller's web UI (you might want to use the option to "Scan Multibranch Pipeline Now" in your Jenkins Controller UI).

## Deploying step 3-networks-hub-and-spoke

1. Clone the repo you created manually in 0-bootstrap.

   ```bash
   git clone <YOUR_NEW_REPO-gcp-networks> gcp-networks
   ```

1. Navigate into the repo and change to a nonproduction branch. All subsequent
   steps assume you are running them from the `gcp-networks` directory. If
   you run them from another directory, adjust your copy paths accordingly.

   ```bash
   cd gcp-networks
   git checkout -b plan
   ```

1. Copy contents of foundation to new repo.

   ```bash
   cp -RT ../terraform-example-foundation/3-networks-hub-and-spoke/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   cp ../terraform-example-foundation/build/Jenkinsfile .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. Update the variables located in the `environment {}` section of the `Jenkinsfile` with values from your environment:

   ```text
   _TF_SA_EMAIL
   _STATE_BUCKET_NAME
   _PROJECT_ID (the CI/CD project ID)
   ```

1. You can re-run `terraform output` in the gcp-bootstrap directory to find these values.

   ```bash
   BACKEND_STATE_BUCKET_NAME=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw gcs_bucket_tfstate)
   echo "_STATE_BUCKET_NAME = ${BACKEND_STATE_BUCKET_NAME}"
   sed -i'' -e "s/BACKEND_STATE_BUCKET_NAME/${BACKEND_STATE_BUCKET_NAME}/" ./Jenkinsfile

   TERRAFORM_SA_EMAIL=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw networks_step_terraform_service_account_email)
   echo "_TF_SA_EMAIL = ${TERRAFORM_SA_EMAIL}"
   sed -i'' -e "s/TERRAFORM_SA_EMAIL/${TERRAFORM_SA_EMAIL}/" ./Jenkinsfile

   CICD_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw cicd_project_id)
   echo "_PROJECT_ID = ${CICD_PROJECT_ID}"
   sed -i'' -e "s/CICD_PROJECT_ID/${CICD_PROJECT_ID}/" ./Jenkinsfile
   ```

1. Rename `common.auto.example.tfvars` to `common.auto.tfvars`, rename `shared.auto.example.tfvars` to `shared.auto.tfvars` and rename `access_context.auto.example.tfvars` to `access_context.auto.tfvars`.

   ```bash
   mv common.auto.example.tfvars common.auto.tfvars
   mv shared.auto.example.tfvars shared.auto.tfvars
   mv access_context.auto.example.tfvars access_context.auto.tfvars
   ```

1. Update `common.auto.tfvars` file with values from your environment and bootstrap. See any of the envs folder [README.md](../3-networks-hub-and-spoke/envs/production/README.md) files for additional information on the values in the `common.auto.tfvars` file.
1. Update `shared.auto.tfvars` file with the `target_name_server_addresses`.
1. Update `access_context.auto.tfvars` file with the `access_context_manager_policy_id`.
1. Use `terraform output` to get the backend bucket value from gcp-bootstrap output.

   ```bash
   export ORGANIZATION_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -json common_config | jq '.org_id' --raw-output)
   export ACCESS_CONTEXT_MANAGER_ID=$(gcloud access-context-manager policies list --organization ${ORGANIZATION_ID} --format="value(name)")
   echo "access_context_manager_policy_id = ${ACCESS_CONTEXT_MANAGER_ID}"
   sed -i'' -e "s/ACCESS_CONTEXT_MANAGER_ID/${ACCESS_CONTEXT_MANAGER_ID}/" ./access_context.auto.tfvars

   export backend_bucket=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${backend_bucket}"
   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" ./common.auto.tfvars
   ```

1. Commit changes.

   ```bash
   git add .
   git commit -m 'Initialize networks repo'
   ```

1. You must manually plan and apply the `shared` environment (only once) since the `development`, `nonproduction` and `production` environments depend on it.
1. To use the `validate` option of the `tf-wrapper.sh` script, please follow the [instructions](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install) to install the terraform-tools component.
1. Also update `backend.tf` with your backend bucket from gcp-bootstrap output.

   ```bash
   for i in `find . -name 'backend.tf'`; do sed -i'' -e "s/UPDATE_ME/${backend_bucket}/" $i; done
   ```

1. Use `terraform output` to get the Cloud Build project ID and the networks step Terraform Service Account from gcp-bootstrap output. An environment variable `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` will be set using the Terraform Service Account to enable impersonation.

   ```bash
   export CICD_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw cicd_project_id)
   echo ${CICD_PROJECT_ID}

   export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw networks_step_terraform_service_account_email)
   echo ${GOOGLE_IMPERSONATE_SERVICE_ACCOUNT}
   ```

1. Run `init` and `plan` and review output for environment shared.

   ```bash
   ./tf-wrapper.sh init shared
   ./tf-wrapper.sh plan shared
   ```

1. Run `validate` and check for violations.

   ```bash
   ./tf-wrapper.sh validate shared $(pwd)/policy-library ${CICD_PROJECT_ID}
   ```

1. Run `apply` shared.

   ```bash
   ./tf-wrapper.sh apply shared
   ```

1. Push your plan branch.

   ```bash
   git push --set-upstream origin plan
   ```

   - Assuming you configured an automatic trigger in your Jenkins Controller (see [Jenkins sub-module README](./modules/jenkins-agent/README.md)), this will trigger a plan. You can also trigger a Jenkins job manually. Given the many options to do this in Jenkins, it is out of the scope of this document see [Jenkins website](https://www.jenkins.io) for more details.
1. Review the plan output in your Controller's web UI.
1. Merge changes to production branch.

   ```bash
   git checkout -b production
   git push --set-upstream origin production
   ```

1. Review the apply output in your Controller's web UI (you might want to use the option to "Scan Multibranch Pipeline Now" in your Jenkins Controller UI).
1. After production has been applied, apply development and nonproduction.
1. Merge changes to development

   ```bash
   git checkout -b development
   git push --set-upstream origin development
   ```

1. Review the apply output in your Controller's web UI (you might want to use the option to "Scan Multibranch Pipeline Now" in your Jenkins Controller UI).
1. Merge changes to nonproduction.

   ```bash
   git checkout -b nonproduction
   git push --set-upstream origin nonproduction
   ```

1. Review the apply output in your Controller's web UI (you might want to use the option to "Scan Multibranch Pipeline Now" in your Jenkins Controller UI).

## Deploying step 4-projects

1. Clone the repo you created manually in 0-bootstrap.

   ```bash
   git clone <YOUR_NEW_REPO-gcp-projects> gcp-projects
   ```

1. Navigate into the repo and change to a nonproduction branch. All subsequent
   steps assume you are running them from the `gcp-projects` directory. If
   you run them from another directory, adjust your copy paths accordingly.

   ```bash
   cd gcp-projects
   git checkout -b plan
   ```

1. Copy contents of foundation to new repo.

   ```bash
   cp -RT ../terraform-example-foundation/4-projects/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   cp ../terraform-example-foundation/build/Jenkinsfile .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. Update the variables located in the `environment {}` section of the `Jenkinsfile` with values from your environment:

   ```text
   _TF_SA_EMAIL
   _STATE_BUCKET_NAME
   _PROJECT_ID (the CI/CD project ID)
   ```

1. You can re-run `terraform output` in the gcp-bootstrap directory to find these values.

   ```bash
   BACKEND_STATE_BUCKET_NAME=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw gcs_bucket_tfstate)
   echo "_STATE_BUCKET_NAME = ${BACKEND_STATE_BUCKET_NAME}"
   sed -i'' -e "s/BACKEND_STATE_BUCKET_NAME/${BACKEND_STATE_BUCKET_NAME}/" ./Jenkinsfile

   TERRAFORM_SA_EMAIL=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw projects_step_terraform_service_account_email)
   echo "_TF_SA_EMAIL = ${TERRAFORM_SA_EMAIL}"
   sed -i'' -e "s/TERRAFORM_SA_EMAIL/${TERRAFORM_SA_EMAIL}/" ./Jenkinsfile

   CICD_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw cicd_project_id)
   echo "_PROJECT_ID = ${CICD_PROJECT_ID}"
   sed -i'' -e "s/CICD_PROJECT_ID/${CICD_PROJECT_ID}/" ./Jenkinsfile
   ```

1. Rename `auto.example.tfvars` files to `auto.tfvars`.

   ```bash
   mv common.auto.example.tfvars common.auto.tfvars
   mv shared.auto.example.tfvars shared.auto.tfvars
   mv development.auto.example.tfvars development.auto.tfvars
   mv nonproduction.auto.example.tfvars nonproduction.auto.tfvars
   mv production.auto.example.tfvars production.auto.tfvars
   ```

1. See any of the envs folder [README.md](../4-projects/business_unit_1/production/README.md) files for additional information on the values in the `common.auto.tfvars`, `development.auto.tfvars`, `nonproduction.auto.tfvars`, and `production.auto.tfvars` files.
1. See any of the shared folder [README.md](../4-projects/business_unit_1/shared/README.md) files for additional information on the values in the `shared.auto.tfvars` file.
1. Use `terraform output` to get the backend bucket value from gcp-bootstrap output.

   ```bash
   export backend_bucket=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${backend_bucket}"
   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" ./common.auto.tfvars
   ```

1. (Optional) If you want additional subfolders for separate business units or entities, make additional copies of the folder `business_unit_1` and modify any values that vary across business unit like `business_code`, `business_unit`, or `subnet_ip_range`.

For example, to create a new business unit similar to business_unit_1, run the following:

   ```bash
   #copy the business_unit_1 folder and it's contents to a new folder business_unit_2
   cp -r  business_unit_1 business_unit_2

   # search all files under the folder `business_unit_2` and replace strings for business_unit_1 with strings for business_unit_2
   grep -rl bu1 business_unit_2/ | xargs sed -i 's/bu1/bu2/g'
   grep -rl business_unit_1 business_unit_2/ | xargs sed -i 's/business_unit_1/business_unit_2/g'
   ```


1. Commit changes.

   ```bash
   git add .
   git commit -m 'Initialize projects repo'
   ```

1. Also update `backend.tf` with your backend bucket from gcp-bootstrap output.

   ```bash
   for i in `find . -name 'backend.tf'`; do sed -r -i "s/UPDATE_ME|UPDATE_PROJECTS_BACKEND/${backend_bucket}/" $i; done
   ```

1. You need to manually plan and apply only once the `shared` environments since `development`, `nonproduction`, and `production` depend on it.
1. Use `terraform output` to get the Cloud Build project ID and the projects step Terraform Service Account from gcp-bootstrap output. An environment variable `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT` will be set using the Terraform Service Account to enable impersonation.

   ```bash
   export CICD_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw cicd_project_id)
   echo ${CICD_PROJECT_ID}

   export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw projects_step_terraform_service_account_email)
   echo ${GOOGLE_IMPERSONATE_SERVICE_ACCOUNT}
   ```

1. Run `init` and `plan` and review output for environments `shared`.

   ```bash
   ./tf-wrapper.sh init shared
   ./tf-wrapper.sh plan shared
   ```

1. Run `validate` and check for violations.

   ```bash
   ./tf-wrapper.sh validate shared $(pwd)/policy-library ${CICD_PROJECT_ID}
   ```

1. Run `apply` shared.

   ```bash
   ./tf-wrapper.sh apply shared
   ```

1. Push your plan branch.

   ```bash
   git push --set-upstream origin plan
   ```

   - Assuming you configured an automatic trigger in your Jenkins Controller (see [Jenkins sub-module README](./modules/jenkins-agent/README.md)), this will trigger a plan. You can also trigger a Jenkins job manually. Given the many options to do this in Jenkins, it is out of the scope of this document see [Jenkins website](https://www.jenkins.io) for more details.
1. Review the plan output in your Controller's web UI.
1. Merge changes to production branch.

   ```bash
   git checkout -b production
   git push --set-upstream origin production
   ```

1. Review the apply output in your Controller's web UI (you might want to use the option to "Scan Multibranch Pipeline Now" in your Jenkins Controller UI).
1. After production has been applied, apply development.
1. Merge changes to development branch.

   ```bash
   git checkout -b development
   git push --set-upstream origin development
   ```

1. Review the apply output in your Controller's web UI (you might want to use the option to "Scan Multibranch Pipeline Now" in your Jenkins Controller UI).
1. After development has been applied, apply nonproduction.
1. Merge changes to nonproduction branch.

   ```bash
   git checkout -b nonproduction
   git push --set-upstream origin nonproduction
   ```

1. Review the apply output in your Controller's web UI (you might want to use the option to "Scan Multibranch Pipeline Now" in your Jenkins Controller UI).

## Contributing

Refer to the [contribution guidelines](../CONTRIBUTING.md) for
information on contributing to this module.
