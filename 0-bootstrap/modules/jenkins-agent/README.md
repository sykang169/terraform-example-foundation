# 개요

이 모듈의 목적은 온프레미스의 기존 Jenkins Controller와 연결할 수 있는 Jenkins Agent를 호스팅하기 위한 Google Cloud Platform 프로젝트 `prj-b-cicd`를 배포하는 것입니다. 이 모듈은 더 이상 사용되지 않는 [cloudbuild 모듈](https://github.com/terraform-google-modules/terraform-google-bootstrap/tree/master/modules/cloudbuild)의 복제본이지만, Jenkins를 사용하도록 재구성되었습니다. 이 모듈은 다음을 생성합니다:

- `prj-b-cicd` 프로젝트, 여기에는 다음이 포함됩니다:
  - Jenkins Agent용 GCE 인스턴스 (SSH를 사용하여 기존 Jenkins Controller에 연결하도록 구성)
  - Jenkins GCE 인스턴스에 연결할 VPC
  - 포트 22를 통한 통신을 허용하는 방화벽 규칙
  - 온프레미스(또는 Jenkins Controller가 위치한 곳)와의 VPN 연결
  - GCE 인스턴스용 사용자 정의 서비스 계정 `sa-jenkins-agent-gce@prj-b-cicd-xxxx.iam.gserviceaccount.com`. 이 서비스 계정은 제공된 Terraform 사용자 정의 서비스 계정에서 토큰을 생성할 수 있는 권한이 부여됩니다.
이 모듈에는 Jenkins Controller를 생성하는 옵션이 포함되어 있지 않다는 점에 유의하십시오. Jenkins Controller를 배포하려면 [GCP의 Jenkins](https://cloud.google.com/jenkins)에 대한 사용 가능한 사용자 가이드 중 하나를 따라야 합니다.

**Jenkins 구현이 없고 원하지 않는다면**, [Cloud Build 모듈을 사용](../../README.md#deploying-with-cloud-build)하는 것을 권장합니다.

## 사용법

이 하위 모듈의 기본 사용법은 다음과 같습니다:

```hcl
module "jenkins_bootstrap" {
  source                                    = "./modules/jenkins-agent"
  org_id                                    = "<ORGANIZATION_ID>"
  folder_id                                 = "<FOLDER_ID>"
  billing_account                           = "<BILLING_ACCOUNT_ID>"
  group_org_admins                          = "gcp-organization-admins@example.com"
  default_region                            = "us-central1"
  terraform_sa_names                        = "<SERVICE_ACCOUNT_NAMES>"
  terraform_state_bucket                    = "<GCS_STATE_BUCKET_NAME>"
  sa_enable_impersonation                   = true
  jenkins_controller_subnetwork_cidr_range  = ["10.1.0.6/32"]
  jenkins_agent_gce_subnetwork_cidr_range   = "172.16.1.0/24"
  jenkins_agent_gce_private_ip_address      = "172.16.1.6"
  nat_bgp_asn                               = "BGP_ASN_FOR_NAT_CLOUD_ROUTE"
  jenkins_agent_sa_email                    = "jenkins-agent-gce" # service_account_prefix will be added
  jenkins_agent_gce_ssh_pub_key             = var.jenkins_agent_gce_ssh_pub_key
}
```

## 기능

1. `project_prefix`를 사용하여 새로운 GCP 프로젝트를 생성합니다
1. `activate_apis`를 사용하여 프로젝트에서 API를 활성화합니다
1. 제공된 공개 키를 사용하여 SSH 액세스가 가능한 Jenkins Agent를 실행할 GCE 인스턴스를 생성합니다
1. Jenkins Agent GCE 인스턴스를 실행할 서비스 계정(`jenkins_agent_sa_email`)을 생성합니다
1. `project_prefix`를 사용하여 Jenkins 아티팩트용 GCS 버킷을 생성합니다
1. `sa_enable_impersonation`과 `terraform_sa_names`에 제공된 값을 사용하여 `jenkins_agent_sa_email` 서비스 계정이 terraform 서비스 계정(`seed` 프로젝트에 존재)을 가장할 수 있는 권한을 허용합니다
1. Agent가 업데이트와 필요한 바이너리를 다운로드할 수 있도록 Cloud NAT를 추가합니다.

<!-- BEGINNING OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| activate\_apis | List of APIs to enable in the CICD project. | `list(string)` | <pre>[<br>  "serviceusage.googleapis.com",<br>  "servicenetworking.googleapis.com",<br>  "compute.googleapis.com",<br>  "logging.googleapis.com",<br>  "bigquery.googleapis.com",<br>  "cloudresourcemanager.googleapis.com",<br>  "cloudbilling.googleapis.com",<br>  "iam.googleapis.com",<br>  "admin.googleapis.com",<br>  "appengine.googleapis.com",<br>  "storage-api.googleapis.com",<br>  "dns.googleapis.com"<br>]</pre> | no |
| bgp\_peer\_asn | BGP ASN for peer cloud routes. | `number` | `"64513"` | no |
| billing\_account | The ID of the billing account to associate projects with. | `string` | n/a | yes |
| default\_region | Default region to create resources where applicable. | `string` | `"us-central1"` | no |
| folder\_id | The ID of a folder to host this project | `string` | `""` | no |
| group\_org\_admins | Google Group for GCP Organization Administrators | `string` | n/a | yes |
| jenkins\_agent\_gce\_machine\_type | Jenkins Agent GCE Instance type. | `string` | `"n1-standard-1"` | no |
| jenkins\_agent\_gce\_name | Jenkins Agent GCE Instance name. | `string` | `"jenkins-agent-01"` | no |
| jenkins\_agent\_gce\_private\_ip\_address | The private IP Address of the Jenkins Agent. This IP Address must be in the CIDR range of `jenkins_agent_gce_subnetwork_cidr_range` and be reachable through the VPN that exists between on-prem (Jenkins Controller) and GCP (CICD Project, where the Jenkins Agent is located). | `string` | n/a | yes |
| jenkins\_agent\_gce\_ssh\_pub\_key | SSH public key needed by the Jenkins Agent GCE Instance. The Jenkins Controller holds the SSH private key. The correct format is `'ssh-rsa [KEY_VALUE] [USERNAME]'` | `string` | n/a | yes |
| jenkins\_agent\_gce\_subnetwork\_cidr\_range | The subnetwork to which the Jenkins Agent will be connected to (in CIDR range 0.0.0.0/0) | `string` | n/a | yes |
| jenkins\_agent\_sa\_email | Email for Jenkins Agent service account. | `string` | `"jenkins-agent-gce"` | no |
| jenkins\_controller\_subnetwork\_cidr\_range | A list of CIDR IP ranges of the Jenkins Controller in the form ['0.0.0.0/0']. Usually only one IP in the form '0.0.0.0/32'. Needed to create a FW rule that allows communication with the Jenkins Agent GCE Instance. | `list(string)` | n/a | yes |
| nat\_bgp\_asn | BGP ASN for NAT cloud route. This is needed to allow the Jenkins Agent to download packages and updates from the internet without having an external IP address. | `number` | n/a | yes |
| on\_prem\_vpn\_public\_ip\_address | The public IP Address of the Jenkins Controller. | `string` | n/a | yes |
| on\_prem\_vpn\_public\_ip\_address2 | The secondpublic IP Address of the Jenkins Controller. | `string` | n/a | yes |
| org\_id | GCP Organization ID | `string` | n/a | yes |
| project\_deletion\_policy | The deletion policy for the project created. | `string` | `"PREVENT"` | no |
| project\_labels | Labels to apply to the project. | `map(string)` | `{}` | no |
| project\_prefix | Name prefix to use for projects created. | `string` | `"prj"` | no |
| router\_asn | BGP ASN for cloud routes. | `number` | `"64515"` | no |
| sa\_enable\_impersonation | Allow org\_admins group to impersonate service account & enable APIs required. | `bool` | `false` | no |
| service\_account\_prefix | Name prefix to use for service accounts. | `string` | `"sa"` | no |
| storage\_bucket\_labels | Labels to apply to the storage bucket. | `map(string)` | `{}` | no |
| storage\_bucket\_prefix | Name prefix to use for storage buckets. | `string` | `"bkt"` | no |
| terraform\_sa\_names | Fully-qualified name of the Terraform Service Accounts. It must be supplied by the Seed Project | `map(string)` | n/a | yes |
| terraform\_state\_bucket | Default state bucket, used in Cloud Build substitutions. It must be supplied by the Seed Project | `string` | n/a | yes |
| terraform\_version | Default terraform version. | `string` | `"1.5.7"` | no |
| terraform\_version\_sha256sum | sha256sum for default terraform version. | `string` | `"380ca822883176af928c80e5771d1c0ac9d69b13c6d746e6202482aedde7d457"` | no |
| tunnel0\_bgp\_peer\_address | BGP peer address for tunnel 0 | `string` | n/a | yes |
| tunnel0\_bgp\_session\_range | BGP session range for tunnel 0 | `string` | n/a | yes |
| tunnel1\_bgp\_peer\_address | BGP peer address for tunnel 1 | `string` | n/a | yes |
| tunnel1\_bgp\_session\_range | BGP session range for tunnel 1 | `string` | n/a | yes |
| vpn\_shared\_secret | The shared secret used in the VPN | `string` | n/a | yes |

## Outputs

| Name | Description |
|------|-------------|
| cicd\_project\_id | Project where the [CI/CD Pipeline](/docs/GLOSSARY.md#foundation-cicd-pipeline) (Jenkins Agents and terraform builder container image) reside. |
| gcs\_bucket\_jenkins\_artifacts | Bucket used to store Jenkins artifacts in Jenkins project. |
| jenkins\_agent\_gce\_instance\_id | Jenkins Agent GCE Instance id. |
| jenkins\_agent\_sa\_email | Email for privileged custom service account for Jenkins Agent GCE instance. |
| jenkins\_agent\_sa\_name | Fully qualified name for privileged custom service account for Jenkins Agent GCE instance. |
| jenkins\_agent\_vpc\_id | Jenkins Agent VPC name. |

<!-- END OF PRE-COMMIT-TERRAFORM DOCS HOOK -->

## 요구 사항

### 소프트웨어

- [gcloud sdk](https://cloud.google.com/sdk/install) >= 393.0.0
- [Terraform](https://www.terraform.io/downloads.html) = 1.5.7
  - 이 코드베이스의 스크립트는 Terraform v1.5.7을 사용합니다. terraform 버전 차이로 인한 [Terraform State Snapshot Lock](https://github.com/hashicorp/terraform/issues/23290) 오류를 방지하기 위해 수동 단계에서 동일한 버전을 사용해야 합니다.

### 인프라

- **Jenkins Controller:** 이 모듈에는 Jenkins Controller를 생성하는 옵션이 포함되어 있지 않으므로 Jenkins Controller가 필요합니다. Jenkins Controller를 배포하려면 [GCP의 Jenkins](https://cloud.google.com/jenkins)에 대한 사용 가능한 사용자 가이드 중 하나를 따라야 합니다. Jenkins 구현이 없고 원하지 않는다면 [Cloud Build 모듈을 사용](../../README.md#deploying-with-cloud-build)하는 것을 권장합니다.

- **온프레미스와의 VPN 연결:** 이 모듈을 실행하면 GCP의 CI/CD 프로젝트에 Jenkins Agent가 생성됩니다. [GCP에서 VPN 터널을 배포하는 방법](https://cloud.google.com/network-connectivity/docs/vpn/how-to/adding-a-tunnel)에 대한 사용자 가이드에 따라 VPN 연결을 수동으로 추가하십시오. 이 VPN은 Jenkins Controller(온프레미스 또는 클라우드 환경)와 CI/CD 프로젝트의 Jenkins Agent 간의 통신을 허용하는 데 필요합니다.

- **Jenkins Agent용 바이너리 및 패키지:** Jenkins Agent는 이 모듈에 의해 생성된 새로운 GCE 인스턴스입니다. 생성 후 시작 스크립트는 파이프라인 실행 중 나중에 사용할 여러 바이너리를 가져와야 합니다. 이러한 바이너리에는 `java`, `terraform` 및 사용자 자체 스크립트에서 사용하는 기타 바이너리가 포함됩니다. Jenkins Agent에서 이러한 바이너리와 라이브러리를 사용할 수 있도록 하는 몇 가지 옵션이 있습니다:
  - Jenkins Agent의 인터넷 액세스를 허용합니다(기본적으로 구현되는 Cloud NAT를 통해 이상적으로).
  - VPN 연결을 통해 이상적으로 온프레미스의 로컬 패키지 저장소에 대한 Jenkins Agent 액세스를 허용합니다.
  - Jenkins Agent용 골든 이미지를 준비합니다(`jenkins_agent_gce_instance.boot_disk.initialize_params.image` terraform 변수에 이미지를 할당). Packer와 같은 도구로 골든 이미지를 생성할 수 있습니다. 하지만 파이프라인을 실행하는 동안 종속성을 다운로드하려면 여전히 네트워크 액세스가 필요할 수 있습니다.

### 권한

다음 [권한](https://github.com/terraform-google-modules/terraform-google-bootstrap#permissions)이 있는 계정:

- 제공된 청구 계정에 대한 `roles/billing.user`
- GCP 조직에 대한 `roles/resourcemanager.organizationAdmin`
- GCP 조직 또는 폴더에 대한 `roles/resourcemanager.projectCreator`

다음과 같은 오류가 발생할 수 있으므로 이는 특히 중요합니다:

```text
Error: google: could not find default credentials. See https://developers.google.com/accounts/docs/application-default-credentials for more information.
   on <empty> line 0:
  (source code not available)
```

```text
Error: Error setting billing account "aaaaaa-bbbbbb-cccccc" for project "projects/prj-jenkins-dc3a": googleapi: Error 400: Precondition check failed., failedPrecondition
      on .terraform/modules/jenkins/terraform-google-project-factory-7.1.0/modules/core_project_factory/main.tf line 96, in resource "google_project" "main":
      96: resource "google_project" "main" {
```

```text
Error: failed pre-requisites: missing permission on "billingAccounts/aaaaaa-bbbbbb-cccccc": billing.resourceAssociations.create
  on .terraform/modules/jenkins/terraform-google-project-factory-7.1.0/modules/core_project_factory/main.tf line 96, in resource "google_project" "main":
  96: resource "google_project" "main" {
```

### API

이 모듈의 리소스를 호스팅하려면 다음 API가 활성화된 프로젝트를 사용해야 합니다:

- Google Cloud Resource Manager API: `cloudresourcemanager.googleapis.com`
- Google Cloud Billing API: `cloudbilling.googleapis.com`
- Google Cloud IAM API: `iam.googleapis.com`
- Google Cloud Storage API `storage-api.googleapis.com`
- Google Cloud Service Usage API: `serviceusage.googleapis.com`
- Google Cloud Compute API: `compute.googleapis.com`
- Google Cloud KMS API: `cloudkms.googleapis.com`

이 API는 조직을 설정하는 동안 생성된 기본 프로젝트에서 활성화할 수 있습니다.

## 기여

이 모듈에 기여하는 방법에 대한 정보는 [기여 가이드라인](../../../CONTRIBUTING.md)을 참조하십시오.
