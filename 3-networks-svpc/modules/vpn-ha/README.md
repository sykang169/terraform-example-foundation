# 고가용성 VPN 모듈

이 모듈은 [고가용성 VPN](https://cloud.google.com/network-connectivity/docs/vpn/concepts/topologies#overview)에서 제안된 권장 사항을 구현합니다.

Dedicated Interconnect나 Partner Interconnect를 사용할 수 없는 경우, 고가용성 Cloud VPN을 사용하여 온프레미스를 Google 조직에 연결할 수도 있습니다.

## 사용법

1. `3-networks-svpc/envs/<environment>`의 환경 폴더에서 `vpn.tf.example`을 `vpn.tf`로 이름을 바꿉니다.
1. VPN 사전 공유 키에 대한 시크릿을 생성합니다: `echo 'MY_PSK' | gcloud secrets create VPN_PSK_SECRET_NAME --project ENV_SECRETS_PROJECT --replication-policy=automatic --data-file=-`
1. 파일에서 `environment`, `vpn_psk_secret_name`, `on_prem_router_ip_address1`, `on_prem_router_ip_address2` 및 `bgp_peer_asn`의 값을 업데이트합니다.
1. 다른 기본값이 사용자 환경에 유효한지 확인합니다.

**이 모듈은 두 개의 리전에서만 작동합니다.**


<!-- BEGINNING OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
## 입력

| 이름 | 설명 | 유형 | 기본값 | 필수 |
|------|-------------|------|---------|:--------:|
| bgp\_peer\_asn | BGP ASN for cloud routes. | `number` | n/a | yes |
| default\_region1 | Default region 1 for Cloud Routers | `string` | n/a | yes |
| default\_region2 | Default region 2 for Cloud Routers | `string` | n/a | yes |
| env\_secret\_project\_id | the environment secrets project ID | `string` | n/a | yes |
| on\_prem\_router\_ip\_address1 | On-Prem Router IP address | `string` | n/a | yes |
| on\_prem\_router\_ip\_address2 | On-Prem Router IP address | `string` | n/a | yes |
| project\_id | VPC Project ID | `string` | n/a | yes |
| region1\_router1\_name | Name of the Router 1 for Region 1 where the attachment resides. | `string` | n/a | yes |
| region1\_router1\_tunnel0\_bgp\_peer\_address | BGP session address for router 1 in region 1 tunnel 0 | `string` | n/a | yes |
| region1\_router1\_tunnel0\_bgp\_peer\_range | BGP session range for router 1 in region 1 tunnel 0 | `string` | n/a | yes |
| region1\_router1\_tunnel1\_bgp\_peer\_address | BGP session address for router 1 in region 1 tunnel 1 | `string` | n/a | yes |
| region1\_router1\_tunnel1\_bgp\_peer\_range | BGP session range for router 1 in region 1 tunnel 1 | `string` | n/a | yes |
| region1\_router2\_name | Name of the Router 2 for Region 1 where the attachment resides. | `string` | n/a | yes |
| region1\_router2\_tunnel0\_bgp\_peer\_address | BGP session address for router 2 in region 1 tunnel 0 | `string` | n/a | yes |
| region1\_router2\_tunnel0\_bgp\_peer\_range | BGP session range for router 2 in region 1 tunnel 0 | `string` | n/a | yes |
| region1\_router2\_tunnel1\_bgp\_peer\_address | BGP session address for router 2 in region 1 tunnel 1 | `string` | n/a | yes |
| region1\_router2\_tunnel1\_bgp\_peer\_range | BGP session range for router 2 in region 1 tunnel 1 | `string` | n/a | yes |
| region2\_router1\_name | Name of the Router 1 for Region 2 where the attachment resides. | `string` | n/a | yes |
| region2\_router1\_tunnel0\_bgp\_peer\_address | BGP session address for router 1 in region 2 tunnel 0 | `string` | n/a | yes |
| region2\_router1\_tunnel0\_bgp\_peer\_range | BGP session range for router 1 in region 2 tunnel 0 | `string` | n/a | yes |
| region2\_router1\_tunnel1\_bgp\_peer\_address | BGP session address for router 1 in region 2 tunnel 2 | `string` | n/a | yes |
| region2\_router1\_tunnel1\_bgp\_peer\_range | BGP session range for router 1 in region 2 tunnel 2 | `string` | n/a | yes |
| region2\_router2\_name | Name of the Router 2 for Region 2 where the attachment resides | `string` | n/a | yes |
| region2\_router2\_tunnel0\_bgp\_peer\_address | BGP session address for router 2 in region 2 tunnel 0 | `string` | n/a | yes |
| region2\_router2\_tunnel0\_bgp\_peer\_range | BGP session range for router 2 in region 2 tunnel 0 | `string` | n/a | yes |
| region2\_router2\_tunnel1\_bgp\_peer\_address | BGP session address for router 2 in region 1 tunnel 1 | `string` | n/a | yes |
| region2\_router2\_tunnel1\_bgp\_peer\_range | BGP session range for router 2 in region 1 tunnel 1 | `string` | n/a | yes |
| vpc\_name | Label to identify the VPC associated with shared VPC that will use the Interconnect. | `string` | n/a | yes |
| vpn\_psk\_secret\_name | The name of the secret to retrieve from secret manager. This will be retrieved from the environment secrets project. | `string` | n/a | yes |

## 출력

출력 없음.

<!-- END OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
