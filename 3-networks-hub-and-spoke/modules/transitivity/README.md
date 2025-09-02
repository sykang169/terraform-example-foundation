# 허브 앤 스포크 전이성 모듈

이 모듈은 라우트의 넥스트홉으로 사용되는 Internal Load Balancer 뒤에 있는 어플라이언스 VM을 사용하여 허브 앤 스포크 VPC 아키텍처의 전이성을 구현합니다.

## 사용법

사용 예제는 [net-hubs-transitivity.tf](../../envs/shared/net-hubs-transitivity.tf) 파일을 확인하시기 바랍니다.

<!-- BEGINNING OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
## 입력

| 이름 | 설명 | 유형 | 기본값 | 필수 |
|------|-------------|------|---------|:--------:|
| commands | Commands for the transitivity gateway to run on every boot. | `list(string)` | `[]` | no |
| firewall\_enable\_logging | Toggle firewall logging for VPC Firewalls. | `bool` | `true` | no |
| firewall\_policy | Network Firewall Policy Id to deploy transitivity firewall rules. | `string` | n/a | yes |
| gw\_subnets | Subnets in {REGION => SUBNET} format. | `map(string)` | n/a | yes |
| health\_check\_enable\_log | Toggle logging for health checks. | `bool` | `false` | no |
| project\_id | VPC Project ID | `string` | n/a | yes |
| regional\_aggregates | Aggregate ranges for each region in {REGION => [AGGREGATE\_CIDR,] } format. | `map(list(string))` | n/a | yes |
| regions | Regions to deploy the transitivity appliances | `set(string)` | `null` | no |
| vpc\_name | Label to identify the VPC associated with shared VPC that will use the Interconnect. | `string` | n/a | yes |

## 출력

출력 없음.

<!-- END OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
