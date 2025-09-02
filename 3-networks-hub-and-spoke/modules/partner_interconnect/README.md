# Partner Interconnect 모듈

이 모듈은 [Partner Interconnect에 대한 99.99% 가용성 확보](https://cloud.google.com/network-connectivity/docs/interconnect/tutorials/partner-creating-9999-availability)에서 제안된 권장 사항을 구현합니다.

## 전제 조건

1. 지정된 환경의 허브 프로젝트에서 4개의 [VLAN 연결](https://cloud.google.com/network-connectivity/docs/interconnect/concepts/partner-overview)을 프로비저닝합니다. 허브 앤 스포크 아키텍처의 경우 `fldr-common` 폴더 아래의 `prj-c-svpc-net-hub` 및 `prj-net-dns`가 해당됩니다.

## 사용법

1. `3-networks-hub-and-spoke/envs/shared`의 shared envs 폴더에서 `partner_interconnect.tf.example`을 `partner_interconnect.tf`로 이름을 바꿉니다.
1. `3-networks-hub-and-spoke/envs/shared`의 shared envs 폴더에서 `partner_interconnect.auto.tfvars.example`을 `partner_interconnect.auto.tfvars`로 이름을 바꿉니다.
1. VLAN 연결 및 위치에 대해 사용자 환경에 유효한 값으로 `partner_interconnect.tf` 파일을 업데이트합니다.

<!-- BEGINNING OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
## 입력

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| attachment\_project\_id | the Interconnect project ID. | `string` | n/a | yes |
| cloud\_router\_labels | A map of suffixes for labelling vlans with four entries like "vlan\_1" => "suffix1" with keys from `vlan_1` to `vlan_4`. | `map(string)` | `{}` | no |
| preactivate | Preactivate Partner Interconnect attachments, works only for level3 Partner Interconnect | `string` | `false` | no |
| region1 | First subnet region. The Partner Interconnect module only configures two regions. | `string` | n/a | yes |
| region1\_interconnect1\_location | Name of the interconnect location used in the creation of the Interconnect for the first location of region1 | `string` | n/a | yes |
| region1\_interconnect1\_onprem\_dc | Name of the on premisses data center used in the creation of the Interconnect for the first location of region1. | `string` | n/a | yes |
| region1\_interconnect2\_location | Name of the interconnect location used in the creation of the Interconnect for the second location of region1 | `string` | n/a | yes |
| region1\_interconnect2\_onprem\_dc | Name of the on premisses data center used in the creation of the Interconnect for the second location of region1. | `string` | n/a | yes |
| region1\_router1\_name | Name of the Router 1 for Region 1 where the attachment resides. | `string` | n/a | yes |
| region1\_router2\_name | Name of the Router 2 for Region 1 where the attachment resides. | `string` | n/a | yes |
| region2 | Second subnet region. The Partner Interconnect module only configures two regions. | `string` | n/a | yes |
| region2\_interconnect1\_location | Name of the interconnect location used in the creation of the Interconnect for the first location of region2 | `string` | n/a | yes |
| region2\_interconnect1\_onprem\_dc | Name of the on premisses data center used in the creation of the Interconnect for the first location of region2. | `string` | n/a | yes |
| region2\_interconnect2\_location | Name of the interconnect location used in the creation of the Interconnect for the second location of region2 | `string` | n/a | yes |
| region2\_interconnect2\_onprem\_dc | Name of the on premisses data center used in the creation of the Interconnect for the second location of region2. | `string` | n/a | yes |
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
