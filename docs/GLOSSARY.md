# 용어집

Terraform Example Foundation 문서에서 정의된 용어들은 대문자로 표시되며
해당 지식 영역 내에서 특정 의미를 갖습니다.

## Terraform 서비스 계정

0-bootstrap 단계의 시드 프로젝트에서 생성된 권한 있는 서비스 계정의 이메일입니다.
이 서비스 계정들은 Cloud Build와 Jenkins가 Terraform을 실행하는 데 사용됩니다. Jenkins를 사용할 때, Jenkins 에이전트의 서비스 계정은 이 Terraform 서비스 계정에 대한 가장을 사용합니다. 각 단계마다 Terraform 서비스 계정이 생성됩니다.

## 시드 프로젝트

0-bootstrap 단계에서 생성된 시드 프로젝트입니다. Terraform 서비스 계정(`terraform_service_account`)이 생성되고 후속 단계에서 각 환경의 Terraform 상태를 저장하는 데 사용되는 GCS 버킷을 호스팅하는 프로젝트입니다.

## Foundation CI/CD 파이프라인

**조직 내**의 인프라를 관리하기 위해 0-bootstrap 단계에서 생성된 프로젝트입니다.
파이프라인은 상황에 따라 **Cloud Build**, **Github Actions**, **GitLab pipeline**, **Terraform Cloud** 또는 **Jenkins** 중 하나를 사용할 수 있으며, Terraform은 시드 프로젝트 서비스 계정을 사용하여 실행됩니다.
CI/CD 프로젝트라고도 알려져 있습니다.
`bootstrap` 폴더 아래에 위치합니다.

## 앱 인프라 파이프라인

**프로젝트 내**에서 애플리케이션 인프라를 관리하도록 구성된 Cloud Build 파이프라인을 호스팅하기 위해 4-projects 단계에서 생성된 프로젝트입니다.
각 비즈니스 유닛에 대해 별도의 파이프라인이 존재하며, 4-projects에서 생성된 특정 프로젝트에 배포할 수 있는 제한된 권한을 가진 서비스 계정을 사용하도록 구성할 수 있습니다.
`common` 폴더 아래에 위치합니다.

## Terraform 원격 상태 데이터 소스

원격 [백엔드 구성](https://www.terraform.io/language/settings/backends/configuration)에서 출력 값을 검색하는 Terraform 데이터 소스입니다.
Terraform Example Foundation 문맥에서는 `0-bootstrap`과 같은 이전 단계의 출력 값을 읽어와서 사용자가 이전 단계에서 입력으로 제공된 값들을 다시 제공하거나 이전 단계에서 생성된 리소스의 값/속성을 찾을 필요가 없도록 합니다.
