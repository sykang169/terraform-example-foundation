# 업그레이드 가이드

V2 구성 요소 채택을 진행하기 전에 아래의 주요 변경 사항 목록을 검토하시기 바랍니다. 
모든 변경 사항의 전체 목록은 
[변경 로그](https://github.com/terraform-google-modules/terraform-example-foundation/blob/master/CHANGELOG.md)에서 확인할 수 있습니다.

**참고:** v1에서 v2로의 현재 위치 업그레이드 경로는 제공되지 않습니다.

## 주요 변경 사항

-  이제 리포지토리에서 최소 Terraform 버전 0.13이 필요합니다. v1의 경우 최소 버전은 
   0.12.x였습니다.
-  V2는 [Google Cloud 보안 기반 가이드](https://services.google.com/fh/files/misc/google-cloud-security-foundations-guide.pdf)의 
   섹션 7.2에 설명된 새로운 대안 허브 및 스포크 네트워크 아키텍처를 도입합니다.
-  V2에서 인프라 파이프라인은 
[Google Container Registry](https://cloud.google.com/container-registry/docs) (GCR) 사용에서 [Google Artifact Registry](https://cloud.google.com/artifact-registry/docs) 사용으로 전환되었습니다. Artifact
   Registry는 [여기](https://cloud.google.com/artifact-registry/docs/transition/transition-from-gcr#compare)에 설명된 대로 
   GCR의 기능을 확장합니다.
-  일부 [VPC 방화벽 규칙](https://cloud.google.com/vpc/docs/firewalls)이 [계층적 방화벽 정책 규칙](https://cloud.google.com/vpc/docs/firewall-policies)로 대체되었습니다. 이는 가상 머신 인스턴스에 대한 연결을 허용하거나 거부하는 동일한 기능을 제공하지만 조직 전체에서 일관된 방화벽 정책의 시행을 가능하게 합니다.

## 코드베이스 업그레이드 단계

**참고:** `terraform apply`를 실행할 때 리소스가 삭제되고 재생성될 것으로 예상됩니다. 
`terraform apply` 프로세스 중 오류를 반드시 모니터링하시기 바랍니다.

1. 개인 리포지토리에서 V1을 이미 포크한 경우, 수정된 V1 버전에 
   V2의 변경 사항을 수동으로 병합할 수 있습니다.
1. V1을 수정하지 않은 경우, 기존 포크를 
   V2로 업그레이드할 수 있습니다.
