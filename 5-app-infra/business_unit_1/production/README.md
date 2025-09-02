<!-- BEGINNING OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
## 입력

| 이름 | 설명 | 유형 | 기본값 | 필수 |
|------|-------------|------|---------|:--------:|
| instance\_region | The region where compute instance will be created. A subnetwork must exists in the instance region. | `string` | `null` | no |
| remote\_state\_bucket | Backend bucket to load remote state information from previous steps. | `string` | n/a | yes |

## 출력

| 이름 | 설명 |
|------|-------------|
| available\_zones | List of available zones in region |
| instances\_details | List of details for compute instances |
| instances\_names | List of names for compute instances |
| instances\_self\_links | List of self-links for compute instances |
| instances\_zones | List of zone for compute instances |
| project\_id | Project where compute instance was created |
| region | Region where compute instance was created |

<!-- END OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
