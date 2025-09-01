# terraform-example-foundation

이 예제 리포지토리는 CFT Terraform 모듈이 [Google Cloud Enterprise Foundations Blueprint](https://cloud.google.com/architecture/security-foundations)(이전 명칭: _Security Foundations Guide_)를 따라 안전한 Google Cloud 기반을 구축하는 방법을 보여줍니다.
제공된 구조와 코드는 여러분의 요구사항에 맞게 사용자 정의할 수 있는 실용적인 기본값으로 자체 기반을 구축하기 위한 시작점을 형성하도록 의도되었습니다.

이 블루프린트의 대상 사용자는 GCP 환경을 배포하고 유지 관리하는 전담 플랫폼 팀이 있는 대규모 엔터프라이즈 조직으로, 여러 팀 간의 역할 분리를 수행하고 버전 관리된 Infrastructure as Code를 통해서만 환경을 관리하는 데 전념하는 조직입니다. 턴키 솔루션을 찾는 소규모 조직은 [Google Cloud Setup](https://console.cloud.google.com/cloud-setup/overview)과 같은 다른 옵션을 선호할 수 있습니다

## 사용 목적 및 지원

이 리포지토리는 포크하고, 수정하고, 사용자 자체 버전 관리 시스템에서 유지 관리할 예제로 의도되었습니다. 이 리포지토리 내의 모듈은 원격 참조용으로 사용하도록 의도되지 않았습니다.
이 블루프린트가 기반 설계 및 구축을 가속화하는 데 도움이 될 수 있지만, 자체 요구사항에 따라 자체 기반을 배포하고 사용자 정의할 수 있는 엔지니어링 기술과 팀이 있다고 가정합니다.

지원 사항:
 - 코드가 의미론적으로 유효하고, 알려진 양호한 버전에 고정되며, terraform validate 및 lint 검사를 통과함
 - 이 리포지토리에 대한 모든 PR은 병합되기 전에 테스트 환경에 모든 리소스를 배포하는 통합 테스트를 통과해야 함
 - 코드 사용 편의성에 대한 기능 요청 또는 모든 사용자에게 일반적으로 적용되는 기능 요청 환영

지원하지 않는 사항:
 - 이전 버전으로 배포된 기반에서 최신 버전으로의 현재 위치 업그레이드는 사소한 버전 변경의 경우에도 실행 가능하지 않을 수 있습니다. 리포지토리 관리자는 사용자가 기반 위에 배포하는 리소스나 배포 시 기반이 어떻게 사용자 정의되었는지에 대한 가시성이 없으므로 주요 변경 사항을 피하는 것에 대해 보장하지 않습니다.
 - 단일 사용자의 요구사항에 특정되고 일반적인 모범 사례를 대표하지 않는 기능 요청

## 개요

이 리포지토리에는 각각 별도로 적용해야 하지만 순서대로 적용해야 하는 고유한 디렉토리 내에 여러 개의 개별 Terraform 프로젝트가 포함되어 있습니다.
`0-bootstrap` 단계는 수동으로 실행되며, 후속 단계는 선호하는 CI/CD 도구를 사용하여 실행됩니다.

이러한 각 Terraform 프로젝트는 서로 계층화되어야 하며 다음 순서로 실행됩니다.

### [0. bootstrap](./0-bootstrap/)

이 단계는 기존 Google Cloud 조직을 부트스트랩하는 [CFT Bootstrap 모듈](https://github.com/terraform-google-modules/terraform-google-bootstrap)을 실행하여 Cloud Foundation Toolkit (CFT)를 사용하기 시작하는 데 필요한 모든 Google Cloud 리소스와 권한을 생성합니다.
[CI/CD 파이프라인](/docs/GLOSSARY.md#foundation-cicd-pipeline)의 경우 Cloud Build(기본값) 또는 Jenkins를 사용할 수 있습니다. Cloud Build 대신 Jenkins를 사용하려면 Jenkins 하위 모듈 사용 방법에 대한 [README-Jenkins](./0-bootstrap/README-Jenkins.md)를 참조하세요.

부트스트랩 단계에는 다음이 포함됩니다:

- 다음을 포함하는 `prj-b-seed` 프로젝트:
  - Terraform 상태 버킷
  - Google Cloud에서 새 리소스를 생성하기 위해 Terraform이 사용하는 사용자 정의 서비스 계정
- 다음을 포함하는 `prj-b-cicd` 프로젝트:
  - Cloud Build 또는 Jenkins로 구현된 [CI/CD 파이프라인](/docs/GLOSSARY.md#foundation-cicd-pipeline)
  - Cloud Build 사용 시 다음 항목:
    - Cloud Source Repository
    - Artifact Registry
  - Jenkins 사용 시 다음 항목:
    - Jenkins Agent로 구성된 Compute Engine 인스턴스
    - Jenkins Agent용 Compute Engine 인스턴스를 실행하기 위한 사용자 정의 서비스 계정
    - 온프레미스(또는 Jenkins Controller가 위치한 곳)와의 VPN 연결

여기에 두 개의 프로젝트를 두어 관심사를 분리하는 것이 모범 사례입니다: 하나는 Terraform 상태용이고 하나는 CI/CD 도구용입니다.
  - `prj-b-seed` 프로젝트는 Terraform 상태를 저장하고 인프라를 생성하거나 수정할 수 있는 서비스 계정을 보유합니다.
  - `prj-b-cicd` 프로젝트는 인프라 배포를 조정하는 CI/CD 도구(Cloud Build 또는 Jenkins)를 보유합니다.

IAM 수준에서도 관심사를 더욱 분리하기 위해 각 단계마다 별도의 서비스 계정이 생성됩니다. Terraform 사용자 정의 서비스 계정에는 기반을 구축하는 데 필요한 IAM 권한이 부여됩니다.
Cloud Build를 CI/CD 도구로 사용하는 경우 이러한 서비스 계정은 파이프라인 단계(`plan` 또는 `apply`)를 실행하기 위해 파이프라인에서 직접 사용됩니다.
이 구성에서는 CI/CD 도구의 기본 권한이 변경되지 않습니다.

Jenkins를 CI/CD 도구로 사용하는 경우, Jenkins Agent의 서비스 계정(`sa-jenkins-agent-gce@prj-b-cicd-xxxx.iam.gserviceaccount.com`)에 Terraform 사용자 정의 서비스 계정에 대한 토큰을 생성할 수 있도록 [가장](https://cloud.google.com/iam/docs/create-short-lived-credentials-direct) 액세스 권한이 부여됩니다.
이 구성에서는 CI/CD 도구의 기본 권한이 제한됩니다.

이 단계를 실행한 후 다음과 같은 구조를 갖게 됩니다:

```
example-organization/
└── fldr-bootstrap
    ├── prj-b-cicd
    └── prj-b-seed
```

이 단계에서 Cloud Build 하위 모듈을 사용하면 아래 각 단계에 대한 Cloud Build 및 Cloud Source Repositories로 cicd 프로젝트(`prj-b-cicd`)를 설정합니다.
트리거는 환경이 아닌 브랜치에 대해 `terraform plan`을 실행하고 변경 사항이 환경 브랜치(`development`, `nonproduction` 또는 `production`)에 병합될 때 `terraform apply`를 실행하도록 구성됩니다.
사용 지침은 0-bootstrap [README](./0-bootstrap/README.md)에서 확인할 수 있습니다.

### [1. org](./1-org/)

이 단계의 목적은 Security Command Center 알림, Cloud Key Management Service (KMS), 조직 수준 시크릿 및 조직 수준 로깅과 같은 공유 리소스를 포함하는 프로젝트를 수용하는 데 사용되는 공통 폴더를 설정하는 것입니다.
이 단계에서는 DNS Hub, Interconnect, 네트워크 허브 및 각 환경(`development`, `nonproduction` 또는 `production`)에 대한 프로젝트와 같은 네트워크 관련 프로젝트를 수용하는 데 사용되는 네트워크 폴더도 설정합니다.
다음과 같은 폴더 및 프로젝트 구조가 생성됩니다:

```
example-organization
└── fldr-common
    ├── prj-c-logging
    ├── prj-c-billing-export
    ├── prj-c-scc
    ├── prj-c-kms
    └── prj-c-secrets
└── fldr-network
    ├── prj-net-hub-svpc
    ├── prj-net-dns
    ├── prj-net-interconnect
    ├── prj-d-svpc
    ├── prj-n-svpc
    └── prj-p-svpc
```

#### 로그

공통 폴더 아래에 `prj-c-logging` 프로젝트가 조직 전체 싱크의 대상으로 사용됩니다. 여기에는 조직의 모든 프로젝트와 청구 계정의 관리자 활동 감사 로그가 포함됩니다.

로그는 연결된 BigQuery 데이터셋이 있는 로깅 버킷에 수집되며, 임시 로그 조사, 쿼리 또는 보고에 사용할 수 있습니다. 로그 싱크는 외부 시스템으로 내보내기 위한 Pub/Sub 또는 장기 저장을 위한 Cloud Storage로 내보내도록 구성할 수도 있습니다.

**참고사항**:

- Cloud Storage 버킷으로의 로그 내보내기는 `log_export_storage_versioning`을 통해 선택적 객체 버전 관리를 지원합니다.
- BigQuery에서 캡처되는 다양한 감사 로그 유형은 30일 동안 보관됩니다.
- 청구 데이터의 경우 권한이 첨부된 BigQuery 데이터셋이 생성되지만, 현재 이를 자동화하는 쉬운 방법이 없으므로 청구 내보내기를 [수동으로](https://cloud.google.com/billing/docs/how-to/export-data-bigquery) 구성해야 합니다.

#### Security Command Center 알림

공통 폴더 아래에 생성되는 또 다른 프로젝트입니다. 이 프로젝트는 조직 수준에서 Security Command Center 알림 리소스를 호스팅합니다.
이 프로젝트에는 Pub/Sub 주제, Pub/Sub 구독 및 생성된 주제로 모든 새 발견 사항을 보내도록 구성된 [Security Command Center 알림](https://cloud.google.com/security-command-center/docs/how-to-notifications)이 포함됩니다.
이 단계를 배포할 때 필터를 조정할 수 있습니다.

#### KMS

공통 폴더 아래에 생성되는 또 다른 프로젝트입니다. 이 프로젝트는 조직에서 공유하는 KMS 리소스에 대한 [Cloud Key Management](https://cloud.google.com/security-key-management)를 위해 할당됩니다.

org 단계에 대한 사용 지침은 [README](./1-org/README.md)에서 확인할 수 있습니다.

#### 시크릿

공통 폴더 아래에 생성되는 또 다른 프로젝트입니다. 이 프로젝트는 조직에서 공유하는 시크릿을 위한 [Secret Manager](https://cloud.google.com/secret-manager)를 위해 할당됩니다.

org 단계에 대한 사용 지침은 [README](./1-org/README.md)에서 확인할 수 있습니다.

#### DNS 허브

이 프로젝트는 네트워크 폴더 아래에 생성됩니다. 이 프로젝트는 조직의 DNS 허브를 호스팅합니다.

#### 인터커넥트

네트워크 폴더 아래에 생성되는 또 다른 프로젝트입니다. 이 프로젝트는 조직의 Dedicated Interconnect [인터커넥트 연결](https://cloud.google.com/network-connectivity/docs/interconnect/concepts/terminology#elements)을 호스팅합니다. Partner Interconnect의 경우 이 프로젝트는 사용되지 않으며 [VLAN 첨부 파일](https://cloud.google.com/network-connectivity/docs/interconnect/concepts/terminology#for-partner-interconnect)이 해당 허브 프로젝트에 직접 배치됩니다.

#### 네트워킹

네트워크 폴더 아래에 환경별(`development`, `nonproduction` 및 `production`)로 공유 VPC 네트워크용 프로젝트가 하나씩 생성되며, 이는 해당 환경의 모든 프로젝트에 대한 [Shared VPC 호스트 프로젝트](https://cloud.google.com/vpc/docs/shared-vpc)로 사용되도록 의도되었습니다.
이 단계는 프로젝트를 생성하고 올바른 API를 활성화하기만 하며, 다음 네트워크 단계인 [3-networks-svpc](./3-networks-svpc/) 및 [3-networks-hub-and-spoke](./3-networks-hub-and-spoke/)에서 실제 Shared VPC 네트워크를 생성합니다.

### [2. environments](./2-environments/)

이 단계의 목적은 각 환경에 대한 공유 프로젝트를 포함하는 환경 폴더를 설정하는 것입니다.
다음과 같은 폴더 및 프로젝트 구조가 생성됩니다:

```
example-organization
└── fldr-development
    ├── prj-d-kms
    └── prj-d-secrets
└── fldr-nonproduction
    ├── prj-n-kms
    └── prj-n-secrets
└── fldr-production
    ├── prj-p-kms
    └── prj-p-secrets
```

#### KMS

환경 폴더 아래에 환경별(`development`, `nonproduction` 및 `production`)로 프로젝트가 생성되며, 이는 환경에서 공유하는 KMS 리소스를 위한 [Cloud Key Management](https://cloud.google.com/security-key-management)에서 사용되도록 의도되었습니다.

environments 단계에 대한 사용 지침은 [README](./2-environments/README.md)에서 확인할 수 있습니다.

#### 시크릿

환경 폴더 아래에 환경별(`development`, `nonproduction` 및 `production`)로 프로젝트가 생성되며, 이는 환경에서 공유하는 시크릿을 위한 [Secret Manager](https://cloud.google.com/secret-manager)에서 사용되도록 의도되었습니다.

environments 단계에 대한 사용 지침은 [README](./2-environments/README.md)에서 확인할 수 있습니다.

### [3. networks-svpc](./3-networks-svpc/)

이 단계는 합리적인 보안 기준선을 갖춘 표준 구성으로 환경별(`development`, `nonproduction` 및 `production`)로 [Shared VPC](https://cloud.google.com/architecture/security-foundations/networking#vpcsharedvpc-id7-1-shared-vpc-)를 생성하는 데 중점을 둡니다. 현재 다음이 포함됩니다:

- (선택사항) Google Kubernetes Engine을 사용하려는 사용자를 위한 보조 범위를 포함한 `development`, `nonproduction` 및 `production`용 예제 서브넷
- 공용 IP 없이 [IAP를 통한 VM](https://cloud.google.com/iap/docs/using-tcp-forwarding)에 대한 원격 액세스를 허용하도록 생성된 계층적 방화벽 정책
- [로드 밸런싱 상태 확인](https://cloud.google.com/load-balancing/docs/health-checks#firewall_rules)을 허용하도록 생성된 계층적 방화벽 정책
- [Windows KMS 활성화](https://cloud.google.com/compute/docs/instances/windows/creating-managing-windows-instances#kms-server)를 허용하도록 생성된 계층적 방화벽 정책
- Cloud SQL과 같은 워크로드 종속 리소스를 활성화하도록 구성된 [프라이빗 서비스 네트워킹](https://cloud.google.com/vpc/docs/configure-private-services-access)
- googleapis.com 및 gcr.io에 대한 제한된 액세스를 위해 구성된 [restricted.googleapis.com](https://cloud.google.com/vpc-service-controls/docs/supported-products)이 있는 Shared VPC. API에 액세스하는 데 인터넷 액세스가 필요하지 않도록 VIP에 대한 경로가 추가됨
- 인터넷으로의 기본 경로가 제거되고, 인터넷에 도달하기 위해 VM에 태그 기반 경로 `egress-internet`이 필요함
- (선택사항) 로깅 및 정적 아웃바운드 IP를 사용하여 모든 서브넷에 대해 구성된 Cloud NAT
- DNS 로깅 및 [인바운드 쿼리 전달](https://cloud.google.com/dns/docs/overview#dns-server-policy-in)이 켜진 기본 Cloud DNS 정책 적용

networks 단계에 대한 사용 지침은 [README](./3-networks-svpc/README.md)에서 확인할 수 있습니다.

### [3. networks-hub-and-spoke](./3-networks-hub-and-spoke/)

이 단계는 3-networks-svpc 단계와 동일한 네트워크 리소스를 구성하지만, 이번에는 [허브 앤 스포크](https://cloud.google.com/architecture/security-foundations/networking#hub-and-spoke) 참조 네트워크 모델을 기반으로 한 아키텍처를 사용합니다.

networks 단계에 대한 사용 지침은 [README](./3-networks-hub-and-spoke/README.md)에서 확인할 수 있습니다.

### [4. projects](./4-projects/)

이 단계는 이전 단계에서 생성된 Shared VPC에 연결되고 애플리케이션 인프라 파이프라인이 있는 표준 구성으로 서비스 프로젝트를 생성하는 데 중점을 둡니다.
이 코드를 그대로 실행하면 아래와 같은 구조가 생성됩니다:

```
example-organization/
└── fldr-development
    └── fldr-development-bu1
        ├── prj-d-bu1-sample-floating
        ├── prj-d-bu1-sample-svpc
        ├── prj-d-bu1-sample-peering
    └── fldr-development-bu2
        ├── prj-d-bu2-sample-floating
        ├── prj-d-bu2-sample-svpc
        └── prj-d-bu2-sample-peering
└── fldr-nonproduction
    └── fldr-nonproduction-bu1
        ├── prj-n-bu1-sample-floating
        ├── prj-n-bu1-sample-svpc
        ├── prj-n-bu1-sample-peering
    └── fldr-nonproduction-bu2
        ├── prj-n-bu2-sample-floating
        ├── prj-n-bu2-sample-svpc
        └── prj-n-bu2-sample-peering
└── fldr-production
    └── fldr-production-bu1
        ├── prj-p-bu1-sample-floating
        ├── prj-p-bu1-sample-svpc
        ├── prj-p-bu1-sample-peering
    └── fldr-production-bu2
        ├── prj-p-bu2-sample-floating
        ├── prj-p-bu2-sample-svpc
        └── prj-p-bu2-sample-peering
└── fldr-common
    ├── prj-c-bu1-infra-pipeline
    └── prj-c-bu2-infra-pipeline
```

이 단계의 코드에는 프로젝트 생성을 위한 두 가지 옵션이 포함되어 있습니다.
첫 번째는 환경별로 프로젝트를 생성하는 표준 프로젝트 모듈이고, 두 번째는 한 환경에 대한 독립 실행형 프로젝트를 생성합니다.
사용 사례와 관련이 있는 경우 프로젝트당 서브넷과 프로젝트당 전용 프라이빗 DNS 영역을 생성하는 데 사용할 수 있는 두 개의 선택적 하위 모듈도 있습니다.

projects 단계에 대한 사용 지침은 [README](./4-projects/README.md)에서 확인할 수 있습니다.

### [5. app-infra](./5-app-infra/)

이 단계의 목적은 4-projects에서 설정한 인프라 파이프라인을 사용하여 비즈니스 단위 프로젝트 중 하나에 간단한 [Compute Engine](https://cloud.google.com/compute/) 인스턴스를 배포하는 것입니다.

app-infra 단계에 대한 사용 지침은 [README](./5-app-infra/README.md)에서 확인할 수 있습니다.

### 최종 뷰

위의 모든 단계가 실행된 후 Google Cloud 조직은 아래와 같은 구조를 나타내야 하며, 프로젝트가 트리의 가장 낮은 노드가 됩니다.

```
example-organization
└── fldr-common
    ├── prj-c-logging
    ├── prj-c-billing-export
    ├── prj-c-scc
    ├── prj-c-kms
    ├── prj-c-secrets
    ├── prj-c-bu1-infra-pipeline
    └── prj-c-bu2-infra-pipeline
└── fldr-network
    ├── prj-net-hub-svpc
    ├── prj-net-dns
    ├── prj-net-interconnect
    ├── prj-d-svpc
    ├── prj-n-svpc
    └── prj-p-svpc
└── fldr-development
    ├── prj-d-kms
    └── prj-d-secrets
    └── fldr-development-bu1
        ├── prj-d-bu1-sample-floating
        ├── prj-d-bu1-sample-svpc
        ├── prj-d-bu1-sample-peering
    └── fldr-development-bu2
        ├── prj-d-bu2-sample-floating
        ├── prj-d-bu2-sample-svpc
        └── prj-d-bu2-sample-peering
└── fldr-nonproduction
    ├── prj-n-kms
    └── prj-n-secrets
    └── fldr-nonproduction-bu1
        ├── prj-n-bu1-sample-floating
        ├── prj-n-bu1-sample-svpc
        ├── prj-n-bu1-sample-peering
    └── fldr-nonproduction-bu2
        ├── prj-n-bu2-sample-floating
        ├── prj-n-bu2-sample-svpc
        └── prj-n-bu2-sample-peering
└── fldr-production
    ├── prj-p-kms
    └── prj-p-secrets
    └── fldr-production-bu1
        ├── prj-p-bu1-sample-floating
        ├── prj-p-bu1-sample-svpc
        ├── prj-p-bu1-sample-peering
    └── fldr-production-bu2
        ├── prj-p-bu2-sample-floating
        ├── prj-p-bu2-sample-svpc
        └── prj-p-bu2-sample-peering
└── fldr-bootstrap
    ├── prj-b-cicd
    └── prj-b-seed
```

### 브랜칭 전략

해당 환경을 반영하는 세 가지 주요 명명된 브랜치가 있습니다: `development`, `nonproduction` 및 `production`. 이러한 브랜치는 [보호](https://docs.github.com/en/github/administering-a-repository/about-protected-branches)되어야 합니다. [CI/CD 파이프라인](/docs/GLOSSARY.md#foundation-cicd-pipeline)(Jenkins 또는 Cloud Build)이 특정 명명된 브랜치(예: `development`)에서 실행되면 해당 환경(`development`)만 적용됩니다. 예외는 `production` 브랜치에서 트리거될 때만 적용되는 `shared` 환경입니다. 이는 `shared` 환경의 변경 사항이 다른 환경의 리소스에 영향을 미칠 수 있고 올바르게 검증되지 않으면 부정적인 영향을 미칠 수 있기 때문입니다.

개발은 기능 및 버그 수정 브랜치(이름은 `feature/new-foo`, `bugfix/fix-bar` 등으로 지정 가능)에서 수행되며, 완료되면 `development` 브랜치를 대상으로 하는 [풀 리퀘스트(PR)](https://docs.github.com/en/github/collaborating-with-issues-and-pull-requests/about-pull-requests) 또는 [머지 리퀘스트(MR)](https://docs.gitlab.com/ee/user/project/merge_requests/)를 열 수 있습니다. 이렇게 하면 [CI/CD 파이프라인](/docs/GLOSSARY.md#foundation-cicd-pipeline)이 트리거되어 모든 환경(`development`, `nonproduction`, `shared` 및 `production`)에 대해 계획을 수행하고 검증합니다. 코드 검토가 완료되고 변경 사항이 검증되면 이 브랜치를 `development`에 병합할 수 있습니다. 이렇게 하면 `development` 환경에서 `development` 브랜치의 최신 변경 사항을 적용하는 [CI/CD 파이프라인](/docs/GLOSSARY.md#foundation-cicd-pipeline)이 트리거됩니다.

`development`에서 검증된 후 `nonproduction` 브랜치를 대상으로 PR 또는 MR을 열고 병합하여 변경 사항을 `nonproduction`으로 승격할 수 있습니다. 마찬가지로 변경 사항을 `nonproduction`에서 `production`으로 승격할 수 있습니다.

### 정책 검증

이 리포지토리는 `gcloud` CLI의 [terraform-tools](https://cloud.google.com/docs/terraform/policy-validation/validate-policies) 구성 요소를 사용하여 [Google Cloud 정책 라이브러리](https://github.com/GoogleCloudPlatform/policy-library)에 대해 Terraform 계획을 검증합니다.

[Scorecard 번들](https://github.com/GoogleCloudPlatform/policy-library/blob/master/docs/bundles/scorecard-v1.md)을 사용하여 [추가 제약 조건 하나](https://github.com/GoogleCloudPlatform/policy-library/blob/master/samples/serviceusage_allow_basic_apis.yaml)가 추가된 [policy-library 폴더](./policy-library)를 생성했습니다.

워크로드 유형에 따라 구성에서 [샘플 폴더](https://github.com/GoogleCloudPlatform/policy-library/tree/master/samples)에서 더 많은 제약 조건을 추가해야 하는 경우 [policy-library 문서](https://github.com/GoogleCloudPlatform/policy-library/blob/master/docs/index.md)를 참조하세요.

1-org 단계에는 이러한 정책을 호스팅할 공유 리포지토리 생성에 대한 [지침](./1-org/README.md#deploying-with-cloud-build)이 있습니다.

### 선택적 변수

단계를 배포하는 데 사용되는 일부 변수에는 기본값이 있으므로 **배포 전에** 요구사항과 일치하는지 확인하세요. 자세한 내용은 변수에 대한 자세한 설명과 함께 Terraform 모듈의 입력 및 출력 테이블이 있습니다. 다음 README의 **Inputs** 섹션에서 **필수 아님**으로 표시된 변수를 찾으세요:

- 0-bootstrap 단계: [CI/CD 파이프라인](/docs/GLOSSARY.md#foundation-cicd-pipeline)에서 Cloud Build를 사용하는 경우 단계의 기본 [README](./0-bootstrap/README.md#Inputs)를 확인하세요. Jenkins를 사용하는 경우 `jenkins-agent` 모듈의 [README](./0-bootstrap/modules/jenkins-agent/README.md#Inputs)를 확인하세요.
- 1-org 단계: `shared` 환경의 [README](./1-org/envs/shared/README.md#Inputs)
- 2-environments 단계: [development](./2-environments/envs/development/README.md#Inputs), [nonproduction](./2-environments/envs/nonproduction/README.md#Inputs) 및 [production](./2-environments/envs/production/README.md#Inputs) 환경의 README
- 3-networks-svpc 단계: [shared](./3-networks-svpc/envs/shared/README.md#inputs), [development](./3-networks-svpc/envs/development/README.md#Inputs), [nonproduction](./3-networks/envs/nonproduction/README.md#Inputs) 및 [production](./3-networks/envs/production/README.md#Inputs) 환경의 README
- 3-networks-hub-and-spoke 단계: [shared](./3-networks-hub-and-spoke/envs/shared/README.md#inputs), [development](./3-networks-hub-and-spoke/envs/development/README.md#Inputs), [nonproduction](./3-networks/envs/nonproduction/README.md#Inputs) 및 [production](./3-networks/envs/production/README.md#Inputs) 환경의 README
- 4-projects 단계: [shared](./4-projects/business_unit_1/shared/README.md#inputs), [development](./4-projects/business_unit_1/development/README.md#Inputs), [nonproduction](./4-projects/business_unit_1/nonproduction/README.md#Inputs) 및 [production](./4-projects/business_unit_1/production/README.md#Inputs) 환경의 README

## 정오표 요약

예제 기반 리포지토리와 [Google Cloud 보안 기반 가이드](https://cloud.google.com/architecture/security-foundations) 간의 차이점에 대한 개요는 [정오표 요약](./ERRATA.md)을 참조하세요.

## 기여하기

이 모듈에 기여하는 방법에 대한 정보는 [기여 가이드라인](./CONTRIBUTING.md)을 참조하세요.
