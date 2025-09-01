# 업그레이드 가이드
v4 구성 요소 채택을 진행하기 전에 아래의 주요 변경 사항 목록을 검토하시기 바랍니다. 기능, 버그 수정 및 기타 업데이트의 전체 목록은 [변경 로그](https://github.com/terraform-google-modules/terraform-example-foundation/blob/master/CHANGELOG.md)에서 확인할 수 있습니다.

**중요:** v3에서 v4로의 현재 위치 업그레이드 경로는 제공되지 않습니다.

## 주요 변경 사항

- 1-org 단계에서 생성된 중앙 집중식 로깅에서 BigQuery 로그 대상이 제거되고, Log Analytics 지원이 활성화된 Log 버킷 대상과 연관된 BigQuery 데이터세트로 대체되었습니다.
- 0-bootstrap에서 생성되는 Terraform 상태 버킷에 대해 고객 관리 암호화 키(CMEK)가 활성화되었습니다.
- 프로젝트의 예산 알림 구성이 **지출** 값 기준 알림에서 **예측** 값 기준 알림으로 변경되었습니다.
- `compute.disableGuestAttributesAccess` 조직 정책이 제거되었습니다.
- Cloud Platform 리소스 계층 구조 변경:
  - 4-projects 단계에서 비즈니스 유닛용 하위 폴더가 생성되었습니다.
  - 네트워크 프로젝트의 상위로 사용될 새로운 Network 폴더가 생성되었습니다:
    - `prj-ENV-shared-base`
    - `prj-ENV-shared-restricted`
    - `prj-net-hub-base`
    - `prj-net-hub-restricted`
    - `prj-net-dns`
    - `prj-net-interconnect`
- 네트워크 리팩터링
  - 네트워크 프로젝트가 이제 새로운 `network` 폴더 하위에 생성됩니다.
  - VPC 방화벽 규칙(`google_compute_firewall`) 리소스가 Compute Network 방화벽 정책(`google_compute_network_firewall_policy`) 리소스로 대체되었습니다.

## 새로운 기능 통합

v3에서 v4로의 직접적인 업그레이드 경로는 없습니다. 이는 리소스가 삭제되거나 재생성될 수 있기 때문입니다.

v4의 일부 기능을 통합해야 하는 경우, 관심 있는 기능에 대한 문서를 검토하고 v4의 코드를 구현 가이드로 사용하는 것을 권장합니다. 또한 업데이트를 적용하기 전에 파괴적인 작업에 대한 `terraform plan` 출력을 검토하는 것을 권장합니다.

**참고:** `terraform`과 `gcloud`에 대해 올바른 버전을 사용하고 있는지 확인해야 합니다.
이 [검증 스크립트](https://github.com/terraform-google-modules/terraform-example-foundation/blob/master/scripts/validate-requirements.sh)를 사용하여 이러한 요구사항과 기타 추가 요구사항을 확인할 수 있습니다.
