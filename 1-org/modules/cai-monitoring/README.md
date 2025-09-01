# Cloud Asset Inventory 알림
Google Cloud Asset Inventory를 사용하여 IAM 정책 변경 이벤트의 피드를 생성한 다음, 이를 처리하여 (사전 설정된 목록에서) 역할이 멤버(서비스 계정, 사용자 또는 그룹)에게 부여되는 시점을 감지합니다. 그런 다음 멤버, 역할, 부여된 리소스 및 부여된 시간이 포함된 SCC 발견 사항을 생성합니다.

## 사용법

```hcl
module "secure_cai_notification" {
  source = "terraform-google-modules/terraform-example-foundation/google//1-org/modules/cai-monitoring"

  org_id               = <ORG ID>
  billing_account      = <BILLING ACCOUNT ID>
  project_id           = <PROJECT ID>
  region               = <REGION>
  encryption_key       = <CMEK KEY>
  labels               = <LABELS>
  roles_to_monitor     = <ROLES TO MONITOR>
}
```

<!-- BEGINNING OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
## 입력

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| billing\_account | The ID of the billing account to associate projects with. | `string` | n/a | yes |
| build\_service\_account | Cloud Function Build Service Account Id. This is The fully-qualified name of the service account to be used for building the container. | `string` | n/a | yes |
| enable\_cmek | The KMS Key to Encrypt Artifact Registry repository, Cloud Storage Bucket and Pub/Sub. | `bool` | `false` | no |
| encryption\_key | The KMS Key to Encrypt Artifact Registry repository, Cloud Storage Bucket and Pub/Sub. | `string` | `null` | no |
| labels | Labels to be assigned to resources. | `map(any)` | `{}` | no |
| location | Default location to create resources where applicable. | `string` | `"us-central1"` | no |
| org\_id | GCP Organization ID | `string` | n/a | yes |
| project\_id | The Project ID where the resources will be created | `string` | n/a | yes |
| random\_suffix | Adds a suffix of 4 random characters to the created resources names. | `bool` | `true` | no |
| roles\_to\_monitor | List of roles that will save a SCC Finding if granted to any member (service account, user or group) on an update in the IAM Policy. | `list(string)` | <pre>[<br>  "roles/owner",<br>  "roles/editor",<br>  "roles/resourcemanager.organizationAdmin",<br>  "roles/compute.networkAdmin",<br>  "roles/compute.orgFirewallPolicyAdmin"<br>]</pre> | no |

## 출력

| 이름 | 설명 |
|------|-------------|
| artifact\_registry\_name | Cloud Function 이미지를 저장하는 Artifact Registry 리포지토리. |
| asset\_feed\_name | 조직 자산 피드. |
| bucket\_name | 소스 코드가 있는 스토리지 버킷. |
| function\_uri | Cloud Function의 URI. |
| scc\_source | SCC 발견 사항 소스. |
| topic\_name | 자산 피드용 Pub/Sub 토픽. |

<!-- END OF PRE-COMMIT-TERRAFORM DOCS HOOK -->

## 요구 사항

### 소프트웨어

다음 종속성을 사용할 수 있어야 합니다:

* [Terraform](https://www.terraform.io/downloads.html) >= 1.3
* [GCP용 Terraform 프로바이더](https://github.com/terraform-providers/terraform-provider-google) >= 3.77

### API

이 모듈의 리소스를 호스팅하려면 다음 API가 활성화된 프로젝트를 사용해야 합니다:

* 프로젝트
  * Google Cloud Key Management Service: `cloudkms.googleapis.com`
  * Cloud Resource Manager API: `cloudresourcemanager.googleapis.com`
  * Cloud Functions API: `cloudfunctions.googleapis.com`
  * Cloud Build API: `cloudbuild.googleapis.com`
  * Cloud Asset API`cloudasset.googleapis.com`
  * Cloud Pub/Sub API: `pubsub.googleapis.com`
  * Identity and Access Management (IAM) API: `iam.googleapis.com`
  * Cloud Billing API: `cloudbilling.googleapis.com`
