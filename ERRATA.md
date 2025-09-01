# 정정사항 요약
코드 차이점과 향후 자동화에 대한 메모를 포함하여 예시 기반 리포지토리와 [Google Cloud 보안 기반 가이드](https://services.google.com/fh/files/misc/google-cloud-security-foundations-guide.pdf) 간의 차이점에 대한 개요입니다. 이 문서는 새로운 코드가 병합될 때마다 업데이트됩니다.

## 4.x [WIP]

### 코드 차이점

#### 참고사항
- "Architecture/Detective controls" 섹션에서 설명된 "로그 기반 메트릭 및 성능 메트릭에 대한 알림"은 향후 릴리스에서 통합될 예정입니다.

## 3.x [WIP]

### 코드 차이점

#### 네트워킹

- 코드에 존재하는 "allow-windows-activation" 규칙은 가이드에서 명시적으로 언급되지 않습니다.
- 프로젝트 수준의 [태그](https://cloud.google.com/resource-manager/docs/tags/tags-overview)는 향후 릴리스에서 통합될 예정입니다.
- [글로벌 네트워크 방화벽 정책](https://cloud.google.com/vpc/docs/network-firewall-policies)은 향후 릴리스에서 통합될 예정입니다.

#### 명명 규칙

- 허브 앤 스포크 네트워크 모델의 전이성 인프라에서 상태 확인을 위해 생성된 방화벽 규칙이 가이드에서 권장하는 명명 규칙을 따르지 않습니다.

## 2.x [WIP]
### 코드 차이점

#### 라벨링
- 가이드에서는 shared, service, float, nic, peer 프로젝트에 대한 vpc-type을 정의합니다. Jenkins 에이전트(vpc-b-jenkinsagents), DNS 허브(vpc-dns-hub), 4-projects에서 생성된 프로젝트에 대한 vpc-type은 정의하지 않습니다.
이는 블루프린트 가이드의 다음 버전에서 다루어질 예정입니다.

#### 명명 규칙
- 서비스 계정 명명이 블루프린트 가이드와 일치하지 않습니다. 향후 릴리스에서 명명이 적절히 수정될 예정입니다.
- 인프라 파이프라인 프로젝트 명명(`prj-buN-c-infra-pipeline`)이 블루프린트 가이드(`prj-buN-c-sample-infra-pipeline`)와 일치하지 않습니다. 향후 릴리스에서 명명이 적절히 수정될 예정입니다.

#### 네트워킹
- 코드에 존재하는 "allow-windows-activation" 규칙은 가이드에서 명시적으로 언급되지 않습니다.

#### 참고사항
- 섹션 10에서 설명된 BigQuery 로그 탐지 솔루션은 향후 릴리스에서 통합될 예정입니다.
- Splunk 로그 통합은 향후 릴리스에서 통합될 예정입니다.
- Cloud Asset Inventory는 향후 릴리스에서 통합될 예정입니다.
- 섹션 7.3에서 설명된 공유 VPC 네트워크의 할당되지 않은 IP 주소 공간은 현재 이 릴리스에서 Private Service Networking에 의해 사용되고 있습니다.

## [1.x](https://github.com/terraform-google-modules/terraform-example-foundation/releases/tag/v1.0.0)
### 코드 차이점

#### 라벨링
- 가이드에서는 shared, service, float, nic, peer 프로젝트에 대한 vpc-type을 정의합니다. Jenkins 에이전트(vpc-b-jenkinsagents), DNS 허브(vpc-dns-hub), 4-projects에서 생성된 프로젝트에 대한 vpc-type은 정의하지 않습니다.
이는 블루프린트 가이드의 다음 버전에서 다루어질 예정입니다.

#### 명명 규칙
- 서비스 계정 및 스토리지 버킷 명명이 블루프린트 가이드와 일치하지 않습니다. 향후 릴리스에서 명명이 적절히 수정될 예정입니다.

#### 배포 전 확인
- 섹션 5.2에서 설명된 Terraform Validator는 Cloud Build 및 Jenkins 파이프라인에서 구현되지 않았지만, 향후 릴리스에서 통합될 예정입니다.

#### 참고사항
- 섹션 10에서 설명된 BigQuery 로그 탐지 솔루션은 향후 릴리스에서 통합될 예정입니다.
- Splunk 로그 통합은 향후 릴리스에서 통합될 예정입니다.
- Cloud Asset Inventory는 향후 릴리스에서 통합될 예정입니다.
- 섹션 7.3에서 설명된 공유 VPC 네트워크의 할당되지 않은 IP 주소 공간은 현재 이 릴리스에서 Private Service Networking에 의해 사용되고 있습니다.
