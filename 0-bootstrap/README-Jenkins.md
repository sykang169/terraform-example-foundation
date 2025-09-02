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

1. terraform 명령을 실행합니다.
   - 자격 증명이 구성된 후, `prj-b-seed` 프로젝트(GCS 상태 버킷 및 Terraform 사용자 지정 서비스 계정 포함)와 `prj-b-cicd` 프로젝트(Jenkins Agent, 사용자 지정 서비스 계정 포함 및 VPN 구성을 추가할 위치)를 생성합니다
   - 아래 명령으로 terraform 스크립트를 실행하려면 **Terraform 1.5.7을 사용**합니다

   ```bash
   terraform init
   terraform plan
   terraform apply
   ```

   - Terraform 스크립트는 약 10~15분이 소요됩니다. 완료되면 온프레미스와 `prj-b-cicd` 프로젝트 간의 통신이 아직 이루어지지 않는다는 점에 유의하세요. [III. VPN 연결 구성](#iii-configure-vpn-connection) 단계에서 VPN 네트워크 연결을 구성할 것입니다.

1. Seed 프로젝트에서 생성된 GCS 버킷으로 Terraform 상태 이동
   1. tfstate 버킷 이름 가져오기

   ```bash
   export backend_bucket=$(terraform output -raw gcs_bucket_tfstate)
   echo "backend_bucket = ${backend_bucket}"
   ```

   1. `backend.tf.example`을 `backend.tf`로 이름을 변경하고 tfstate 버킷 이름으로 업데이트

   ```bash
   mv backend.tf.example backend.tf
   for i in `find . -name 'backend.tf'`; do sed -i'' -e "s/UPDATE_ME/${backend_bucket}/" $i; done
   ```

1. `terraform init`을 다시 실행하고 메시지가 표시되면 gcs로 상태를 복사하는 것에 동의합니다

   ```bash
   terraform init
   ```

   - (선택사항) `terraform apply`를 실행하여 상태가 올바르게 구성되었는지 확인합니다. Seed 프로젝트의 버킷 URL을 방문하여 terraform 상태가 이제 해당 버킷에 있는지 확인할 수 있습니다.

1. 변경 사항 커밋

   ```bash
   git add backend.tf
   git commit -m 'Terraform Backend configuration using GCS'
   ```

1. 다음으로 `plan` 브랜치를 YOUR_NEW_REPO-gcp-bootstrap 저장소에 푸시합니다

   ```bash
   git push
   ```

### III. VPN 연결 구성

여기서 `prj-b-cicd` 프로젝트와 온프레미스 환경 간의 연결을 활성화하기 위해 VPN 네트워크 터널을 구성합니다. [GCP의 VPN 터널](https://cloud.google.com/network-connectivity/docs/vpn/how-to)에 대해 자세히 알아보세요.

- 필수 정보:
  - 온프레미스 VPN 공용 IP 주소
  - Jenkins Controller의 네트워크 CIDR (예시 코드에서는 "10.1.0.0/24" 사용)
  - Jenkins Agent 네트워크 CIDR (예시 코드에서는 "172.16.1.0/24" 사용)
  - VPN PSK (사전 공유 비밀 키)

1. `prj-b-cicd` 프로젝트에서 예약된 VPN 게이트웨이 정적 IP 주소를 확인합니다. 이러한 주소는 네트워크 관리자가 온프레미스 측 VPN 터널을 GCP로 구성하는 데 필요합니다.
   1. 네트워크 관리자가 이미 온프레미스 측 VPN을 구성했다고 가정하면, CI/CD 측 VPN은 약 5분 동안 `First Handshake` 메시지를 표시할 수 있습니다.
   1. VPN이 준비되면 상태가 `Tunnel is up and running`으로 표시됩니다. 이 시점에서 Jenkins Controller(온프레미스)와 Jenkins Agent(`prj-b-cicd` 프로젝트)는 VPN을 통해 네트워크 연결이 가능해야 합니다.

1. Jenkins Controller 웹 UI를 사용하여 파이프라인 테스트:
   1. [SSH Agent](https://plugins.jenkins.io/ssh-agent)가 온라인 상태인지 확인하고 필요한 경우 네트워크 연결 문제를 해결합니다.
   1. Jenkins Controller가 `prj-b-cicd` 프로젝트에 위치한 Jenkins Agent에 [파이프라인](https://www.jenkins.io/doc/book/pipeline/getting-started/)을 배포할 수 있는지 테스트합니다 (간단한 `echo "Hello World"` 파이프라인 빌드를 실행하여 테스트할 수 있음).

### IV. Jenkins Controller에서 Git 저장소 및 멀티브랜치 파이프라인 구성

- **참고:** 이 섹션은 이 문서의 범위를 벗어나는 것으로 간주됩니다. Jenkins Controller에서 Git 저장소와 **멀티브랜치 파이프라인**을 구성하는 방법에는 여러 옵션이 있기 때문에, 여기서는 이 단계를 완료하는 동안 염두에 두어야 할 몇 가지 지침만 제공할 수 있습니다. 자세한 정보는 [Jenkins 웹사이트](https://jenkins.io)를 방문하십시오. 이 작업에 도움이 될 수 있는 많은 Jenkins 플러그인이 있습니다.
  - **"멀티브랜치 파이프라인"**을 구성해야 합니다. `Jenkinsfile`과 `tf-wrapper.sh` 파일이 `$BRANCH_NAME` 환경 변수를 사용한다는 점에 주의하십시오. **`$BRANCH_NAME` 변수는 Jenkins의 멀티브랜치 파이프라인에서만 사용할 수 있습니다**.
- **Jenkinsfile:** Cloud Build 파이프라인과 밀접하게 일치하는 [Jenkinsfile](../build/Jenkinsfile)이 포함되어 있습니다. 또한 `terraform apply`를 진행하기 전에 Jenkins UI를 통해 확인할 수 있는 `TF wait for approval` 단계는 기본적으로 비활성화되어 있습니다. 파일에서 해당 단계의 주석을 해제하여 활성화할 수 있습니다.

1. 새 저장소(`YOUR_NEW_REPO-gcp-org, YOUR_NEW_REPO-gcp-environments, YOUR_NEW_REPO-gcp-networks, YOUR_NEW_REPO-gcp-projects`)에 대한 멀티브랜치 파이프라인을 생성합니다.
   - **`YOUR_NEW_REPO-gcp-bootstrap` 저장소에는 자동 파이프라인을 구성하지 마십시오**

1. Jenkins Controller 웹 UI에서 **다음 저장소에만 멀티브랜치 파이프라인을 생성합니다:**

   ```text
   YOUR_NEW_REPO-gcp-org
   YOUR_NEW_REPO-gcp-environments
   YOUR_NEW_REPO-gcp-networks
   YOUR_NEW_REPO-gcp-projects
   ```

1. 새로운 Git 저장소가 비공개라고 가정하면, 저장소에 연결할 수 있도록 Jenkins Controller 웹 UI에서 새 자격 증명을 구성해야 할 수 있습니다.

1. 저장소에 커밋할 때마다 Jenkins 웹 UI에서 파이프라인을 수동으로 실행하려는 경우가 아니라면, 각 Jenkins 멀티브랜치 파이프라인에서 자동 트리거를 구성하는 것이 좋습니다.

1. 이제 1-org 단계 지침으로 이동할 수 있습니다.

## 1-org 단계 배포

1. 0-bootstrap 지침에서 수동으로 생성한 리포지토리를 복제합니다.

   ```bash
   git clone <YOUR_NEW_REPO-gcp-org> gcp-org
   ```

1. 저장소로 이동하여 비프로덕션 브랜치로 변경합니다. 모든 후속 단계는 `gcp-org` 디렉토리에서 실행한다고 가정합니다. 다른 디렉토리에서 실행하는 경우, 복사 경로를 적절히 조정하세요.

   ```bash
   cd gcp-org
   git checkout -b plan
   ```

1. foundation의 내용을 새 저장소로 복사합니다.

   ```bash
   cp -RT ../terraform-example-foundation/1-org/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   cp ../terraform-example-foundation/build/Jenkinsfile .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. `Jenkinsfile`의 `environment {}` 섹션에 있는 변수를 gcp-bootstrap의 값으로 업데이트합니다:

   ```text
   _TF_SA_EMAIL
   _STATE_BUCKET_NAME
   _PROJECT_ID (the CI/CD project ID)
   ```

1. gcp-bootstrap 디렉토리에서 `terraform output`을 다시 실행하여 이러한 값들을 찾을 수 있습니다.

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

1. `./envs/shared/terraform.example.tfvars`를 `./envs/shared/terraform.tfvars`로 이름 변경

   ```bash
   mv ./envs/shared/terraform.example.tfvars ./envs/shared/terraform.tfvars
   ```

1. 기본 이름인 **scc-notify**로 된 보안 명령 센터 알림이 이미 존재하는지 확인합니다. 존재하는 경우, `./envs/shared/terraform.tfvars` 파일의 `scc_notification_name` 변수에 다른 값을 선택합니다.

   ```bash
   export ORGANIZATION_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -json common_config | jq '.org_id' --raw-output)
   gcloud scc notifications describe "scc-notify" --organization=${ORGANIZATION_ID}
   ```

1. 조직에 액세스 컨텍스트 관리자 정책이 이미 있는지 확인합니다.

   ```bash
   export ACCESS_CONTEXT_MANAGER_ID=$(gcloud access-context-manager policies list --organization ${ORGANIZATION_ID} --format="value(name)")
   echo "access_context_manager_policy_id = ${ACCESS_CONTEXT_MANAGER_ID}"
   ```

1. 환경과 0-bootstrap 단계의 값으로 `envs/shared/terraform.tfvars` 파일을 업데이트합니다. 이전 단계에서 숫자 값이 표시된 경우, `create_access_context_manager_access_policy = false` 변수의 주석을 해제해야 합니다. `terraform.tfvars` 파일의 값에 대한 추가 정보는 shared 폴더의 [README.md](../1-org/envs/shared/README.md)를 참조하십시오.

   ```bash
   export backend_bucket=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${backend_bucket}"

   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" ./envs/shared/terraform.tfvars

   if [ ! -z "${ACCESS_CONTEXT_MANAGER_ID}" ]; then sed -i'' -e "s=//create_access_context_manager_access_policy=create_access_context_manager_access_policy=" ./envs/shared/terraform.tfvars; fi
   ```

1. 변경 사항을 커밋합니다.

   ```bash
   git add .
   git commit -m 'Initialize org repo'
   ```

1. plan 브랜치를 푸시합니다.
   - Jenkins Controller에서 자동 트리거를 구성했다고 가정하면([Jenkins 하위 모듈 README](./modules/jenkins-agent/README.md) 참조), 이것이 plan을 트리거합니다. Jenkins 작업을 수동으로 트리거할 수도 있습니다. Jenkins에서 이를 수행하는 옵션이 많으므로 이 문서의 범위를 벗어나며, 자세한 내용은 [Jenkins 웹사이트](https://www.jenkins.io)를 참조하십시오.

   ```bash
   git push --set-upstream origin plan
   ```

1. Controller의 웹 UI에서 플랜 출력을 검토합니다.
1. 변경 사항을 프로덕션 브랜치에 병합합니다.

   ```bash
   git checkout -b production
   git push --set-upstream origin production
   ```

1. Controller의 웹 UI에서 적용 출력을 검토합니다. (Jenkins Controller UI에서 "Scan Multibranch Pipeline Now" 옵션을 사용하는 것이 좋을 수 있습니다.)

## 2-environments 단계 배포

1. 0-bootstrap에서 수동으로 생성한 저장소를 복제합니다.

   ```bash
   git clone <YOUR_NEW_REPO-gcp-environments> gcp-environments
   ```

1. 저장소로 이동하여 비프로덕션 브랜치로 변경합니다. 모든 후속 단계는 `gcp-environments` 디렉토리에서 실행한다고 가정합니다. 다른 디렉토리에서 실행하는 경우, 복사 경로를 적절히 조정하세요.

   ```bash
   cd gcp-environments
   git checkout -b plan
   ```

1. foundation의 내용을 새 저장소로 복사합니다.

   ```bash
   cp -RT ../terraform-example-foundation/2-environments/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   cp ../terraform-example-foundation/build/Jenkinsfile .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. `Jenkinsfile`의 `environment {}` 섹션에 있는 변수를 환경의 값으로 업데이트합니다:

   ```text
   _TF_SA_EMAIL
   _STATE_BUCKET_NAME
   _PROJECT_ID (the CI/CD project ID)
   ```

1. gcp-bootstrap 디렉토리에서 `terraform output`을 다시 실행하여 이러한 값들을 찾을 수 있습니다.

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

1. `terraform.example.tfvars`를 `terraform.tfvars`로 이름을 변경하고 환경과 0-bootstrap의 값으로 파일을 업데이트합니다.

   ```bash
   mv terraform.example.tfvars terraform.tfvars
   ```

1. gcp-bootstrap 디렉토리에서 `terraform output`을 다시 실행하여 이러한 값들을 찾을 수 있습니다. `terraform.tfvars` 파일의 값에 대한 추가 정보는 envs 폴더의 [README.md](../2-environments/envs/production/README.md) 파일들을 참조하십시오.

   ```bash
   export backend_bucket=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${backend_bucket}"
   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" ./terraform.tfvars
   ```

1. 변경 사항을 커밋합니다.

   ```bash
   git add .
   git commit -m 'Initialize environments repo'
   ```

1. plan 브랜치를 푸시합니다.

   ```bash
   git push --set-upstream origin plan
   ```

   - Jenkins Controller에서 자동 트리거를 구성했다고 가정하면([Jenkins 하위 모듈 README](./modules/jenkins-agent/README.md) 참조), 이것이 plan을 트리거합니다. Jenkins 작업을 수동으로 트리거할 수도 있습니다. Jenkins에서 이를 수행하는 옵션이 많으므로 이 문서의 범위를 벗어나며, 자세한 내용은 [Jenkins 웹사이트](https://www.jenkins.io)를 참조하십시오.
1. Controller의 웹 UI에서 플랜 출력을 검토합니다.
1. 변경 사항을 development에 병합합니다.

   ```bash
   git checkout -b development
   git push --set-upstream origin development
   ```

1. Controller의 웹 UI에서 적용 출력을 검토합니다. (Jenkins Controller UI에서 "Scan Multibranch Pipeline Now" 옵션을 사용하는 것이 좋을 수 있습니다.)
1. 변경 사항을 nonproduction에 병합합니다.

   ```bash
   git checkout -b nonproduction
   git push --set-upstream origin nonproduction
   ```

1. Controller의 웹 UI에서 적용 출력을 검토합니다. (Jenkins Controller UI에서 "Scan Multibranch Pipeline Now" 옵션을 사용하는 것이 좋을 수 있습니다.)
1. 변경 사항을 프로덕션 브랜치에 병합합니다.

   ```bash
   git checkout -b production
   git push --set-upstream origin production
   ```

1. Controller의 웹 UI에서 적용 출력을 검토합니다. (Jenkins Controller UI에서 "Scan Multibranch Pipeline Now" 옵션을 사용하는 것이 좋을 수 있습니다.)
1. 이제 다음 단계의 지침으로 이동할 수 있습니다. 듀얼 공유 VPC 모드를 사용하려면 [3-networks-svpc 단계 배포](#deploying-step-3-networks-svpc)로 이동하거나, 허브 앤 스포크 네트워크 모드를 사용하려면 [3-networks-hub-and-spoke 단계 배포](#deploying-step-3-networks-hub-and-spoke)로 이동하십시오.

## 3-networks-svpc 단계 배포

1. 0-bootstrap에서 수동으로 생성한 저장소를 복제합니다.

   ```bash
   git clone <YOUR_NEW_REPO-gcp-networks> gcp-networks
   ```

1. 저장소로 이동하여 비프로덕션 브랜치로 변경합니다. 모든 후속 단계는 `gcp-networks` 디렉토리에서 실행한다고 가정합니다. 다른 디렉토리에서 실행하는 경우, 복사 경로를 적절히 조정하세요.

   ```bash
   cd gcp-networks
   git checkout -b plan
   ```

1. foundation의 내용을 새 저장소로 복사합니다.

   ```bash
   cp -RT ../terraform-example-foundation/3-networks-svpc/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   cp ../terraform-example-foundation/build/Jenkinsfile .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. `Jenkinsfile`의 `environment {}` 섹션에 있는 변수를 환경의 값으로 업데이트합니다:

   ```text
   _TF_SA_EMAIL
   _STATE_BUCKET_NAME
   _PROJECT_ID (the CI/CD project ID)
   ```

1. gcp-bootstrap 디렉토리에서 `terraform output`을 다시 실행하여 이러한 값들을 찾을 수 있습니다.

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

1. `common.auto.example.tfvars`를 `common.auto.tfvars`로, `production.auto.example.tfvars`를 `production.auto.tfvars`로, `access_context.auto.example.tfvars`를 `access_context.auto.tfvars`로 이름을 변경합니다.

   ```bash
   mv common.auto.example.tfvars common.auto.tfvars
   mv production.auto.example.tfvars production.auto.tfvars
   mv access_context.auto.example.tfvars access_context.auto.tfvars
   ```

1. 환경과 bootstrap의 값으로 `common.auto.tfvars` 파일을 업데이트합니다. `common.auto.tfvars` 파일의 값에 대한 추가 정보는 envs 폴더의 [README.md](../3-networks-svpc/envs/production/README.md) 파일들을 참조하십시오.
1. `target_name_server_addresses`로 `production.auto.tfvars` 파일을 업데이트합니다.
1. `access_context_manager_policy_id`로 `access_context.auto.tfvars` 파일을 업데이트합니다.
1. `terraform output`을 사용하여 gcp-bootstrap 출력에서 백엔드 버킷과 네트워크 단계 Terraform 서비스 계정 값을 가져옵니다.

   ```bash
   export ORGANIZATION_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -json common_config | jq '.org_id' --raw-output)
   export ACCESS_CONTEXT_MANAGER_ID=$(gcloud access-context-manager policies list --organization ${ORGANIZATION_ID} --format="value(name)")
   echo "access_context_manager_policy_id = ${ACCESS_CONTEXT_MANAGER_ID}"
   sed -i'' -e "s/ACCESS_CONTEXT_MANAGER_ID/${ACCESS_CONTEXT_MANAGER_ID}/" ./access_context.auto.tfvars

   export backend_bucket=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${backend_bucket}"
   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" ./common.auto.tfvars
   ```

1. 변경 사항을 커밋합니다.

   ```bash
   git add .
   git commit -m 'Initialize networks repo'
   ```

1. `development`, `nonproduction`, `production` 환경이 이에 종속되므로 `shared` 환경을 수동으로 계획하고 적용해야 합니다(한 번만).
1. `tf-wrapper.sh` 스크립트의 `validate` 옵션을 사용하려면, [지침](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install)을 따라 terraform-tools 구성 요소를 설치하세요.
1. gcp-bootstrap 출력에서 백엔드 버킷으로 `backend.tf`도 업데이트합니다.

   ```bash
   for i in `find . -name 'backend.tf'`; do sed -i'' -e "s/UPDATE_ME/${backend_bucket}/" $i; done
   ```

1. `terraform output`을 사용하여 gcp-bootstrap 출력에서 Cloud Build 프로젝트 ID와 네트워크 단계 Terraform 서비스 계정을 가져옵니다. 가장을 활성화하기 위해 Terraform 서비스 계정을 사용하여 환경 변수 `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT`가 설정됩니다.

   ```bash
   export CICD_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw cicd_project_id)
   echo ${CICD_PROJECT_ID}

   export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw networks_step_terraform_service_account_email)
   echo ${GOOGLE_IMPERSONATE_SERVICE_ACCOUNT}
   ```

1. `init`과 `plan`을 실행하고 shared 환경의 출력을 검토합니다.

   ```bash
   ./tf-wrapper.sh init shared
   ./tf-wrapper.sh plan shared
   ```

1. `validate`를 실행하고 위반 사항을 확인합니다.

   ```bash
   ./tf-wrapper.sh validate shared $(pwd)/policy-library ${CICD_PROJECT_ID}
   ```

1. `apply` shared를 실행합니다.

   ```bash
   ./tf-wrapper.sh apply shared
   ```

1. plan 브랜치를 푸시합니다.

   ```bash
   git push --set-upstream origin plan
   ```

   - Jenkins Controller에서 자동 트리거를 구성했다고 가정하면([Jenkins 하위 모듈 README](./modules/jenkins-agent/README.md) 참조), 이것이 plan을 트리거합니다. Jenkins 작업을 수동으로 트리거할 수도 있습니다. Jenkins에서 이를 수행하는 옵션이 많으므로 이 문서의 범위를 벗어나며, 자세한 내용은 [Jenkins 웹사이트](https://www.jenkins.io)를 참조하십시오.
1. Controller의 웹 UI에서 플랜 출력을 검토합니다.
1. 변경 사항을 프로덕션 브랜치에 병합합니다.

   ```bash
   git checkout -b production
   git push --set-upstream origin production
   ```

1. Controller의 웹 UI에서 적용 출력을 검토합니다. (Jenkins Controller UI에서 "Scan Multibranch Pipeline Now" 옵션을 사용하는 것이 좋을 수 있습니다.)
1. 프로덕션이 적용된 후, development와 nonproduction을 적용합니다.
1. 변경 사항을 development에 병합합니다.

   ```bash
   git checkout -b development
   git push --set-upstream origin development
   ```

1. Controller의 웹 UI에서 적용 출력을 검토합니다. (Jenkins Controller UI에서 "Scan Multibranch Pipeline Now" 옵션을 사용하는 것이 좋을 수 있습니다.)
1. 변경 사항을 nonproduction에 병합합니다.

   ```bash
   git checkout -b nonproduction
   git push --set-upstream origin nonproduction
   ```

1. Controller의 웹 UI에서 적용 출력을 검토합니다. (Jenkins Controller UI에서 "Scan Multibranch Pipeline Now" 옵션을 사용하는 것이 좋을 수 있습니다.)

## 3-networks-hub-and-spoke 단계 배포

1. 0-bootstrap에서 수동으로 생성한 저장소를 복제합니다.

   ```bash
   git clone <YOUR_NEW_REPO-gcp-networks> gcp-networks
   ```

1. 저장소로 이동하여 비프로덕션 브랜치로 변경합니다. 모든 후속 단계는 `gcp-networks` 디렉토리에서 실행한다고 가정합니다. 다른 디렉토리에서 실행하는 경우, 복사 경로를 적절히 조정하세요.

   ```bash
   cd gcp-networks
   git checkout -b plan
   ```

1. foundation의 내용을 새 저장소로 복사합니다.

   ```bash
   cp -RT ../terraform-example-foundation/3-networks-hub-and-spoke/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   cp ../terraform-example-foundation/build/Jenkinsfile .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. `Jenkinsfile`의 `environment {}` 섹션에 있는 변수를 환경의 값으로 업데이트합니다:

   ```text
   _TF_SA_EMAIL
   _STATE_BUCKET_NAME
   _PROJECT_ID (the CI/CD project ID)
   ```

1. gcp-bootstrap 디렉토리에서 `terraform output`을 다시 실행하여 이러한 값들을 찾을 수 있습니다.

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

1. `common.auto.example.tfvars`를 `common.auto.tfvars`로, `shared.auto.example.tfvars`를 `shared.auto.tfvars`로, `access_context.auto.example.tfvars`를 `access_context.auto.tfvars`로 이름을 변경합니다.

   ```bash
   mv common.auto.example.tfvars common.auto.tfvars
   mv shared.auto.example.tfvars shared.auto.tfvars
   mv access_context.auto.example.tfvars access_context.auto.tfvars
   ```

1. 환경과 bootstrap의 값으로 `common.auto.tfvars` 파일을 업데이트합니다. `common.auto.tfvars` 파일의 값에 대한 추가 정보는 envs 폴더의 [README.md](../3-networks-hub-and-spoke/envs/production/README.md) 파일들을 참조하십시오.
1. `target_name_server_addresses`로 `shared.auto.tfvars` 파일을 업데이트합니다.
1. `access_context_manager_policy_id`로 `access_context.auto.tfvars` 파일을 업데이트합니다.
1. `terraform output`을 사용하여 gcp-bootstrap 출력에서 백엔드 버킷 값을 가져옵니다.

   ```bash
   export ORGANIZATION_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -json common_config | jq '.org_id' --raw-output)
   export ACCESS_CONTEXT_MANAGER_ID=$(gcloud access-context-manager policies list --organization ${ORGANIZATION_ID} --format="value(name)")
   echo "access_context_manager_policy_id = ${ACCESS_CONTEXT_MANAGER_ID}"
   sed -i'' -e "s/ACCESS_CONTEXT_MANAGER_ID/${ACCESS_CONTEXT_MANAGER_ID}/" ./access_context.auto.tfvars

   export backend_bucket=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${backend_bucket}"
   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" ./common.auto.tfvars
   ```

1. 변경 사항을 커밋합니다.

   ```bash
   git add .
   git commit -m 'Initialize networks repo'
   ```

1. `development`, `nonproduction`, `production` 환경이 이에 종속되므로 `shared` 환경을 수동으로 계획하고 적용해야 합니다(한 번만).
1. `tf-wrapper.sh` 스크립트의 `validate` 옵션을 사용하려면, [지침](https://cloud.google.com/docs/terraform/policy-validation/validate-policies#install)을 따라 terraform-tools 구성 요소를 설치하세요.
1. gcp-bootstrap 출력에서 백엔드 버킷으로 `backend.tf`도 업데이트합니다.

   ```bash
   for i in `find . -name 'backend.tf'`; do sed -i'' -e "s/UPDATE_ME/${backend_bucket}/" $i; done
   ```

1. `terraform output`을 사용하여 gcp-bootstrap 출력에서 Cloud Build 프로젝트 ID와 네트워크 단계 Terraform 서비스 계정을 가져옵니다. 가장을 활성화하기 위해 Terraform 서비스 계정을 사용하여 환경 변수 `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT`가 설정됩니다.

   ```bash
   export CICD_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw cicd_project_id)
   echo ${CICD_PROJECT_ID}

   export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw networks_step_terraform_service_account_email)
   echo ${GOOGLE_IMPERSONATE_SERVICE_ACCOUNT}
   ```

1. `init`과 `plan`을 실행하고 shared 환경의 출력을 검토합니다.

   ```bash
   ./tf-wrapper.sh init shared
   ./tf-wrapper.sh plan shared
   ```

1. `validate`를 실행하고 위반 사항을 확인합니다.

   ```bash
   ./tf-wrapper.sh validate shared $(pwd)/policy-library ${CICD_PROJECT_ID}
   ```

1. `apply` shared를 실행합니다.

   ```bash
   ./tf-wrapper.sh apply shared
   ```

1. plan 브랜치를 푸시합니다.

   ```bash
   git push --set-upstream origin plan
   ```

   - Jenkins Controller에서 자동 트리거를 구성했다고 가정하면([Jenkins 하위 모듈 README](./modules/jenkins-agent/README.md) 참조), 이것이 plan을 트리거합니다. Jenkins 작업을 수동으로 트리거할 수도 있습니다. Jenkins에서 이를 수행하는 옵션이 많으므로 이 문서의 범위를 벗어나며, 자세한 내용은 [Jenkins 웹사이트](https://www.jenkins.io)를 참조하십시오.
1. Controller의 웹 UI에서 플랜 출력을 검토합니다.
1. 변경 사항을 프로덕션 브랜치에 병합합니다.

   ```bash
   git checkout -b production
   git push --set-upstream origin production
   ```

1. Controller의 웹 UI에서 적용 출력을 검토합니다. (Jenkins Controller UI에서 "Scan Multibranch Pipeline Now" 옵션을 사용하는 것이 좋을 수 있습니다.)
1. 프로덕션이 적용된 후, development와 nonproduction을 적용합니다.
1. 변경 사항을 development에 병합합니다.

   ```bash
   git checkout -b development
   git push --set-upstream origin development
   ```

1. Controller의 웹 UI에서 적용 출력을 검토합니다. (Jenkins Controller UI에서 "Scan Multibranch Pipeline Now" 옵션을 사용하는 것이 좋을 수 있습니다.)
1. 변경 사항을 nonproduction에 병합합니다.

   ```bash
   git checkout -b nonproduction
   git push --set-upstream origin nonproduction
   ```

1. Controller의 웹 UI에서 적용 출력을 검토합니다. (Jenkins Controller UI에서 "Scan Multibranch Pipeline Now" 옵션을 사용하는 것이 좋을 수 있습니다.)

## 4-projects 단계 배포

1. 0-bootstrap에서 수동으로 생성한 저장소를 복제합니다.

   ```bash
   git clone <YOUR_NEW_REPO-gcp-projects> gcp-projects
   ```

1. 저장소로 이동하여 비프로덕션 브랜치로 변경합니다. 모든 후속 단계는 `gcp-projects` 디렉토리에서 실행한다고 가정합니다. 다른 디렉토리에서 실행하는 경우, 복사 경로를 적절히 조정하세요.

   ```bash
   cd gcp-projects
   git checkout -b plan
   ```

1. foundation의 내용을 새 저장소로 복사합니다.

   ```bash
   cp -RT ../terraform-example-foundation/4-projects/ .
   cp -RT ../terraform-example-foundation/policy-library/ ./policy-library
   cp ../terraform-example-foundation/build/Jenkinsfile .
   cp ../terraform-example-foundation/build/tf-wrapper.sh .
   chmod 755 ./tf-wrapper.sh
   ```

1. `Jenkinsfile`의 `environment {}` 섹션에 있는 변수를 환경의 값으로 업데이트합니다:

   ```text
   _TF_SA_EMAIL
   _STATE_BUCKET_NAME
   _PROJECT_ID (the CI/CD project ID)
   ```

1. gcp-bootstrap 디렉토리에서 `terraform output`을 다시 실행하여 이러한 값들을 찾을 수 있습니다.

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

1. `auto.example.tfvars` 파일들을 `auto.tfvars`로 이름을 변경합니다.

   ```bash
   mv common.auto.example.tfvars common.auto.tfvars
   mv shared.auto.example.tfvars shared.auto.tfvars
   mv development.auto.example.tfvars development.auto.tfvars
   mv nonproduction.auto.example.tfvars nonproduction.auto.tfvars
   mv production.auto.example.tfvars production.auto.tfvars
   ```

1. `common.auto.tfvars`, `development.auto.tfvars`, `nonproduction.auto.tfvars`, `production.auto.tfvars` 파일의 값에 대한 추가 정보는 envs 폴더의 [README.md](../4-projects/business_unit_1/production/README.md) 파일들을 참조하세요.
1. `shared.auto.tfvars` 파일의 값에 대한 추가 정보는 shared 폴더의 [README.md](../4-projects/business_unit_1/shared/README.md) 파일들을 참조하세요.
1. `terraform output`을 사용하여 gcp-bootstrap 출력에서 백엔드 버킷 값을 가져옵니다.

   ```bash
   export backend_bucket=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw gcs_bucket_tfstate)
   echo "remote_state_bucket = ${backend_bucket}"
   sed -i'' -e "s/REMOTE_STATE_BUCKET/${backend_bucket}/" ./common.auto.tfvars
   ```

1. (Optional) If you want additional subfolders for separate business units or entities, make additional copies of the folder `business_unit_1` and modify any values that vary across business unit like `business_code`, `business_unit`, or `subnet_ip_range`.

예를 들어, business_unit_1과 비슷한 새로운 비즈니스 유닛을 생성하려면 다음을 실행하세요:

   ```bash
   # business_unit_1 폴더와 그 내용을 새로운 폴더 business_unit_2로 복사
   cp -r  business_unit_1 business_unit_2

   # `business_unit_2` 폴더 아래의 모든 파일을 검색하여 business_unit_1의 문자열을 business_unit_2의 문자열로 대체
   grep -rl bu1 business_unit_2/ | xargs sed -i 's/bu1/bu2/g'
   grep -rl business_unit_1 business_unit_2/ | xargs sed -i 's/business_unit_1/business_unit_2/g'
   ```


1. 변경 사항을 커밋합니다.

   ```bash
   git add .
   git commit -m 'Initialize projects repo'
   ```

1. gcp-bootstrap 출력에서 백엔드 버킷으로 `backend.tf`도 업데이트합니다.

   ```bash
   for i in `find . -name 'backend.tf'`; do sed -r -i "s/UPDATE_ME|UPDATE_PROJECTS_BACKEND/${backend_bucket}/" $i; done
   ```

1. `development`, `nonproduction`, `production`이 이에 종속되므로 `shared` 환경을 한 번만 수동으로 계획하고 적용해야 합니다.
1. `terraform output`을 사용하여 gcp-bootstrap 출력에서 Cloud Build 프로젝트 ID와 projects 단계 Terraform 서비스 계정을 가져옵니다. 가장을 활성화하기 위해 Terraform 서비스 계정을 사용하여 환경 변수 `GOOGLE_IMPERSONATE_SERVICE_ACCOUNT`가 설정됩니다.

   ```bash
   export CICD_PROJECT_ID=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw cicd_project_id)
   echo ${CICD_PROJECT_ID}

   export GOOGLE_IMPERSONATE_SERVICE_ACCOUNT=$(terraform -chdir="../gcp-bootstrap/envs/shared" output -raw projects_step_terraform_service_account_email)
   echo ${GOOGLE_IMPERSONATE_SERVICE_ACCOUNT}
   ```

1. `shared` 환경에 대해 `init`과 `plan`을 실행하고 출력을 검토합니다.

   ```bash
   ./tf-wrapper.sh init shared
   ./tf-wrapper.sh plan shared
   ```

1. `validate`를 실행하고 위반 사항을 확인합니다.

   ```bash
   ./tf-wrapper.sh validate shared $(pwd)/policy-library ${CICD_PROJECT_ID}
   ```

1. `apply` shared를 실행합니다.

   ```bash
   ./tf-wrapper.sh apply shared
   ```

1. plan 브랜치를 푸시합니다.

   ```bash
   git push --set-upstream origin plan
   ```

   - Jenkins Controller에서 자동 트리거를 구성했다고 가정하면([Jenkins 하위 모듈 README](./modules/jenkins-agent/README.md) 참조), 이것이 plan을 트리거합니다. Jenkins 작업을 수동으로 트리거할 수도 있습니다. Jenkins에서 이를 수행하는 옵션이 많으므로 이 문서의 범위를 벗어나며, 자세한 내용은 [Jenkins 웹사이트](https://www.jenkins.io)를 참조하십시오.
1. Controller의 웹 UI에서 플랜 출력을 검토합니다.
1. 변경 사항을 프로덕션 브랜치에 병합합니다.

   ```bash
   git checkout -b production
   git push --set-upstream origin production
   ```

1. Controller의 웹 UI에서 적용 출력을 검토합니다. (Jenkins Controller UI에서 "Scan Multibranch Pipeline Now" 옵션을 사용하는 것이 좋을 수 있습니다.)
1. 프로덕션이 적용된 후, development를 적용합니다.
1. 변경 사항을 development 브랜치에 병합합니다.

   ```bash
   git checkout -b development
   git push --set-upstream origin development
   ```

1. Controller의 웹 UI에서 적용 출력을 검토합니다. (Jenkins Controller UI에서 "Scan Multibranch Pipeline Now" 옵션을 사용하는 것이 좋을 수 있습니다.)
1. development가 적용된 후, nonproduction을 적용합니다.
1. 변경 사항을 nonproduction 브랜치에 병합합니다.

   ```bash
   git checkout -b nonproduction
   git push --set-upstream origin nonproduction
   ```

1. Controller의 웹 UI에서 적용 출력을 검토합니다. (Jenkins Controller UI에서 "Scan Multibranch Pipeline Now" 옵션을 사용하는 것이 좋을 수 있습니다.)

## 기여

이 모듈에 기여하는 방법에 대한 정보는 [기여 가이드라인](../CONTRIBUTING.md)을 참조하십시오.
