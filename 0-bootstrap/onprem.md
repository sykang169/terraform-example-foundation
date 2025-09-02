# Cloud Build 온프레미스 액세스

Cloud Build용으로 생성된 인프라는 [Private Pools](https://cloud.google.com/build/docs/private-pools/private-pools-overview), Google의 [서비스 프로듀서 네트워크](https://cloud.google.com/build/docs/private-pools/set-up-private-pool-to-use-in-vpc-network#setup-private-connection)와 CI/CD 프로젝트의 VPC 네트워크 간 [VPC 네트워크 피어링](https://cloud.google.com/vpc/docs/vpc-peering), 그리고 다음 세 가지 옵션 중 하나를 통한 온프레미스 연결을 설정하여 온프레미스 리소스에 대한 액세스를 허용합니다:

- [High Availability VPN](https://cloud.google.com/network-connectivity/docs/vpn/concepts/topologies#overview)
- [Dedicated Interconnect](https://cloud.google.com/network-connectivity/docs/interconnect/tutorials/dedicated-creating-9999-availability)
- [Partner Interconnect](https://cloud.google.com/network-connectivity/docs/interconnect/tutorials/partner-creating-9999-availability)

세 가지 연결 옵션 모두에서 Cloud Build 작업을 실행하는 Google 서비스 네트워크 프라이빗 풀 인스턴스가 온프레미스 네트워크의 인스턴스에 도달할 수 있도록 [사용자 지정 경로 광고 모드](https://cloud.google.com/network-connectivity/docs/router/concepts/overview#route-advertisement-custom)를 사용하여 라우터를 구성해야 합니다.

HA VPN, Dedicated Interconnect 및 Partner Interconnect 구성은 [듀얼 공유 VPC](https://cloud.google.com/architecture/security-foundations/networking#vpcsharedvpc-id7-1-shared-vpc-) 또는 [허브 앤 스포크](https://cloud.google.com/architecture/security-foundations/networking#hub-and-spoke) 두 가지 네트워크 모드 중 하나로 설정할 수 있습니다.

Cloud Build 작업이 온프레미스 인프라에 액세스할 수 있도록 피어링 설정에서 [사용자 지정 경로 가져오기 및 내보내기](https://cloud.google.com/vpc/docs/vpc-peering#importing-exporting-routes)도 구성됩니다.

0-bootstrap 단계에는 온프레미스 연결에 사용할 수 있는 선택적 고가용성 VPN 구성도 있습니다.
이 구성을 활성화하려면 다음 단계를 수행하십시오:

1. VPN 사설 사전 공유 키에 대한 Secret을 생성하고 배포에 사용되는 ID(사용자 이메일 또는 Bootstrap Terraform 서비스 계정)에 필요한 역할을 부여합니다.


   ```bash
   export project_id=<ENV_SECRETS_PROJECT>
   export secret_name=<VPN_PSK_SECRET_NAME>
   export member="serviceAccount:<BOOTSTRAP_TERRAFORM_SERVICE_ACCOUNT>|user:<YOUR_EMAIL>"

   echo '<YOUR-PRESHARED-KEY-SECRET>' | gcloud secrets create "${secret_name}" --project "${project_id}" --replication-policy=automatic --data-file=-

   gcloud secrets add-iam-policy-binding "${secret_name}"  --member="${member}" --role='roles/secretmanager.viewer' --project "${project_id}"

   gcloud secrets add-iam-policy-binding "${secret_name}"  --member="${member}"  --role='roles/secretmanager.secretAccessor' --project "${project_id}"
   ```

1. In the file `0-bootstrap/cb.tf`, in the module `tf_private_pool`, update variable `vpn_configuration.enable_vpn` to `true` and provide the required values that are valid for your environment. See the `cb-private-pool` module [README](./modules/cb-private-pool/README.md) file for additional information on the required values.
