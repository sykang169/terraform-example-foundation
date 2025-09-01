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

Eight Blueprint tests are defined and should be executed in serial order:

- `bootstrap`
- `org`
- `envs`
- `shared`
- `networks`
- `projects-shared`
- `projects`
- `app-infra`

### Test Environment

The easiest way to test the repo is in an isolated folder. The setup for such a project is defined in [test/setup](./test/setup/) directory.

To use this setup, you need a service account with:

- Organization Admin access within an organization.
- Folder Creator and Project Creator within a folder/organization.
- Billing Account Administrator on a billing account

Export the Service Account credentials to your environment like so:

```bash
export SERVICE_ACCOUNT_JSON=$(< credentials.json)
```

You will also need to set a few environment variables:

```bash
export TF_VAR_org_id="your_org_id"
export TF_VAR_folder_id="your_folder_id"
export TF_VAR_billing_account="your_billing_account_id"
export TF_VAR_group_email="your_group_email"
export TF_VAR_domain_to_allow="your_test_domain"
export TF_VAR_example_foundations_mode="your_network_mode(base|HubAndSpoke)"
```

With these settings in place, you can prepare a test project using Docker:

```bash
make docker_test_prepare
```

### Test Execution

1. Run `make docker_run` to start the testing Docker container in
   interactive mode.

1. Run `cd test/integration` to go to the integration test directory.

1. Run `cft test list --test-dir /workspace/test/integration` to list the available test.

1. Run `cft test run <TEST_NAME> --stage init --verbose --test-dir /workspace/test/integration` to initialize the working
   directory for the stage.

1. Run `cft test run <TEST_NAME> --stage apply --verbose --test-dir /workspace/test/integration` to apply the stage.

1. Run `cft test run <TEST_NAME> --stage verify --verbose --test-dir /workspace/test/integration` to test the resources created in the current stage.

Destruction of resources should be done in the reverse order of creation.

1. Run `cft test run <TEST_NAME> --stage destroy --verbose --test-dir /workspace/test/integration` to destroy the stage.

## Linting and Formatting

Many of the files in the repository can be linted or formatted to
maintain a standard of quality.

### Execution

Run `make docker_test_lint` to list lint issues.

Use Terraform command [fmt] to format terraform code.

Use [gofmt] to format Go code.

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
