# 기여하기

이 문서는 모듈에 기여하기 위한 가이드라인을 제공합니다.

## 종속성

개발 시스템에 다음 종속성을 설치해야 합니다:

- [Docker Engine][docker-engine]
- [Google Cloud SDK][google-cloud-sdk]
- [make]

## 입력 및 출력 문서 생성

루트 모듈, 서브모듈 및 예제 모듈의 README에 있는 입력 및 출력 테이블은
해당 모듈의 `variables`와 `outputs`를 기반으로 자동 생성됩니다.
모듈 인터페이스가 변경되면 이러한 테이블을 새로고침해야 합니다.

### 실행

새로운 입력 및 출력 테이블을 생성하려면 `make docker_generate_docs`를 실행하세요.

## 통합 테스트

통합 테스트는 이 리포지토리의 각 단계의 동작을 확인하는 데 사용됩니다.
추가, 변경 및 수정 사항에는 테스트가 함께 제공되어야 합니다.

통합 테스트는 [Blueprint test][blueprint-test] 프레임워크를 사용하여 실행됩니다. 프레임워크는 편의를 위해 Docker 이미지 내에 패키지되어 있습니다.

8개의 블루프린트 테스트가 정의되어 있으며 순차적으로 실행되어야 합니다:

- `bootstrap`
- `org`
- `envs`
- `shared`
- `networks`
- `projects-shared`
- `projects`
- `app-infra`

### 테스트 환경

리포지토리를 테스트하는 가장 쉬운 방법은 격리된 폴더에서 테스트하는 것입니다. 이러한 프로젝트의 설정은 [test/setup](./test/setup/) 디렉토리에 정의되어 있습니다.

이 설정을 사용하려면 다음 권한을 가진 서비스 계정이 필요합니다:

- 조직 내에서 조직 관리자 액세스 권한
- 폴더/조직 내에서 폴더 생성자 및 프로젝트 생성자 권한
- 청구 계정에 대한 청구 계정 관리자 권한

다음과 같이 서비스 계정 자격 증명을 환경으로 내보냅니다:

```bash
export SERVICE_ACCOUNT_JSON=$(< credentials.json)
```

또한 몇 가지 환경 변수를 설정해야 합니다:

```bash
export TF_VAR_org_id="your_org_id"
export TF_VAR_folder_id="your_folder_id"
export TF_VAR_billing_account="your_billing_account_id"
export TF_VAR_group_email="your_group_email"
export TF_VAR_domain_to_allow="your_test_domain"
export TF_VAR_example_foundations_mode="your_network_mode(base|HubAndSpoke)"
```

이러한 설정이 완료되면 Docker를 사용하여 테스트 프로젝트를 준비할 수 있습니다:

```bash
make docker_test_prepare
```

### 테스트 실행

1. `make docker_run`을 실행하여 대화형 모드에서 테스트 Docker 컨테이너를 시작합니다.

1. `cd test/integration`을 실행하여 통합 테스트 디렉토리로 이동합니다.

1. `cft test list --test-dir /workspace/test/integration`을 실행하여 사용 가능한 테스트를 나열합니다.

1. `cft test run <TEST_NAME> --stage init --verbose --test-dir /workspace/test/integration`을 실행하여 단계의 작업 디렉토리를 초기화합니다.

1. `cft test run <TEST_NAME> --stage apply --verbose --test-dir /workspace/test/integration`을 실행하여 단계를 적용합니다.

1. `cft test run <TEST_NAME> --stage verify --verbose --test-dir /workspace/test/integration`을 실행하여 현재 단계에서 생성된 리소스를 테스트합니다.

리소스 제거는 생성의 역순으로 수행되어야 합니다.

1. `cft test run <TEST_NAME> --stage destroy --verbose --test-dir /workspace/test/integration`을 실행하여 단계를 제거합니다.

## 린팅 및 포맷팅

리포지토리의 많은 파일들이 품질 표준을 유지하기 위해 린팅되거나 포맷팅될 수 있습니다.

### 실행

`make docker_test_lint`를 실행하여 린트 문제를 나열합니다.

Terraform [fmt] 명령을 사용하여 테라폼 코드를 포맷합니다.

[gofmt]를 사용하여 Go 코드를 포맷합니다.

[docker-engine]: https://www.docker.com/products/docker-engine
[flake8]: https://flake8.pycqa.org/en/latest/
[fmt]: https://www.terraform.io/cli/commands/fmt
[gofmt]: https://golang.org/cmd/gofmt/
[google-cloud-sdk]: https://cloud.google.com/sdk/install
[hadolint]: https://github.com/hadolint/hadolint
[make]: https://en.wikipedia.org/wiki/Make_(software)
[shellcheck]: https://www.shellcheck.net/
[terraform-docs]: https://github.com/segmentio/terraform-docs
[terraform]: https://terraform.io/
[blueprint-test]: https://github.com/GoogleCloudPlatform/cloud-foundation-toolkit/tree/master/infra/blueprint-test
