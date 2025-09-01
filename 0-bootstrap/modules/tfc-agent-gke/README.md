# GKE에서의 자체 호스팅 Terraform Cloud 에이전트

이 모듈은 프라이빗 autopilot Google Kubernetes Engine(GKE)에서 Terraform Cloud 에이전트를 배포하는 데 필요한 인프라의 독단적인 생성을 처리합니다.

여기에는 다음이 포함됩니다:

- VPC
- Autopilot이 있는 GKE 프라이빗 클러스터
- Kubernetes 시크릿
- Kubernetes 배포
- Kubernetes Fleet Hub
- Private Service Connect

<!-- BEGINNING OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
## 입력

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| autopilot\_gke\_io\_warden\_version | Autopilot GKE IO Warden Version | `string` | `"2.7.41"` | no |
| create\_service\_account | Set to true to create a new service account, false to use an existing one | `bool` | `true` | no |
| firewall\_enable\_logging | n/a | `bool` | `true` | no |
| ip\_range\_pods\_cidr | The secondary IP range CIDR to use for pods | `string` | `"192.168.0.0/18"` | no |
| ip\_range\_pods\_name | The secondary IP range to use for pods | `string` | `"ip-range-pods"` | no |
| ip\_range\_services\_cider | The secondary IP range CIDR to use for services | `string` | `"192.168.64.0/18"` | no |
| ip\_range\_services\_name | The secondary IP range to use for services | `string` | `"ip-range-scv"` | no |
| machine\_type | Machine type for TFC agent node pool | `string` | `"n1-standard-4"` | no |
| max\_node\_count | Maximum number of nodes in the TFC agent node pool | `number` | `4` | no |
| min\_node\_count | Minimum number of nodes in the TFC agent node pool | `number` | `2` | no |
| nat\_bgp\_asn | BGP ASN for NAT cloud routes. | `number` | `64514` | no |
| nat\_enabled | n/a | `bool` | `true` | no |
| nat\_num\_addresses | n/a | `number` | `2` | no |
| network\_name | Name for the VPC network | `string` | `"tfc-agent-network"` | no |
| network\_project\_id | The project ID of the shared VPCs host (for shared vpc support).<br>If not provided, the project\_id is used | `string` | `""` | no |
| private\_service\_connect\_ip | n/a | `string` | `"10.10.64.5"` | no |
| project\_id | The Google Cloud Platform project ID to deploy Terraform Cloud agent cluster | `string` | n/a | yes |
| project\_number | The project number to host the cluster in | `any` | n/a | yes |
| region | The GCP region to use when deploying resources | `string` | `"us-central1"` | no |
| service\_account\_email | Optional Service Account for the GKE nodes, required if create\_service\_account is set to false | `string` | `""` | no |
| service\_account\_id | Optional Service Account for the GKE nodes, required if create\_service\_account is set to false | `string` | `""` | no |
| subnet\_ip | IP range for the subnet | `string` | `"10.0.0.0/17"` | no |
| subnet\_name | Name for the subnet | `string` | `"tfc-agent-subnet"` | no |
| tfc\_agent\_address | The HTTP or HTTPS address of the Terraform Cloud/Enterprise API | `string` | `"https://app.terraform.io"` | no |
| tfc\_agent\_auto\_update | Controls automatic core updates behavior. Acceptable values include disabled, patch, and minor | `string` | `"minor"` | no |
| tfc\_agent\_cpu\_request | CPU request for the Terraform Cloud agent container | `string` | `"2"` | no |
| tfc\_agent\_ephemeral\_storage | A temporary storage for a container that gets wiped out and lost when the container is stopped or restarted | `string` | `"1Gi"` | no |
| tfc\_agent\_image | The Terraform Cloud agent image to use | `string` | `"hashicorp/tfc-agent:latest"` | no |
| tfc\_agent\_k8s\_secrets | Name for the k8s secret required to configure TFC agent on GKE | `string` | `"tfc-agent-k8s-secrets"` | no |
| tfc\_agent\_max\_replicas | Maximum replicas for the Terraform Cloud agent pod autoscaler | `string` | `"10"` | no |
| tfc\_agent\_memory\_request | Memory request for the Terraform Cloud agent container | `string` | `"2Gi"` | no |
| tfc\_agent\_min\_replicas | Minimum replicas for the Terraform Cloud agent pod autoscaler | `string` | `"1"` | no |
| tfc\_agent\_name\_prefix | This name may be used in the Terraform Cloud user interface to help easily identify the agent | `string` | `"tfc-agent-k8s"` | no |
| tfc\_agent\_single | Enable single mode. This causes the agent to handle at most one job and<br>immediately exit thereafter. Useful for running agents as ephemeral<br>containers, VMs, or other isolated contexts with a higher-level scheduler<br>or process supervisor. | `bool` | `false` | no |
| tfc\_agent\_token | Terraform Cloud agent token. (Organization Settings >> Agents) | `string` | n/a | yes |
| zones | The GCP zone to use when deploying resources | `list(string)` | <pre>[<br>  "us-central1-a"<br>]</pre> | no |

## 출력

| 이름 | 설명 |
|------|-------------|
| cluster\_name | GKE 클러스터 이름 |
| hub\_cluster\_membership\_id | 클러스터 멤버십의 ID |
| kubernetes\_endpoint | GKE 클러스터 엔드포인트 |
| service\_account | TFC 에이전트 노드에 사용되는 기본 서비스 계정 |

<!-- END OF PRE-COMMIT-TERRAFORM DOCS HOOK -->

## 요구 사항

프로젝트에서 이 모듈을 사용하기 전에 다음 전제 조건이 충족되는지 확인해야 합니다:

1. 필요한 API가 활성화됨

    ```text
    "iam.googleapis.com",
    "cloudresourcemanager.googleapis.com",
    "containerregistry.googleapis.com",
    "container.googleapis.com",
    "storage-component.googleapis.com",
    "logging.googleapis.com",
    "monitoring.googleapis.com"
    ```
