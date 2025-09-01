# Dedicated Interconnect 모듈

이 모듈은 [Dedicated Interconnect에 대한 99.99% 가용성 확보](https://cloud.google.com/network-connectivity/docs/interconnect/tutorials/dedicated-creating-9999-availability)에서 제안된 권장 사항을 구현합니다.

## 전제 조건

1. `1-org` 단계에서 `fldr-common` 폴더 하위에 생성된 `prj-interconnect` 프로젝트에서 4개의 [Dedicated Interconnect 연결](https://cloud.google.com/network-connectivity/docs/interconnect/concepts/dedicated-overview)을 프로비저닝합니다.

## 사용법

1. `3-networks-hub-and-spoke/envs/shared`의 공유 환경 폴더에서 `interconnect.tf.example`을 `interconnect.tf`로 이름을 변경합니다
1. `3-networks-hub-and-spoke/envs/shared`의 공유 환경 폴더에서 `interconnect.auto.tfvars.example`을 `interconnect.auto.tfvars`로 이름을 변경합니다
1. interconnect, 위치, 후보 서브네트워크, vlan_tag8021q 및 피어 정보에 대해 환경에 유효한 값으로 `interconnect.tf` 파일을 업데이트합니다.
1. 후보 서브네트워크 및 vlan_tag8021q 변수는 `null`로 설정하여 interconnect 모듈이 이러한 값을 자동으로 생성하도록 할 수 있습니다.

<!-- BEGINNING OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
## 입력

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| cloud\_router\_labels | A map of suffixes for labelling vlans with four entries like "vlan\_1" => "suffix1" with keys from `vlan_1` to `vlan_4`. | `map(string)` | `{}` | no |
| interconnect\_project\_id | Interconnect project ID. | `string` | n/a | yes |
| peer\_asn | Peer BGP Autonomous System Number (ASN). | `number` | n/a | yes |
| peer\_name | Name of this BGP peer. The name must be 1-63 characters long, and comply with RFC1035. Specifically, the name must be 1-63 characters long and match the regular expression [a-z]([-a-z0-9]*[a-z0-9])? | `string` | n/a | yes |
| region1 | First subnet region. The Dedicated Interconnect module only configures two regions. | `string` | n/a | yes |
| region1\_interconnect1 | URL of the underlying Interconnect object that this attachment's traffic will traverse through. | `string` | n/a | yes |
| region1\_interconnect1\_candidate\_subnets | Up to 16 candidate prefixes that can be used to restrict the allocation of cloudRouterIpAddress and customerRouterIpAddress for this attachment. All prefixes must be within link-local address space (169.254.0.0/16) and must be /29 or shorter (/28, /27, etc). | `list(string)` | `null` | no |
| region1\_interconnect1\_location | Name of the interconnect location used in the creation of the Interconnect for the first location of region1 | `string` | n/a | yes |
| region1\_interconnect1\_onprem\_dc | Name of the on premisses data center used in the creation of the Interconnect for the first location of region1. | `string` | n/a | yes |
| region1\_interconnect1\_vlan\_tag8021q | The IEEE 802.1Q VLAN tag for this attachment, in the range 2-4094. | `string` | `null` | no |
| region1\_interconnect2 | URL of the underlying Interconnect object that this attachment's traffic will traverse through. | `string` | n/a | yes |
| region1\_interconnect2\_candidate\_subnets | Up to 16 candidate prefixes that can be used to restrict the allocation of cloudRouterIpAddress and customerRouterIpAddress for this attachment. All prefixes must be within link-local address space (169.254.0.0/16) and must be /29 or shorter (/28, /27, etc). | `list(string)` | `null` | no |
| region1\_interconnect2\_location | Name of the interconnect location used in the creation of the Interconnect for the second location of region1 | `string` | n/a | yes |
| region1\_interconnect2\_onprem\_dc | Name of the on premisses data center used in the creation of the Interconnect for the second location of region1. | `string` | n/a | yes |
| region1\_interconnect2\_vlan\_tag8021q | The IEEE 802.1Q VLAN tag for this attachment, in the range 2-4094. | `string` | `null` | no |
| region1\_router1\_name | Name of the Router 1 for Region 1 where the attachment resides. | `string` | n/a | yes |
| region1\_router2\_name | Name of the Router 2 for Region 1 where the attachment resides. | `string` | n/a | yes |
| region2 | Second subnet region. The Dedicated Interconnect module only configures two regions. | `string` | n/a | yes |
| region2\_interconnect1 | URL of the underlying Interconnect object that this attachment's traffic will traverse through. | `string` | n/a | yes |
| region2\_interconnect1\_candidate\_subnets | Up to 16 candidate prefixes that can be used to restrict the allocation of cloudRouterIpAddress and customerRouterIpAddress for this attachment. All prefixes must be within link-local address space (169.254.0.0/16) and must be /29 or shorter (/28, /27, etc). | `list(string)` | `null` | no |
| region2\_interconnect1\_location | Name of the interconnect location used in the creation of the Interconnect for the first location of region2 | `string` | n/a | yes |
| region2\_interconnect1\_onprem\_dc | Name of the on premisses data center used in the creation of the Interconnect for the first location of region2. | `string` | n/a | yes |
| region2\_interconnect1\_vlan\_tag8021q | The IEEE 802.1Q VLAN tag for this attachment, in the range 2-4094. | `string` | `null` | no |
| region2\_interconnect2 | URL of the underlying Interconnect object that this attachment's traffic will traverse through. | `string` | n/a | yes |
| region2\_interconnect2\_candidate\_subnets | Up to 16 candidate prefixes that can be used to restrict the allocation of cloudRouterIpAddress and customerRouterIpAddress for this attachment. All prefixes must be within link-local address space (169.254.0.0/16) and must be /29 or shorter (/28, /27, etc). | `list(string)` | `null` | no |
| region2\_interconnect2\_location | Name of the interconnect location used in the creation of the Interconnect for the second location of region2 | `string` | n/a | yes |
| region2\_interconnect2\_onprem\_dc | Name of the on premisses data center used in the creation of the Interconnect for the second location of region2. | `string` | n/a | yes |
| region2\_interconnect2\_vlan\_tag8021q | The IEEE 802.1Q VLAN tag for this attachment, in the range 2-4094. | `string` | `null` | no |
| region2\_router1\_name | Name of the Router 1 for Region 2 where the attachment resides. | `string` | n/a | yes |
| region2\_router2\_name | Name of the Router 2 for Region 2 where the attachment resides | `string` | n/a | yes |
| vpc\_name | Label to identify the VPC associated with shared VPC that will use the Interconnect. | `string` | n/a | yes |

## 출력

| Name | Description |
|------|-------------|
| interconnect\_attachment1\_region1 | The interconnect attachment 1 for region 1 |
| interconnect\_attachment1\_region1\_customer\_router\_ip\_address | IPv4 address + prefix length to be configured on the customer router subinterface for this interconnect attachment. |
| interconnect\_attachment1\_region2 | The interconnect attachment 1 for region 2 |
| interconnect\_attachment1\_region2\_customer\_router\_ip\_address | IPv4 address + prefix length to be configured on the customer router subinterface for this interconnect attachment. |
| interconnect\_attachment2\_region1 | The interconnect attachment 2 for region 1 |
| interconnect\_attachment2\_region1\_customer\_router\_ip\_address | IPv4 address + prefix length to be configured on the customer router subinterface for this interconnect attachment. |
| interconnect\_attachment2\_region2 | The interconnect attachment 2 for region 2 |
| interconnect\_attachment2\_region2\_customer\_router\_ip\_address | IPv4 address + prefix length to be configured on the customer router subinterface for this interconnect attachment. |

<!-- END OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
